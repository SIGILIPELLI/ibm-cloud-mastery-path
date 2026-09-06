# 08 · Monitoring & Log Analysis

Once something is running (a VSI, a Cloud Functions action, a Kubernetes
workload), you need to see whether it's healthy and, when it isn't, why.
IBM Cloud splits that into two managed services: **IBM Cloud Monitoring**
(metrics and dashboards, built on Sysdig) and **IBM Cloud Logs / Log
Analysis** (centralized log search). This module sets up both against the
resources from earlier lessons.

## Provision a Monitoring instance

```bash
ibmcloud resource service-instance-create mastery-monitoring \
  sysdig-monitor graduated-tier us-south \
  --resource-group-name mastery-path

# Get the ingestion key + Sysdig-compatible dashboard URL
ibmcloud resource service-key-create mastery-monitoring-key Manager \
  --instance-name mastery-monitoring
```

## Provision a Log Analysis instance

```bash
ibmcloud resource service-instance-create mastery-logging \
  logdna 7-day us-south \
  --resource-group-name mastery-path

ibmcloud resource service-key-create mastery-logging-key Manager \
  --instance-name mastery-logging
```

The plan name (`7-day` here) controls log retention — pick the shortest
retention that meets your needs to keep Lite-friendly usage low.

## Wiring a VSI's OS metrics/logs into these services

```bash
# Run inside the VSI from Module 3, over SSH
curl -sSL https://ibm.biz/install-sysdig-agent | sudo bash -s -- \
  -a "<monitoring-access-key>" \
  -c ingest.us-south.monitoring.cloud.ibm.com

curl -sSL https://logdna.com/install.sh | sudo bash -s <logging-ingestion-key>
```

## Pulling recent logs from the CLI

```bash
ibmcloud plugin install observe-service

ibmcloud logging tail --instance mastery-logging
ibmcloud logging query --instance mastery-logging \
  --query 'app:mastery-path AND level:error' \
  --since 1h
```

## Dashboards and alerts

Dashboards live in the Sysdig-based Monitoring UI (reached via
`ibmcloud resource service-instance mastery-monitoring --output json` for
the direct link, or the console's Observability section) and are typically
built from a small set of primitives:

- **Panels** — a chart bound to a metric (CPU%, request latency, action
  invocation count) and an aggregation (avg, max, rate).
  A capstone-relevant example: a panel tracking Cloud Functions
  `invocations` and `errors` for the `visits-api` package from Module 7.
- **Alerts** — a threshold or anomaly rule on a metric that notifies a
  channel (email, Slack via webhook, PagerDuty) when crossed.

```bash
# Alerts can also be defined as code and applied via the API/Terraform
# provider covered in Module 9 -- for now, a conceptual example:
cat <<'EOF' > high-error-rate-alert.json
{
  "name": "High Cloud Functions error rate",
  "description": "Fires if action error rate exceeds 5% over 5 minutes",
  "type": "METRIC",
  "condition": "avg(functions.errors_rate) > 0.05",
  "timespan": 300,
  "severity": "high",
  "notificationChannels": ["email:you@example.com"]
}
EOF
```

## How It Actually Works

- **IBM Cloud Monitoring is built on Sysdig, and metrics arrive via an
  agent that scrapes the local kernel/cgroup and process tables on each
  host or container, batching data points and shipping them to a
  time-series backend rather than the backend polling each instance
  directly.** That's why installing the monitoring agent (or, on
  managed services, IBM's own instrumentation) is a prerequisite for
  data to exist at all — without an agent pushing data, there is
  nothing for the dashboard to query.
- **A dashboard panel is a saved query against that time-series store,
  re-executed live every time you view it** — nothing is pre-rendered
  and cached; the numbers you see are the query's result as of that
  instant, aggregated (sum/avg/percentile) over the time window and
  grouping you configured, which is why zooming a time range or
  changing the aggregation recomputes the panel from raw samples rather
  than redrawing a static image.
- **Log Analysis is built on ELK-family technology (Elasticsearch under
  the hood) — every log line you ship is tokenized and indexed into an
  inverted index at write time**, which is the actual mechanism that
  makes full-text search across millions of log lines return in
  milliseconds: you're not scanning raw log files at query time, you're
  looking up pre-built token-to-document mappings.
- **An alert isn't a background thread watching your infrastructure
  continuously — it's a query re-evaluated on a fixed polling interval
  (e.g. every minute) against the same time-series backend the
  dashboards use**, and the alert fires the moment that query's result
  crosses your configured threshold for the configured duration; the
  duration requirement exists specifically to avoid firing on a single
  noisy sample rather than a sustained condition.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> sysdig-monitor <plan> <region>` | Provision Cloud Monitoring |
| `ibmcloud resource service-instance-create <name> logdna <plan> <region>` | Provision Log Analysis |
| `ibmcloud resource service-key-create <key> Manager --instance-name <svc>` | Get an ingestion/dashboard key |
| `ibmcloud logging tail --instance <name>` | Stream live logs |
| `ibmcloud logging query --instance <name> --query "<q>" --since <window>` | Search historical logs |
| Sysdig agent install script | Ship OS-level metrics from a VSI |
| LogDNA agent install script | Ship OS-level logs from a VSI |

## Exercise

Provision a Monitoring instance and a Log Analysis instance (`7-day` plan),
then run `ibmcloud fn action invoke hello --param name "test"` several
times from Module 7 and use `ibmcloud logging query` to find the resulting
activation log lines. Sketch (in words, no need to actually build it) what
a "high error rate" alert on the `hello` action's invocation errors would
need: which metric, what threshold, and who it should notify.
