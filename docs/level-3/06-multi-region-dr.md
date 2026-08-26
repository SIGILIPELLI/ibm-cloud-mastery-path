# 06 · Multi-Region & Disaster Recovery

Level 2's multi-zone VPC survives a zone outage. It does not survive a
regional outage — a rare but real event (network backbone failure, natural
disaster, regional control-plane incident). This module designs for that:
active-passive failover across `us-south` and `us-east`.

## Decide the DR posture first

| Posture | RTO | RPO | Cost |
|---|---|---|---|
| Backup & restore | Hours | Hours (last backup) | Lowest |
| Pilot light | 10s of minutes | Minutes | Low-medium |
| Warm standby | Minutes | Seconds-minutes | Medium-high |
| Active-active | Near zero | Near zero | Highest |

This module builds a **warm standby**: infrastructure exists in both
regions, but `us-east` runs at reduced capacity until a failover promotes
it.

## Mirror the VPC topology to a second region

Reuse the Level 2 Terraform module, parameterized by region, rather than
hand-building a second copy that drifts from the first:

```hcl
module "vpc_primary" {
  source           = "./modules/ha-vpc"
  region           = "us-south"
  address_prefix   = "10.10.0.0/16"
  worker_count     = 3
}

module "vpc_dr" {
  source           = "./modules/ha-vpc"
  region           = "us-east"
  address_prefix   = "10.20.0.0/16"
  worker_count     = 1   # warm standby: scaled down, not zero
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

Non-overlapping CIDRs (`10.10.0.0/16` vs `10.20.0.0/16`) matter here for
the same reason as Module 2's Transit Gateway note — if the two regions
are ever connected (for replication traffic or a global Transit Gateway),
overlap breaks routing.

## Database replication: Databases for PostgreSQL cross-region read replica

```bash
ibmcloud cdb deployment-create orders-db-dr \
  --datacenter us-east \
  --plan standard \
  --version 15 \
  --replica-of $(ibmcloud cdb deployment orders-db --output json | jq -r .id)
```

```text
Creating read replica orders-db-dr from orders-db...
This may take up to 20 minutes.
```

A read replica is a live standby, not a promoted primary — promoting it
during a real failover is a separate, deliberate action:

```bash
ibmcloud cdb deployment-promote-read-replica orders-db-dr
```

```text
Promoting orders-db-dr to a standalone deployment...
Warning: this action is irreversible and breaks replication with the source.
```

Practice the promotion path in a drill well before a real incident —
"we've never actually run this command" is the most common reason DR
plans fail under pressure.

## Cross-Region Cloud Object Storage for anything stateless-durable

```bash
ibmcloud cos bucket-create --bucket orders-archive-cr \
  --ibm-service-instance-id <cos-guid> \
  --region us-geo \
  --class standard
```

`us-geo` (cross-region) buckets replicate objects across `us-south`,
`us-east`, and `us-central` automatically — the simplest DR story in this
module, because IBM operates the replication, not you.

## DNS-based failover with Global Load Balancer (GLB)

```bash
ibmcloud cis glb-create \
  --instance cis-mastery \
  --name app.example.com \
  --fallback-pool pool-us-south \
  --default-pools pool-us-south \
  --health-check-region us-south \
  --proxy true
```

```bash
ibmcloud cis glb-pool-create \
  --instance cis-mastery \
  --name pool-us-east \
  --origins '[{"name":"us-east-lb","address":"198.51.100.20","enabled":true}]' \
  --check-regions WNAM \
  --minimum-origins 1
```

```text
Pool pool-us-east created.
```

Attach `pool-us-east` as the GLB's failover pool so a health check failure
against `us-south` automatically shifts DNS answers to `us-east` without
a manual DNS change during an incident — the difference between a runbook
step and an automated response.

## Terraform for the GLB failover pool

```hcl
resource "ibm_cis_origin_pool" "us_east_pool" {
  cis_id = data.ibm_cis.cis_instance.id
  name   = "pool-us-east"
  origins {
    name    = "us-east-lb"
    address = ibm_is_lb.dr_lb.hostname
    enabled = true
  }
  check_regions = ["WNAM"]
}

resource "ibm_cis_global_load_balancer" "app_glb" {
  cis_id           = data.ibm_cis.cis_instance.id
  domain_id        = data.ibm_cis_domain.example.domain_id
  name             = "app.example.com"
  fallback_pool_id = ibm_cis_origin_pool.us_east_pool.id
  default_pool_ids = [ibm_cis_origin_pool.us_south_pool.id]
  proxied          = true
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Run a failover drill

1. Simulate the primary being down: disable the `us-south` health check
   origin, or scale its worker pool to 0.
2. Confirm GLB shifts traffic (`dig app.example.com` should resolve to the
   `us-east` load balancer's address within the health-check interval).
3. Promote the read replica.
4. Point application config/secrets at the promoted database.
5. Time the whole exercise — that measured duration is your real RTO, not
   the number in the DR document.

## Gotchas

- **Read replica promotion is one-way** — there is no "un-promote"; a
  failback to the original region needs a fresh replica built in the
  reverse direction, which takes as long as the original replica did.
- **Cross-region COS costs more per GB** than regional storage and has
  slightly different consistency characteristics (eventual consistency
  across the replicated regions) — don't default every bucket to `us-geo`
  without a reason.
- **GLB health checks have their own latency** — a check interval of 60s
  plus a few failed-check threshold means real failover detection takes
  minutes, not seconds; factor that into RTO expectations.
- **DR infrastructure that's never exercised typically doesn't work** when
  actually needed — treat the failover drill as a recurring calendar
  event, not a one-time setup task.

## Cheat sheet

| Task | Command |
|---|---|
| Create cross-region replica | `ibmcloud cdb deployment-create <name> --datacenter <region> --replica-of <id>` |
| Promote replica | `ibmcloud cdb deployment-promote-read-replica <name>` |
| Create cross-region bucket | `ibmcloud cos bucket-create --bucket <n> --region us-geo` |
| Create GLB | `ibmcloud cis glb-create --instance <cis> --name <fqdn> --default-pools <pool>` |
| Create origin pool | `ibmcloud cis glb-pool-create --instance <cis> --name <pool> --origins '[...]'` |

## Exercise

1. Design (Terraform, parameterized module) a warm-standby VPC pair across
   two regions with non-overlapping CIDRs.
2. Create a cross-region database read replica and describe what changes
   (and what doesn't) when you promote it.
3. Configure a Global Load Balancer with a primary and failover pool.
4. Run a timed failover drill against your own setup and record the
   measured RTO versus your target.
