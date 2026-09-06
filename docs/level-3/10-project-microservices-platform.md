# 10 · Project — Multi-Tier Microservices Platform

This project combines every Level 3 module into one working platform: an
order-processing system running on OpenShift, connected across a
multi-region network, secured and observed, governed by cost policy, and
fronted by a managed API gateway.

## Architecture

```text
                         ┌────────────────────────┐
 partner apps ─────────▶ │  API Connect gateway    │
                         └───────────┬─────────────┘
                                     │ Route (TLS)
                         ┌───────────▼─────────────┐
                         │  ROKS: orders-frontend    │
                         └───────────┬─────────────┘
                                     │ publishes
                         ┌───────────▼─────────────┐
                         │  Event Streams:           │
                         │  orders.created topic     │
                         └──┬───────────┬───────────┘
                consumes    │           │  consumes
        ┌───────────────────▼┐       ┌──▼──────────────────┐
        │ ROKS: inventory-svc │       │ ROKS: billing-svc    │
        └──────────┬──────────┘      └──────────┬───────────┘
                   │                             │
        ┌──────────▼──────────┐      ┌───────────▼──────────┐
        │ Databases for        │      │ Databases for         │
        │ PostgreSQL (orders)  │      │ PostgreSQL (billing)  │
        │  + us-east replica   │      │  + us-east replica    │
        └───────────────────────┘      └────────────────────────┘

  All VPC traffic routed through Transit Gateway; Direct Link to on-prem
  warehouse system for inventory sync. Sysdig monitoring + Log Analysis
  attached to the ROKS cluster. Key Protect root keys wrap the databases
  and COS buckets. SCC's CIS profile evaluates the whole account.
```

## Step 1 — Network foundation

```bash
ibmcloud is vpc-create platform-vpc --address-prefix-management manual
ibmcloud is vpc-address-prefix-create pfx-z1 platform-vpc us-south-1 10.30.0.0/20
ibmcloud is subnet-create private-z1 platform-vpc --zone us-south-1 --ipv4-address-count 128

ibmcloud tg gateway-create --name platform-tgw --location us-south
ibmcloud tg connection-add platform-tgw --network-type vpc \
  --network-id $(ibmcloud is vpc platform-vpc --output json | jq -r .crn)
```

## Step 2 — OpenShift cluster and namespaces per service

```bash
ibmcloud ks cluster create vpc-gen2 --name platform-roks \
  --vpc-id $(ibmcloud is vpc platform-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-z1 --output json | jq -r .id) \
  --zone us-south-1 --flavor bx2.4x16 --workers 3 --version 4.14_openshift

oc new-project orders-frontend
oc new-project inventory-svc
oc new-project billing-svc
```

## Step 3 — Event Streams backbone

```bash
ibmcloud resource service-instance-create platform-events messagehub standard us-south
ibmcloud es topic-create orders.created --instance platform-events --partitions 3 --replication-factor 3
```

`orders-frontend` publishes to `orders.created`; `inventory-svc` and
`billing-svc` each consume it in their own consumer group, per Module 04
— neither downstream service blocks the other, and neither is a direct
dependency of the frontend.

## Step 4 — Databases with cross-region replicas

```bash
ibmcloud cdb deployment-create orders-db --datacenter us-south --plan standard --version 15
ibmcloud cdb deployment-create orders-db-dr --datacenter us-east \
  --replica-of $(ibmcloud cdb deployment orders-db --output json | jq -r .id)

ibmcloud cdb deployment-create billing-db --datacenter us-south --plan standard --version 15
ibmcloud cdb deployment-create billing-db-dr --datacenter us-east \
  --replica-of $(ibmcloud cdb deployment billing-db --output json | jq -r .id)
```

## Step 5 — Security: Key Protect + SCC

```bash
ibmcloud resource service-instance-create platform-kp kms tiered-pricing us-south
ibmcloud kp key create root-key-platform --instance-id <kp-guid> --standard-key false

ibmcloud scc profile-attach --profile "CIS IBM Cloud Foundations Benchmark" \
  --scope-id $(ibmcloud account show --output json | jq -r .account_id) \
  --instance-id <scc-guid>
```

Reference `root-key-platform`'s CRN when creating both databases'
encryption configuration and the COS bucket used for order-archive
exports, so one key rotation event covers every regulated data store in
the platform.

## Step 6 — Observability

```bash
ibmcloud resource service-instance-create platform-monitoring sysdig-monitor graduated-tier us-south
ibmcloud ob monitoring config create --instance platform-monitoring --cluster platform-roks

ibmcloud ob monitoring alert-create --instance platform-monitoring \
  --name orders-frontend-error-rate \
  --condition 'avg(sysdig_http_error_rate{kube_namespace_name="orders-frontend"}) > 0.05' \
  --duration 300 --severity high
```

## Step 7 — Governance and cost tags

```bash
ibmcloud resource tag-attach \
  --resource-id $(ibmcloud is vpc platform-vpc --output json | jq -r .crn) \
  --tag-names env:prod,team:platform,cost-center:eng-42

ibmcloud billing budget-create --name platform-monthly-budget \
  --amount 3000 --resource-group-name mastery-path --alert-percentages 50,80,100
```

## Step 8 — Front it with API Connect

```bash
ibmcloud resource service-instance-create platform-apic api-connect professional us-south
ibmcloud apic draft-apis:create --server platform-apic orders-api.yaml
ibmcloud apic products:create --server platform-apic --title "Orders API" \
  --apis orders-api:1.0.0 --plan default --rate-limit "1000/1hour"
ibmcloud apic products:publish --server platform-apic orders-api-product:1.0.0 \
  --catalog production-catalog --space default
```

## Verification checklist

- [ ] `oc get pods -A` shows `orders-frontend`, `inventory-svc`, and
      `billing-svc` all `Running`, each with its Route reachable.
- [ ] A test order posted through the API Connect gateway produces an
      `orders.created` event that both consumers process independently
      (confirm via consumer group offsets, not just app logs).
- [ ] `ibmcloud cdb deployment-connections orders-db-dr` shows an active
      replica relationship to `orders-db`.
- [ ] `ibmcloud scc results` shows at least one completed scan cycle
      against the account.
- [ ] The Sysdig dashboard shows live metrics for all three namespaces,
      and the error-rate alert policy exists (`ibmcloud ob monitoring
      alert-list`).
- [ ] `ibmcloud billing resource-group-usage` shows tagged spend for
      `team:platform`.

## Terraform composition (top level)

```hcl
module "network" {
  source = "./modules/ha-vpc"
  region = "us-south"
}

module "cluster" {
  source     = "./modules/roks-cluster"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}

module "events" {
  source = "./modules/event-streams"
  topics = ["orders.created"]
}

module "databases" {
  source        = "./modules/postgres-with-dr"
  primary_region = "us-south"
  dr_region      = "us-east"
  names          = ["orders-db", "billing-db"]
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## How It Actually Works

- **The decoupling this platform relies on holds because `inventory-svc`
  and `billing-svc` are two independent consumer groups reading the same
  Kafka partition log, not two subscribers on a fan-out bus.** Each group
  maintains its own committed offset into `orders.created`; a slow or
  crashed `billing-svc` doesn't block `inventory-svc`'s consumption
  because Kafka retains the log and serves each group from wherever its
  own offset sits — adding `notifications-svc` in the stretch goal is
  purely "start a third consumer group at whatever offset it chooses,"
  which is why it requires zero change to the producer side.
- **One root key covering both databases and the COS bucket means one
  Key Protect rotation event re-wraps every data-encryption key those
  services hold a reference to — it doesn't touch or re-encrypt the
  underlying data itself**, per the envelope-encryption mechanism from
  Module 03. That's the operational payoff of standardizing on a single
  platform-wide root key: a security incident response that needs to
  revoke and rotate a key does it once, for everything the platform
  encrypts, instead of hunting down every service's own key.
- **The Sysdig alert and the API Connect rate limit protect two different
  layers of the same request path and can both be true simultaneously.**
  A caller hammering `/orders` first hits API Connect's per-plan counter
  (Module 09) — if under the limit, the request proceeds to the ROKS
  Route and only then shows up in Sysdig's per-namespace error-rate metric
  (Module 05) if the backend itself starts failing. A spike in gateway
  429s and a spike in backend 5xx alerts are diagnosing different
  problems even though both fire from the same burst of traffic.
- **Cross-region DR readiness here is only as real as the last drill —
  the replica relationship shown by `deployment-connections` proves
  replication is live, not that promotion and failback actually work
  end-to-end**, which is exactly why Module 06 treats a timed promotion
  drill as the real verification, and why this project's checklist checks
  for an active replica but the stretch goal insists on actually running
  the failover.

## Stretch goals

- Add a fourth consumer service (`notifications-svc`) to `orders.created`
  without modifying `orders-frontend`, proving the decoupling actually
  holds.
- Wire SCC findings into Event Notifications so a failed control posts to
  a chat channel automatically.
- Run a full failover drill: promote `orders-db-dr`, repoint `orders-svc`
  config, and measure the real RTO end-to-end.
- Add a second API Connect product version (`v2`) with a breaking change
  to the `/orders` schema, and keep `v1` callers unaffected.
- Turn the manual verification checklist into a script that checks each
  item via CLI and exits non-zero on any failure.
