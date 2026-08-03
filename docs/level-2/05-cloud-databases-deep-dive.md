# 05 · Cloud Databases Deep Dive

Level 1's database module (Module 6) provisioned a single PostgreSQL
deployment and connected to it — enough to store data, not enough to
survive a node failure or a bad migration. This module covers what "fully
managed" actually buys you across the **Databases for...** family: multi
member high availability, read replicas for scaling reads, point-in-time
recovery, and the private-endpoint connectivity that production workloads
should use instead of the public one.

## HA is about members, not instances

A Databases for PostgreSQL (or MongoDB, or most engines in the family)
deployment isn't one server — it's a set of **members** in a replica set.
The `standard` plan already provisions with replication for failover; what
you control is how much capacity each member gets and, for some engines,
how many members exist.

```bash
ibmcloud resource service-instance-create mastery-postgres-ha \
  databases-for-postgresql standard us-south \
  --resource-group-name mastery-path \
  -p '{
    "members_memory_allocation_mb": 4096,
    "members_disk_allocation_mb": 20480,
    "members_cpu_allocation_count": 2
  }'
```

**Gotcha:** these `members_*` values apply *per member*, not to the
deployment total — `members_memory_allocation_mb: 4096` on a 3-member
replica set provisions 3 × 4 GB, not one 4 GB pool split three ways. Sizing
math that ignores this reliably surprises people the first time they see
the bill.

## Read replicas: scale reads without scaling writes

For read-heavy workloads, a **read replica** is a separate deployment that
continuously streams from the primary and serves read-only queries,
keeping expensive analytical or reporting queries off the primary that
your application writes to.

```bash
ibmcloud cdb deployment-replica-create mastery-postgres-ha \
  --name mastery-postgres-read1 \
  --resource-group-name mastery-path

# Replicas have their own connection endpoint -- get its credentials
ibmcloud resource service-key-create mastery-postgres-read1-cred Administrator \
  --instance-name mastery-postgres-read1
```

Point read-only application code (reporting dashboards, exports) at the
replica's connection string and leave transactional writes going to the
primary. Replication lag is normally sub-second but is still *eventual*
consistency — don't read your own write back from a replica immediately
after writing it and expect to always see it.

## Point-in-time recovery

Beyond the on-demand backups from Level 1, Databases deployments support
**point-in-time recovery (PITR)** — restoring to any moment within the
retention window, not just to a specific backup snapshot.

```bash
# List recovery points / backups available
ibmcloud cdb deployment-backups mastery-postgres-ha

# Restore to a specific point in time into a NEW deployment
# (PITR always creates a new instance -- it never overwrites the source)
ibmcloud cdb deployment-restore mastery-postgres-ha \
  --point-in-time "2026-08-01T14:30:00Z"
```

**Gotcha:** restoring never touches the original deployment — it always
creates a fresh one. Plan for that in incident response: recovering from a
bad migration means standing up a new deployment, verifying the data, and
then repointing the application, not waiting for the original to "come
back."

## Private endpoint: the connection production should use

The public endpoint used in Level 1's exercise is fine for a laptop, wrong
for an app running inside the VPC from Module 1 — the private endpoint
keeps traffic on IBM's internal network instead of going out to the public
internet and back in.

```bash
# Enable the private endpoint (also requires VPE/Virtual Private Endpoint
# gateway setup in the consuming VPC for full private connectivity)
ibmcloud cdb deployment-connections mastery-postgres-ha --endpoint-type private

ibmcloud resource service-key mastery-postgres-ha-cred --output json \
  | jq '.credentials.connection.postgres.hosts'
```

Connecting from a VSI or Kubernetes pod inside `ha-app-vpc` (Module 1) over
the private endpoint means database traffic never crosses IBM Cloud's
public network boundary — one less thing an IAM misconfiguration or public
security-group rule could accidentally expose.

## Scaling and maintenance windows

```bash
# Live scale: memory/disk/cpu changes roll through members one at a time
ibmcloud cdb deployment-scale mastery-postgres-ha \
  --memory 8192 --disk 40960 --cpu 4

# Check (and set) the maintenance window for patches you don't control
ibmcloud cdb deployment-maintenance mastery-postgres-ha
ibmcloud cdb deployment-maintenance-update mastery-postgres-ha \
  --day sunday --hour 3
```

Rolling scale/patch operations take each member out of rotation briefly one
at a time, which is exactly why a single-member deployment (some smaller
plans) sees a real connection blip during patching, while a properly
multi-member HA deployment doesn't — the replica set keeps serving from
the members not currently being touched.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> databases-for-postgresql standard <region> -p '{...}'` | Provision with explicit per-member sizing |
| `ibmcloud cdb deployment-replica-create <primary> --name <replica>` | Create a read replica |
| `ibmcloud cdb deployment-backups <name>` | List backups / recovery points |
| `ibmcloud cdb deployment-restore <name> --point-in-time <ts>` | Restore to a point in time (creates a new deployment) |
| `ibmcloud cdb deployment-connections <name> --endpoint-type private` | Switch to the private endpoint |
| `ibmcloud cdb deployment-scale <name> --memory <mb> --disk <mb> --cpu <n>` | Live rolling resize |
| `ibmcloud cdb deployment-maintenance-update <name> --day <d> --hour <h>` | Set the patch maintenance window |

## Exercise

Provision `mastery-postgres-ha` with explicit `members_*` sizing, create a
read replica, and confirm the replica has its own connection endpoint via
a separate service key. Trigger an on-demand backup, then use
`ibmcloud cdb deployment-backups` to find its timestamp and practice a
point-in-time restore into a new deployment. Confirm the restored
deployment has the data you expect, then delete the replica and the
restored deployment (keep only the primary if you're continuing to later
modules, since Module 10's project reuses a database from this family).
