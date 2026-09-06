# 07 · Cost Management & Governance

Everything built in Levels 1–3 costs money the moment it's provisioned.
This module covers seeing that spend clearly (billing/usage APIs, cost
tags), controlling it before it happens (budgets, quotas), and enforcing
governance (Enterprise account structure, resource group policies).

## Understand the billing hierarchy

```text
Enterprise
 └── Account Group (e.g. "Production")
      └── Account (e.g. "orders-platform-prod")
           └── Resource Group (e.g. "mastery-path")
                └── Resources (VPC, ROKS cluster, COS bucket, ...)
```

Costs roll up: an Enterprise views aggregate spend across every child
account; a single account only sees its own. Module 08 in Level 4 goes
deeper into enterprise account design — this module works at the
account/resource-group level.

## View current usage from the CLI

```bash
ibmcloud billing account-usage --output json | jq '.resources[] | {name: .resource_name, charge: .charges}'
```

```json
{"name": "Red Hat OpenShift on IBM Cloud", "charge": 214.37}
{"name": "Cloud Object Storage", "charge": 6.82}
{"name": "Databases for PostgreSQL", "charge": 89.10}
```

```bash
ibmcloud billing resource-group-usage --resource-group-name mastery-path
```

Resource-group-level usage is what makes chargeback to a specific team
possible — put every workload's resources in a dedicated resource group
from day one rather than reorganizing later, since resources can't be
retroactively re-tagged into usage history that already happened.

## Tag resources for cost attribution

```bash
ibmcloud resource tag-attach \
  --resource-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .crn) \
  --tag-names env:prod,team:orders,cost-center:eng-42
```

```text
OK
Tag(s) 'env:prod,team:orders,cost-center:eng-42' was attached.
```

Query spend by tag later:

```bash
ibmcloud billing account-usage --output json | \
  jq '.resources[] | select(.tags[]? == "team:orders")'
```

**Gotcha**: tags attached *after* a billing period has closed don't
retroactively attribute that period's spend — tag at creation time,
ideally by having Terraform apply tags as part of the resource definition:

```hcl
resource "ibm_is_vpc" "ha_app_vpc" {
  name = "ha-app-vpc"
  tags = ["env:prod", "team:orders", "cost-center:eng-42"]
}
```

## Set a budget and get alerted before overrun

```bash
ibmcloud billing budget-create \
  --name orders-monthly-budget \
  --amount 1500 \
  --resource-group-name mastery-path \
  --alert-percentages 50,80,100
```

```text
OK
Budget orders-monthly-budget created. Alerts at 50%, 80%, 100% of $1500/mo.
```

Budgets are visibility, not enforcement — hitting 100% sends a
notification, it does not stop new resources from being created. For hard
limits, pair budgets with quotas (below) or a governance policy that
requires approval above a threshold.

## Quotas: hard caps on resource creation

```bash
ibmcloud is quota-list
```

```text
Resource                    Limit   In Use
vpcs                        10      3
instances-vcpu               200    64
floating-ips                  20      4
```

Default VPC infra quotas are generous per-account safety limits (not
budget controls) — they exist to prevent runaway automation (e.g., a
Terraform loop bug creating hundreds of floating IPs) from silently
draining the account, and support can raise them on request for a
legitimate need.

## Enterprise-level spending controls

```bash
ibmcloud enterprise account-group-create \
  --name production \
  --parent $(ibmcloud enterprise list --output json | jq -r '.resources[0].crn') \
  --primary-contact-iam-id <iam-id>
```

```text
Account group 'production' created.
```

```bash
ibmcloud enterprise account-group-usage-report --account-group-id <ag-id> \
  --billing-month 2026-08
```

Grouping accounts (e.g., `orders-platform-prod`, `orders-platform-dev`)
under one Account Group in the Enterprise gives finance one line item
covering both, while resource groups inside each account still separate
spend by team.

## Autoscaling to control cost, not just performance

Over-provisioned capacity is the most common avoidable cost. ROKS worker
pools support autoscaling:

```bash
ibmcloud ks worker-pool autoscale set \
  --cluster roks-mastery \
  --worker-pool default \
  --enable \
  --min-size 2 \
  --max-size 6 \
  --cooldown 300
```

```text
OK
Autoscaling enabled for worker pool 'default'.
```

## Terraform for the budget

```hcl
resource "ibm_resource_group" "mastery_path" {
  name = "mastery-path"
}

resource "ibm_billing_budget" "orders_budget" {
  name              = "orders-monthly-budget"
  amount            = 1500
  resource_group_id = ibm_resource_group.mastery_path.id
  alert_percentages = [50, 80, 100]
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Budgets don't block spend** — treat a "budget exceeded" alert as a
  trigger for investigation, not proof that spending stopped.
- **Reserved/committed-use discounts require a separate agreement** with
  IBM — the pay-as-you-go rates shown by `ibmcloud billing` are list
  prices, not what a large account with a signed agreement actually pays.
- **Deleted resources still show in the billing period they ran in** —
  deleting a cluster mid-month doesn't erase the partial-month charge
  already accrued.
- **Tags used for cost attribution are user tags, not access-management
  tags** — the two tag systems look similar in the console but serve
  different purposes (attribution vs. IAM scoping) and are managed by
  different `ibmcloud resource tag-attach --tag-type` values.

## How It Actually Works

- **Usage attribution is computed from metering records emitted by each
  service, not from polling resource inventories.** Every billable
  service reports discrete usage events (an hour of ROKS worker time, a
  GB-month of COS storage) tagged with the resource's ID, resource group,
  and whatever access-management/cost tags were attached to it *at the
  moment the event was recorded*. That's the actual mechanism behind
  "tagging after the fact doesn't retroactively attribute spend" — the
  historical metering records were already written without the tag and
  are never rewritten once billing has rolled them up.
- **A budget is a threshold watcher over the same aggregated usage data
  the billing API exposes, evaluated on a periodic sweep — not a live
  gate any provisioning call passes through.** Creating a resource never
  consults the budget at all; the budget engine separately totals a
  resource group's usage-to-date against the configured amount and fires
  a notification when a percentage threshold is crossed, which is exactly
  why it can only ever alert after the spend already happened rather than
  prevent it.
- **VPC infrastructure quotas are enforced synchronously at the API layer,
  which is the real distinction from budgets.** A `POST` to create a
  floating IP or instance is checked against the account's quota
  allowance in the same request path that validates the rest of the
  payload, and a request that would exceed the limit is rejected outright
  — no resource is created, no charge accrues, and no alert is needed
  because the action itself never completed. That's why quotas stop a
  runaway loop and budgets only report on one after the fact.
- **Autoscaling on a worker pool watches pending-pod pressure via the
  cluster autoscaler component, not raw CPU usage on existing nodes.** It
  polls the Kubernetes scheduler for pods that are `Unschedulable` due to
  insufficient resources; if any exist and the pool is under `max-size`,
  it requests a new worker node from the ROKS control plane. Scale-down
  works the opposite way — a node whose pods could all be rescheduled
  elsewhere is cordoned, drained, and removed once utilization stays low
  past the `--cooldown` window, which is why cooldown exists at all: to
  stop nodes being added and removed on every small, momentary fluctuation.

## Cheat sheet

| Task | Command |
|---|---|
| View account usage | `ibmcloud billing account-usage` |
| View resource-group usage | `ibmcloud billing resource-group-usage --resource-group-name <rg>` |
| Attach a cost tag | `ibmcloud resource tag-attach --resource-id <crn> --tag-names <k:v,...>` |
| Create a budget | `ibmcloud billing budget-create --name <n> --amount <usd> --resource-group-name <rg>` |
| List VPC quotas | `ibmcloud is quota-list` |
| Enable worker pool autoscaling | `ibmcloud ks worker-pool autoscale set --cluster <c> --worker-pool <p> --enable` |

## Exercise

1. Tag at least three resources with `env`, `team`, and `cost-center`
   keys, then query billing usage filtered to one tag value.
2. Create a monthly budget on a resource group with alerts at 50/80/100%.
3. Enable autoscaling on a ROKS worker pool with a min/max range, and
   explain in prose what triggers a scale-up versus a scale-down.
4. Write the resource group and budget as Terraform and run
   `terraform validate`.
