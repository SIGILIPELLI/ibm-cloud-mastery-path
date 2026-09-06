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

## How It Actually Works

- **The 14.4x fast-burn threshold isn't an arbitrary round number — it's
  derived directly from the SLO's own error budget arithmetic.** A 28-day
  budget of 0.5% failing requests, if consumed at a constant rate, is
  fully exhausted in 28 days at burn-rate 1x; a 1-hour window sampling a
  burn rate of 14.4x extrapolates to exhausting that same budget in
  28/14.4 ≈ 1.94 days — the multiplier is chosen precisely so that
  sustaining it for the alert's short measurement window represents a
  genuinely budget-threatening trajectory, not a coincidence of common
  SRE folklore.
- **Multi-window burn-rate alerting exists because a single fixed-threshold
  alert can't distinguish a brief severe spike from a mild sustained
  leak, even though both eventually exhaust the same budget.** Evaluating
  the same underlying ratio (failing requests / total requests) over two
  different time windows with two different thresholds catches both
  shapes: a short window with a high threshold reacts fast to a severe
  spike before much budget is spent, while a long window with a low
  threshold accumulates enough signal to notice a leak too gradual to
  cross the short window's threshold at all.
- **"Mitigate first, root-cause later" works because a GitOps rollback
  (Module 07) and root-cause investigation are causally independent
  operations against different systems** — reverting a commit changes
  what Argo CD reconciles the cluster toward, which resolves the
  customer-facing symptom within one sync interval regardless of whether
  anyone yet understands why the prior deploy broke; the investigation
  meanwhile can proceed against logs, traces, and the retained bad commit
  without time pressure, since restoring service doesn't require or
  depend on that investigation completing first.
- **Automating a runbook step (like the failed-pod cleanup script) works
  by encoding the same imperative commands a human would type into a
  script or a Cloud Function triggered by the alert that used to page a
  person** — the underlying Kubernetes API calls (`field-selector`
  filtering, `delete pod`) are identical either way; what changes is
  whether a human's judgment and typing speed sit in the response's
  critical path. That's the actual mechanical distinction SRE draws
  between toil (a human executing a deterministic, repeatable procedure)
  and legitimate incident response (judgment applied to a genuinely novel
  situation the script wasn't written to handle).

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
