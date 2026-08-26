# 05 · Observability at Scale (Activity Tracker, Sysdig)

A handful of resources can be watched by hand in the console. A ROKS
cluster with a dozen microservices and an Event Streams pipeline behind it
cannot. This module wires up **IBM Cloud Monitoring (Sysdig-based)** for
metrics and alerting, **Log Analysis** for centralized logs, and revisits
**Activity Tracker** from an operations (not just compliance) angle.

## Provision the monitoring instance

```bash
ibmcloud resource service-instance-create monitoring-mastery \
  sysdig-monitor graduated-tier us-south --resource-group-name mastery-path
```

```text
Service instance monitoring-mastery is being created.
OK
```

## Wire a ROKS cluster to send metrics

```bash
ibmcloud ob monitoring config create \
  --instance monitoring-mastery \
  --cluster roks-mastery
```

```text
Configuring cluster 'roks-mastery' to monitoring instance 'monitoring-mastery'...
sysdig-agent daemonset deployed to namespace ibm-observe
OK
```

Confirm the agent pods are actually running before assuming metrics are
flowing — a common early mistake is checking the Sysdig dashboard for data
that never arrives because the daemonset failed to schedule (often an SCC
issue, following on from Module 1: the agent needs a `privileged` SCC
grant to read host-level metrics):

```bash
oc get pods -n ibm-observe
```

```text
NAME                     READY   STATUS    RESTARTS
sysdig-agent-4x2kq       1/1     Running   0
sysdig-agent-9dvbn       1/1     Running   0
sysdig-agent-p7z1w       1/1     Running   0
```

## Log Analysis: centralize container and platform logs

```bash
ibmcloud resource service-instance-create logs-mastery \
  logdna 7-day us-south --resource-group-name mastery-path

ibmcloud ob logging config create \
  --instance logs-mastery \
  --cluster roks-mastery
```

```text
OK
Logging agent deployed to namespace ibm-observe
```

Query recent logs from the CLI without opening the console, useful for
scripted health checks:

```bash
ibmcloud logging tail --instance logs-mastery --filter "app:frontend AND level:error"
```

## Alerting on a metric, not just dashboards

A dashboard nobody watches at 2 a.m. doesn't page anyone. Define an alert
policy instead:

```bash
ibmcloud ob monitoring alert-create \
  --instance monitoring-mastery \
  --name high-error-rate \
  --description "Frontend 5xx rate over 5% for 5 minutes" \
  --severity high \
  --condition 'avg(sysdig_http_error_rate{kube_deployment_name="frontend"}) > 0.05' \
  --duration 300 \
  --notification-channel pagerduty:orders-oncall
```

```text
Alert policy high-error-rate created.
```

**Gotcha**: alert conditions are metric-name-sensitive to the exact
Sysdig integration in use (kube-state-metrics label names change between
agent versions) — always confirm the metric name exists first with
`ibmcloud ob monitoring metrics --instance monitoring-mastery | grep http_error`
before wiring a condition that will silently never fire.

## Distributed tracing across services

Metrics show *that* the frontend is slow; tracing shows *which*
downstream call caused it. IBM Cloud doesn't ship a dedicated managed
tracing product distinct from Sysdig's APM features — the common pattern
is OpenTelemetry instrumentation in-app, exporting to the Sysdig-compatible
OTLP endpoint:

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'https://ingest.us-south.monitoring.cloud.ibm.com/otlp/v1/traces',
    headers: { 'IBM-Instance-ID': process.env.SYSDIG_INSTANCE_GUID,
               'Authorization': `Bearer ${process.env.SYSDIG_ACCESS_KEY}` },
  }),
});
sdk.start();
```

Every service in the request path needs this — a trace with a gap because
one service wasn't instrumented is nearly as useless as no trace at all.

## Correlate: Activity Tracker for the "who changed the config" question

When a dashboard shows a sudden behavior change (not an error spike, a
behavior change — e.g. response times permanently doubled), the first
question is often "did someone change something," which is Activity
Tracker's job, not the monitoring instance's:

```bash
ibmcloud atracker events --target <cos-audit-target-id> \
  --start "2026-08-25T00:00:00Z" --end "2026-08-26T00:00:00Z" \
  | jq '.[] | select(.action | contains("worker-pool"))'
```

## Terraform for the monitoring config

```hcl
resource "ibm_resource_instance" "monitoring" {
  name              = "monitoring-mastery"
  service           = "sysdig-monitor"
  plan              = "graduated-tier"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}

resource "ibm_ob_monitoring" "roks_monitoring" {
  cluster     = ibm_container_vpc_cluster.roks.id
  instance_id = ibm_resource_instance.monitoring.guid
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Sysdig agent needs privileged SCC** on ROKS — expect the Module 1
  lesson about SCC denials to resurface the first time you enable
  monitoring on a cluster.
- **7-day log retention (`7-day` plan) is genuinely 7 days** — export or
  archive anything needed for a longer audit window to Cloud Object
  Storage before it ages out.
- **Alert notification channels must be configured separately** (Slack,
  PagerDuty, webhook) before an alert policy referencing them will
  actually notify anyone — a policy referencing a channel that doesn't
  exist creates silently and never fires.
- **Cost scales with ingested volume**, both for logs and for high-
  cardinality custom metrics (e.g. one metric series per unique user ID)
  — cardinality explosions are the most common surprise monitoring bill.

## Cheat sheet

| Task | Command |
|---|---|
| Create monitoring instance | `ibmcloud resource service-instance-create <n> sysdig-monitor graduated-tier <region>` |
| Attach cluster to monitoring | `ibmcloud ob monitoring config create --instance <n> --cluster <c>` |
| Attach cluster to logging | `ibmcloud ob logging config create --instance <n> --cluster <c>` |
| Tail logs with filter | `ibmcloud logging tail --instance <n> --filter "<query>"` |
| Create alert | `ibmcloud ob monitoring alert-create --instance <n> --condition '<expr>'` |
| List available metrics | `ibmcloud ob monitoring metrics --instance <n>` |

## Exercise

1. Attach an existing ROKS cluster to a new monitoring instance and a new
   logging instance, and confirm the agent pods reach `Running`.
2. Write an alert policy on a real metric name you confirmed exists via
   `ibmcloud ob monitoring metrics`.
3. Instrument a small Node.js service with OpenTelemetry exporting to the
   Sysdig OTLP endpoint and generate one trace.
4. Use `ibmcloud atracker events` to find and explain one configuration
   change in your account's recent history.
