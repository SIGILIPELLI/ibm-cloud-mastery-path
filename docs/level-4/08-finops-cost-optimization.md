# 08 · FinOps & Enterprise Cost Optimization

Level 3's cost module covered budgets and tagging for one account.
**FinOps** at enterprise scale is the ongoing practice built on top of
that visibility: reserved capacity decisions, anomaly detection,
showback/chargeback across teams, and rightsizing as a recurring process
rather than a one-time task.

## Enterprise-wide cost visibility

```bash
ibmcloud enterprise account-group-usage-report \
  --account-group-id $(ibmcloud enterprise account-group product-teams --output json | jq -r .id) \
  --billing-month 2026-08
```

```text
Account                Charges (USD)
orders-prod             4,812.33
orders-dev                642.10
catalog-prod             2,109.87
```

This is the enterprise-level view that Module 01's account-group
structure makes possible — one report spans every account under
`product-teams` without needing per-account login.

## Showback and chargeback with cost tags

```bash
ibmcloud billing account-usage --output json | \
  jq -r '.resources[] | select(.tags[]? | startswith("team:")) |
    "\(.tags[] | select(startswith("team:")))\t\(.charges)"' | \
  awk -F'\t' '{sum[$1]+=$2} END {for (t in sum) print t, sum[t]}'
```

```text
team:orders 6,891.20
team:catalog 2,109.87
```

Showback (reporting spend per team without billing them internally) is
the easy first step; chargeback (actually debiting a team's internal
budget) needs the tag discipline from Level 3, Module 07 enforced without
exception — one untagged large resource breaks the whole report's
accuracy silently.

## Detect cost anomalies before the monthly bill does

```bash
ibmcloud billing account-usage --output json > usage-current.json

python3 - <<'EOF'
import json
with open("usage-current.json") as f:
    data = json.load(f)
for r in data["resources"]:
    if r["charges"] > r.get("last_month_charges", 0) * 1.5 and r["charges"] > 50:
        print(f"ANOMALY: {r['resource_name']} jumped to ${r['charges']:.2f}")
EOF
```

```text
ANOMALY: Databases for PostgreSQL jumped to $312.40
```

A 50%+ month-over-month jump on a specific service is usually one of a
few causes: a forgotten dev resource left running at production size, an
autoscaler's `max-size` set too high and actually being hit, or a genuine
traffic increase — the anomaly check's job is to surface it quickly, not
diagnose which cause it is.

## Reserved capacity and committed-use discounts

```bash
ibmcloud billing offering-list --output json | jq '.[] | select(.type=="subscription")'
```

For steady-state workloads whose sizing has stabilized (the opposite of
Module 05's autoscaled traffic-following workloads), a committed-use
agreement with IBM trades flexibility for a lower rate — check actual
utilization trends over at least a full quarter before committing,
since a mis-sized reservation costs more than paying list price for
variable usage.

## Rightsizing as a recurring job, not a one-time audit

```bash
oc adm top pods -A --no-headers | awk '{print $1, $2, $3}' | sort -k3 -n
```

```bash
ibmcloud is instances --output json | \
  jq -r '.[] | select(.profile.name | test("bx2-8x32|bx2-16x64")) | .name'
```

Cross-reference instances provisioned at large profiles against their
actual `ibmcloud ob monitoring` CPU/memory utilization over the trailing
30 days (Level 4, Module 05's baseline habit, applied as a recurring
report instead of a one-off check) — a `bx2-16x64` instance sitting at 8%
average CPU for a month is a rightsizing candidate, not a capacity
decision that was necessarily wrong when originally made.

```bash
ibmcloud is instance-update prod-batch-worker --profile bx2-4x16
```

```text
Updating instance profile requires a reboot. Continue? [y/N]
```

## Idle and orphaned resource cleanup

```bash
ibmcloud is floating-ips --output json | jq '.[] | select(.target == null) | .address'
```

```text
203.0.113.44
```

An unattached floating IP still bills — a common, easy-to-miss leftover
from a deleted instance whose floating IP wasn't explicitly released.
Schedule a monthly sweep (a scripted check, not a manual click-through)
for unattached floating IPs, unattached block storage volumes, and
snapshots past their intended retention.

## FinOps dashboard: pull it together

```hcl
resource "ibm_resource_instance" "cost_dashboard_functions" {
  name              = "finops-anomaly-check"
  service           = "functions"
  plan              = "lite"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}
```

A scheduled Code Engine job or Cloud Function (from Level 1's serverless
module) running the anomaly-detection script monthly and posting results
to a chat channel turns this module's manual commands into an actual
recurring FinOps practice rather than something remembered only when the
bill is surprising.

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Showback numbers are only as good as tag coverage** — a single
  large, untagged resource silently understates every team's real spend
  in a showback report; audit tag coverage percentage as its own metric.
- **Committed-use discounts lock in a minimum spend** — committing based
  on a temporary traffic spike (rather than sustained baseline) can leave
  an account paying for capacity it no longer uses once the spike passes.
- **Instance profile changes need a reboot** — rightsizing a running
  production instance is a planned-maintenance-window activity, not
  something to script as a silent background change.
- **Free-tier and lite-plan resources still show as $0 in billing but
  count against account-level quotas** — a cost review focused only on
  billed dollars can miss quota exhaustion building up on the free side.

## Cheat sheet

| Task | Command |
|---|---|
| Enterprise usage report | `ibmcloud enterprise account-group-usage-report --account-group-id <id> --billing-month <YYYY-MM>` |
| List unattached floating IPs | `ibmcloud is floating-ips --output json \| jq '.[] \| select(.target == null)'` |
| Resize an instance profile | `ibmcloud is instance-update <name> --profile <profile>` |
| List subscription/reserved offerings | `ibmcloud billing offering-list` |
| View pod resource usage cluster-wide | `oc adm top pods -A` |

## Exercise

1. Write a script (Python or `jq`) that flags any resource whose current
   month charge exceeds 150% of the prior month.
2. Compute a showback total per `team:` tag from a sample billing usage
   JSON export.
3. Find (or simulate) an unattached floating IP and describe the cleanup
   steps and their cost impact.
4. Given a hypothetical instance sitting at 8% average CPU for a month,
   propose a rightsized profile and describe the maintenance-window
   process for applying it safely.
