# 10 · Capstone Project

The capstone combines every Level 4 module — enterprise account
structure, landing zone, hybrid connectivity, data/AI services,
performance engineering, deep security, GitOps pipelines, FinOps, and
SRE — into one exercise: stand up a compliant, observable,
enterprise-governed platform for a fictional multi-team organization, on
top of the Level 3 microservices platform.

## Scenario

"Acme Logistics" runs the Level 3 order-processing platform in
production and needs to onboard a second team (`catalog`) under proper
enterprise governance, add AI-assisted customer support, connect a
regional warehouse's on-prem system, and prove the whole platform meets a
reliability target with an auditable cost story.

## Step 1 — Enterprise structure for two teams

```bash
ibmcloud enterprise account-group-create --name product-teams \
  --parent $(ibmcloud enterprise list --output json | jq -r '.resources[0].crn')

ibmcloud enterprise account-create --name orders-prod \
  --parent $(ibmcloud enterprise account-group product-teams --output json | jq -r .crn) \
  --owner-email orders-lead@acme.example

ibmcloud enterprise account-create --name catalog-prod \
  --parent $(ibmcloud enterprise account-group product-teams --output json | jq -r .crn) \
  --owner-email catalog-lead@acme.example
```

## Step 2 — Landing zone per account

```hcl
module "orders_landing_zone" {
  source  = "terraform-ibm-modules/landing-zone/ibm"
  version = "~> 5.0"
  prefix  = "orders"
  region  = "us-south"
  vpcs    = ["management", "workload"]
}

module "catalog_landing_zone" {
  source  = "terraform-ibm-modules/landing-zone/ibm"
  version = "~> 5.0"
  prefix  = "catalog"
  region  = "us-south"
  vpcs    = ["management", "workload"]
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

Both accounts get identical guardrails (MFA enforcement, SCC profile
attachment, mandatory cost tags) automatically because both are built
from the same landing zone module — the governance win Module 02 argued
for, now demonstrated with a second real tenant.

## Step 3 — Connect a warehouse via Satellite

```bash
ibmcloud sat location create --name warehouse-dc \
  --managed-from wdc --zone wh-a --zone wh-b --zone wh-c

ibmcloud sat cluster create --name warehouse-roks \
  --location warehouse-dc --kube-version 4.14_openshift \
  --zone wh-a --zone wh-b --zone wh-c

ibmcloud tg connection-add hub-tgw --network-type gre_tunnel \
  --network-id $(ibmcloud sat location warehouse-dc --output json | jq -r .crn)
```

The warehouse's inventory-sync service runs on Satellite-managed
OpenShift, physically on-prem, but routes through the same Transit
Gateway hub as both cloud accounts.

## Step 4 — AI-assisted support on the catalog service

```python
model = ModelInference(model_id="ibm/granite-13b-instruct-v2", api_client=client, params={...})

def answer_product_question(question, catalog_chunks):
    prompt = f"Using only this catalog data:\n{catalog_chunks}\n\nQuestion: {question}\nAnswer:"
    return model.generate_text(prompt=prompt)
```

Wired to watsonx.governance to track response drift over time, per
Module 04.

## Step 5 — Performance targets and SLOs

```text
orders-frontend SLO:  99.5% of requests < 500ms, 5xx excluded, 28-day window
catalog-frontend SLO: 99.0% of requests < 800ms (looser: less critical path)
```

```bash
oc autoscale deployment/orders-frontend --min 3 --max 12 --cpu-percent 65 -n orders-frontend
ibmcloud ks worker-pool autoscale set --cluster orders-roks --worker-pool default --enable --min-size 3 --max-size 10
```

Load test both services (Module 05's k6 pattern) and derive the max
replica counts from measured per-replica throughput against each SLO
target, not from guesswork.

## Step 6 — Deep security for payment data

```bash
ibmcloud resource service-instance-create hpcs-acme hs-crypto standard us-south \
  --parameters '{"units": 2}'
```

Payment-adjacent fields in the `billing-svc` database use HPCS-wrapped
KYOK keys (Module 06); everything else continues on Key Protect BYOK from
Level 3 — a deliberate, documented decision about which data actually
needs KYOK's cost and operational overhead.

## Step 7 — GitOps pipeline for both teams

```text
infra-repo/
  environments/
    orders-prod/
    catalog-prod/
manifests-repo/
  apps/
    orders-frontend/overlays/{dev,staging,prod}/
    catalog-frontend/overlays/{dev,staging,prod}/
```

One shared CI/CD pattern (Module 07), parameterized per team's
environment folder — a `catalog` engineer's PR triggers plan/apply
against `catalog-prod`'s Schematics workspace only, never `orders-prod`'s.

## Step 8 — FinOps rollup across both accounts

```bash
ibmcloud enterprise account-group-usage-report \
  --account-group-id $(ibmcloud enterprise account-group product-teams --output json | jq -r .id) \
  --billing-month 2026-08
```

```text
Account          Charges (USD)
orders-prod       5,204.11
catalog-prod       1,876.44
```

## Verification checklist

- [ ] Two accounts exist under one enterprise account group, each with
      its own landing zone applied (`terraform state list` shows
      resources scoped correctly per account).
- [ ] The Satellite-hosted warehouse cluster is attached to the shared
      Transit Gateway and reachable from `orders-prod`'s workload VPC.
- [ ] A watsonx-backed catalog question-answering endpoint returns an
      answer grounded only in provided catalog data.
- [ ] Both frontends have HPAs and worker-pool autoscaling sized from an
      actual load test, not a guessed number.
- [ ] `billing-svc`'s sensitive fields reference an HPCS-wrapped key;
      everything else references Key Protect.
- [ ] A PR against `catalog-prod`'s environment folder triggers only
      `catalog-prod`'s Schematics plan, verified in CI logs.
- [ ] An enterprise account-group usage report shows both accounts'
      spend in one view.
- [ ] An SLO and a fast/slow burn-rate alert pair exist for at least one
      service, with the arithmetic behind the thresholds documented.

## How It Actually Works

- **Both accounts inherit identical guardrails from one landing zone
  module because Terraform composes the same underlying resource graph
  twice, once per `module` block, against two different account-scoped
  provider configurations** — the module's HCL is fetched and evaluated
  once but instantiated per module call, so `orders_landing_zone` and
  `catalog_landing_zone` end up as two independent sets of the same VPC,
  IAM, and SCC resources rather than one shared instance split between
  teams. That independence is exactly what makes a GitOps PR against
  `catalog-prod` provably unable to touch `orders-prod`: they don't share
  a Schematics workspace, state file, or Terraform module instance at all.
- **The account-group usage rollup, the Satellite-to-Transit-Gateway
  routing, and the per-team GitOps isolation are all instances of the
  same underlying pattern from earlier modules applied at once: a shared
  control-plane resource (billing hierarchy, routing fabric, or CI
  pipeline definition) parameterized per tenant rather than duplicated
  by hand** — which is the actual argument for the whole capstone's
  architecture: none of Level 4's individual mechanisms changed to
  support two teams, they were simply invoked twice with different
  parameters, exactly as Module 02's landing zone argued a standardized
  starting point should behave.
- **Segregating KYOK to only payment-adjacent fields works because
  envelope encryption operates per data-encryption-key, not per
  database** — a table's columns can reference different wrapping keys
  for different rows or fields as long as the application (or a
  column-level encryption feature) chooses which key CRN wraps which
  DEK, so `billing-svc` genuinely can have some fields protected by an
  HPCS master key with all its ceremony overhead while adjacent
  non-sensitive fields use ordinary Key Protect BYOK, without needing two
  separate databases.
- **A cost-based autoscaling ceiling and an SLO's error budget are
  reconciled by the same arithmetic relationship Module 05 introduced for
  sizing, just run in the opposite direction** — Module 05 derived
  `max-size` from a required replica count to hit a latency target;
  capping `max-size` at whatever the FinOps budget affords instead fixes
  the ceiling first and makes the achievable SLO the dependent variable,
  which is precisely why documenting that tradeoff explicitly (rather
  than treating both as independent, always-satisfiable constraints)
  matters once real traffic exceeds what the budget-capped ceiling can
  serve.

## Stretch goals

- Run a real (simulated) incident: intentionally break `catalog-frontend`,
  page via the fast-burn alert, mitigate with a `git revert`, and write a
  blameless postmortem with owned, dated action items.
- Add a third account (`analytics-prod`) purely from the landing zone
  module and measure how long it takes from account creation to a
  fully-governed, SCC-scanned, cost-tagged, monitored environment —
  that duration is the real payoff of everything built across Level 4.
- Extend the FinOps anomaly script to post directly to a chat channel via
  a scheduled Code Engine job, closing the loop from Module 08's manual
  script to an actual recurring practice.
- Implement a cost-based autoscaling ceiling: cap the HPA/worker-pool max
  at whatever replica count the FinOps budget (Module 08) can sustain,
  and document the tradeoff between that ceiling and the SLO's error
  budget when the two conflict under real load.
