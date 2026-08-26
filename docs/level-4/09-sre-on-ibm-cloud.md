# 09 · Site Reliability Engineering on IBM Cloud

Observability (Level 3, Module 05) tells you what's happening.
**SRE** is the practice built on top: defining what "reliable enough"
means numerically, alerting on the right things, and running incidents
with a process instead of ad hoc scrambling.

## Define an SLI, SLO, and error budget

```text
SLI (indicator):  proportion of orders-frontend requests returning < 500ms
                   and not a 5xx, measured over a rolling 28 days

SLO (objective):  99.5% of requests meet the SLI

Error budget:     0.5% of requests over 28 days may fail the SLI
                   = roughly 3.6 hours of full downtime equivalent per month
```

Everything else in this module — alerting thresholds, deployment
frequency decisions, incident severity — hangs off this one number. An
SLO chosen arbitrarily ("let's say 99.9%") without checking whether it's
achievable given current architecture (and current spend — Module 05 and
08 both feed this decision) sets the team up to burn its error budget
every month regardless of effort.

## Instrument the SLI as a real metric query

```bash
ibmcloud ob monitoring metrics --instance platform-monitoring | grep http_request
```

```text
sysdig_http_request_count{status=~"5.."}
sysdig_http_request_count{}
sysdig_http_request_time_bucket
```

```promql
# Error budget burn rate: fraction of requests failing SLI, last 1h
(
  sum(rate(sysdig_http_request_count{status=~"5.."}[1h]))
  +
  sum(rate(sysdig_http_request_time_bucket{le="0.5"}[1h])) * -1
  +
  sum(rate(sysdig_http_request_count{}[1h]))
) / sum(rate(sysdig_http_request_count{}[1h]))
```

## Multi-window burn-rate alerting

A single "error rate > X%" alert (Level 3, Module 05's approach) either
fires too often on noise or misses slow burns. SRE practice uses paired
fast/slow burn-rate windows:

```bash
ibmcloud ob monitoring alert-create \
  --instance platform-monitoring \
  --name error-budget-fast-burn \
  --condition 'error_budget_burn_rate_1h > 14.4' \
  --duration 120 \
  --severity critical \
  --notification-channel pagerduty:orders-oncall

ibmcloud ob monitoring alert-create \
  --instance platform-monitoring \
  --name error-budget-slow-burn \
  --condition 'error_budget_burn_rate_6h > 3' \
  --duration 1800 \
  --severity warning \
  --notification-channel slack:orders-team
```

A burn rate of 14.4x over 1 hour means the entire 28-day error budget
would be exhausted in about 2 days if sustained — worth paging immediately.
A 3x burn rate over 6 hours exhausts the budget in about 10 days — worth
a Slack notice, not a 2 a.m. page.

## Toil reduction: automate the runbook, not just document it

```bash
oc get pods -n orders-frontend --field-selector=status.phase=Failed \
  -o json | jq -r '.items[].metadata.name' | xargs -I{} oc delete pod {} -n orders-frontend
```

A documented runbook that a human executes by hand every time is toil —
the SRE discipline is turning the *common* incident responses into
scripts or automated remediations (a Cloud Function triggered by the
alert, for instance), keeping human judgment for the genuinely novel
incidents.

## Incident response process

```text
1. Alert fires → on-call acknowledges within 5 min (paging tool tracks this)
2. Declare severity (SEV1: customer-facing outage, SEV2: degraded, SEV3: internal only)
3. Open an incident channel; assign an Incident Commander (not necessarily
   the person who fixes it)
4. Mitigate first, root-cause later — e.g. roll back (Module 07's `git revert`
   pattern) before investigating why the bad deploy passed CI
5. Declare resolved once the SLI recovers, not once the fix is deployed
6. Postmortem within 48 hours, blameless, action items tracked to closure
```

"Mitigate first" is the single most valuable discipline: rolling back a
suspect deploy takes minutes via the GitOps pattern from Module 07, while
root-causing why it broke can take hours — do the fast, low-risk action
first even if it doesn't explain the failure yet.

## Postmortem template (keep it short and action-oriented)

```markdown
## Incident: orders-frontend 5xx spike, 2026-08-24

**Impact**: 12 minutes of ~40% error rate on /orders POST
**Detection**: fast-burn alert fired 3 min after onset; on-call acked in 4 min
**Root cause**: deploy introduced a null dereference on a new order field
**Mitigation**: `git revert`, GitOps synced within 90s of push
**Timeline**: [minute-by-minute]

**Action items**:
- [ ] Add a null-check unit test for the new field (owner: X, due: date)
- [ ] Add a canary deploy stage before full rollout (owner: Y, due: date)
```

Every action item needs an owner and a due date, or it's a wish, not a
plan — this is the difference between a postmortem that prevents a
repeat and one that's filed and forgotten.

## Terraform for the alerting policies

```hcl
resource "ibm_ob_monitoring_alert" "fast_burn" {
  instance_id = ibm_resource_instance.monitoring.guid
  name        = "error-budget-fast-burn"
  severity    = "critical"
  condition   = "error_budget_burn_rate_1h > 14.4"
  duration    = 120
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **An SLO set without a stakeholder conversation** becomes a source of
  friction — engineering and product need to agree the target is
  achievable and the cost of achieving it is worth paying, before it's
  wired into paging.
- **Burn-rate math depends on an accurate total-request denominator** —
  a metric query missing a subset of traffic (e.g., internal health
  checks skewing the denominator) silently distorts every burn-rate
  alert built on top of it.
- **"Mitigate first" can mask root cause if taken too far** — a rollback
  that resolves symptoms without ever completing the postmortem
  investigation lets the same bug ship again in a later, differently
  shaped deploy.
- **Blameless postmortems require actual practice, not just a template**
  — a team's first few postmortems under pressure tend to drift toward
  blame language by default; it's a discipline that needs active
  facilitation, not just a markdown heading.

## Cheat sheet

| Task | Command / concept |
|---|---|
| List available metrics for SLI queries | `ibmcloud ob monitoring metrics --instance <n>` |
| Create a burn-rate alert | `ibmcloud ob monitoring alert-create --condition '<promql-like expr>'` |
| Roll back via GitOps | `git revert <sha> && git push` |
| Error budget formula | `1 - SLO` over the SLO's measurement window |
| Fast-burn multiplier (common default) | 14.4x over 1h ≈ exhausts a 28-day budget in ~2 days |

## Exercise

1. Define an SLI, SLO, and resulting error budget in hours/month for a
   service from an earlier module.
2. Write a fast-burn and slow-burn alert condition and explain, with the
   arithmetic, what traffic pattern would trigger each.
3. Write a blameless postmortem for a hypothetical incident, including at
   least two action items with owners and due dates.
4. Identify one manual runbook step from an earlier module (e.g.,
   restarting failed pods) and describe how you'd automate it to reduce
   toil.
