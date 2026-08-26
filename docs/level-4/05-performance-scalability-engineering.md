# 05 · Performance & Scalability Engineering

Every module so far provisioned infrastructure sized by guesswork
("3 workers," "bx2.4x16"). This module makes sizing an evidence-based
exercise: load testing, capacity planning math, and the IBM-Cloud-specific
levers for scaling once evidence says you need to.

## Baseline before you tune anything

```bash
ibmcloud ob monitoring metrics --instance platform-monitoring | grep -E "cpu|memory|http_request"
```

```bash
oc adm top pods -n orders-frontend
```

```text
NAME                        CPU(cores)   MEMORY(bytes)
frontend-6f9d8c-4x2kq        180m         256Mi
frontend-6f9d8c-9dvbn        165m         240Mi
```

Never resize based on intuition alone — capture current CPU/memory
utilization and current request rate first, so a later change has a
before/after comparison instead of a guess about whether it helped.

## Load test against a non-production environment

```bash
k6 run --vus 200 --duration 5m load-test.js
```

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export default function () {
  const res = http.get('https://frontend-dev.roks-mastery.../orders');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

```text
     http_req_duration..............: avg=142ms  p(95)=310ms  p(99)=890ms
     http_req_failed.................: 0.42%
     vus_max.........................: 200
```

A p(99) nearly 3x the p(95) is a common early finding — a small subset of
requests hit a much slower path (often a cold cache or a lock contention
point), worth investigating with tracing (Level 3, Module 05) before
assuming more replicas alone fixes it.

## Horizontal Pod Autoscaling on OpenShift

```bash
oc autoscale deployment/frontend \
  --min 2 --max 10 --cpu-percent 65 \
  -n orders-frontend
```

```text
horizontalpodautoscaler.autoscaling/frontend autoscaled
```

```bash
oc get hpa -n orders-frontend
```

```text
NAME       REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS
frontend   Deployment/frontend   45%/65%       2         10        3
```

## Cluster autoscaling to back the pod autoscaler

An HPA that wants 10 replicas but the cluster only has capacity for 6
still leaves pods `Pending`. Pair it with worker pool autoscaling (Level
3, Module 07):

```bash
ibmcloud ks worker-pool autoscale set \
  --cluster platform-roks --worker-pool default \
  --enable --min-size 3 --max-size 8 --cooldown 300
```

Cooldown matters for cost, not just stability: a short cooldown chasing
every traffic blip causes node churn (and pod rescheduling disruption)
that a slightly longer cooldown avoids at the cost of slightly slower
scale-down.

## Database read scaling with read replicas

Databases for PostgreSQL supports read replicas for scaling read traffic
independent of write capacity — distinct from the cross-region DR replica
in Level 3's DR module, though it's the same underlying mechanism:

```bash
ibmcloud cdb deployment-create orders-db-read-1 \
  --datacenter us-south \
  --replica-of $(ibmcloud cdb deployment orders-db --output json | jq -r .id)
```

```text
Creating read replica orders-db-read-1...
```

Application code needs to explicitly route read-only queries to the
replica's connection string — Databases for PostgreSQL doesn't
transparently load-balance queries across primary and replicas for you.

## Capacity planning math, not just "add more"

```text
Target: handle 500 req/s at p(95) < 300ms
Observed from load test: 1 replica sustains ~60 req/s at that latency
                          before p(95) crosses 300ms

Naive replica count = 500 / 60 ≈ 9 replicas minimum
Add headroom for a lost zone (N+1 across zones) → 10-11 replicas
```

This arithmetic — not a round number picked by feel — is what an HPA
`--max` value and a worker pool `--max-size` should be derived from.
Re-run the load test after any change that alters per-replica capacity
(a code optimization, a bigger instance flavor) since the per-replica
throughput number is the one that changes, not the target.

## CDN and edge caching to reduce origin load entirely

Not every performance problem should be solved by scaling the origin —
static or slow-changing responses are cheaper to serve from IBM Cloud
Internet Services' CDN:

```bash
ibmcloud cis cache-purge --instance cis-mastery
ibmcloud cis page-rule-create --instance cis-mastery \
  --target "app.example.com/static/*" \
  --actions '{"cache_level":"cache_everything","edge_cache_ttl":3600}'
```

A cache hit at the edge means the request never reaches the OpenShift
cluster at all — the cheapest possible scaling, because it isn't scaling,
it's avoiding the work entirely.

## Terraform for the HPA-backing worker pool

```hcl
resource "ibm_container_vpc_worker_pool" "default" {
  cluster           = ibm_container_vpc_cluster.roks.id
  worker_pool_name  = "default"
  flavor            = "bx2.4x16"
  vpc_id            = ibm_is_vpc.platform_vpc.id
  worker_count      = 3

  autoscale {
    enabled  = true
    min_size = 3
    max_size = 8
  }
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **HPA CPU targets react to requests/limits, not raw usage** — a pod
  without a CPU `requests` value set has undefined HPA scaling behavior;
  always set explicit resource requests before attaching an HPA.
- **Cluster autoscaler add latency**: a new worker node joining a ROKS
  cluster takes several minutes — an HPA reacting to a spike faster than
  the cluster can add capacity still means a window of `Pending` pods;
  size `min-size` with expected baseline traffic, not the absolute
  minimum.
- **Load testing against production traffic patterns matters** — a
  synthetic test hitting one endpoint in a tight loop doesn't reproduce
  the mixed read/write, cache-hit/cache-miss pattern of real usage;
  results from an unrealistic test can justify the wrong fix.
- **Read replicas add replication lag** — a read-after-write pattern
  (create an order, immediately read it back) against a replica can
  return stale data; route strongly-consistent reads to the primary.

## Cheat sheet

| Task | Command |
|---|---|
| View pod resource usage | `oc adm top pods -n <namespace>` |
| Create HPA | `oc autoscale deployment/<name> --min <n> --max <n> --cpu-percent <pct>` |
| Enable worker pool autoscaling | `ibmcloud ks worker-pool autoscale set --cluster <c> --worker-pool <p> --enable` |
| Create database read replica | `ibmcloud cdb deployment-create <name> --replica-of <id>` |
| Run a k6 load test | `k6 run --vus <n> --duration <t> <script>.js` |
| Create a CDN cache page rule | `ibmcloud cis page-rule-create --instance <cis> --target <pattern> --actions '<json>'` |

## Exercise

1. Capture baseline CPU/memory for a running deployment, then run a load
   test against a non-production copy and record p(95)/p(99) latency.
2. Compute a target replica count from the load test's per-replica
   throughput and a stated traffic goal, showing the arithmetic.
3. Configure an HPA and a backing worker pool autoscaler sized from that
   calculation.
4. Identify one endpoint in your test that would benefit from CDN edge
   caching instead of more replicas, and explain why.
