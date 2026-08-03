# 02 · Load Balancing & Auto Scaling

Module 1 built a VPC with redundant subnets across zones — but redundant
subnets don't help if only one VSI is actually running in them. This module
adds the two pieces that turn "multiple zones" into "actually survives
losing one": an **Application Load Balancer for VPC** that spreads traffic
across healthy instances, and an **instance group with an auto scaling
policy** that adds or removes instances as load changes.

## Why a load balancer instead of a floating IP

Module 3 in Level 1 attached a floating IP directly to one VSI — fine for
SSH access to a single test instance, wrong for anything serving real
traffic. A floating IP points at exactly one network interface; if that
instance dies, so does every connection to it. A load balancer instead:

- Exposes one stable front-end address (or hostname).
- Health-checks every backend member and stops sending traffic to
  unhealthy ones automatically.
- Spreads load across members in multiple zones, so losing a zone loses
  only the capacity in it, not the whole service.

## Create the Application Load Balancer

```bash
# Public ALB spanning the public subnets from Module 1
ibmcloud is load-balancer-create ha-app-alb \
  --subnets public-subnet-z1,public-subnet-z2 \
  --public \
  --resource-group-name mastery-path

# Provisioning takes a few minutes -- poll until it's "active"
ibmcloud is load-balancer ha-app-alb
```

An ALB needs at least one subnet per zone it should have a presence in;
each zone's subnet gets its own front-end IP, and IBM Cloud handles the
DNS-level distribution across them under the load balancer's hostname.

## Backend pool and health monitor

A **pool** groups backend members and defines how traffic is balanced and
health-checked; a **pool member** is one backend target (a VSI's private
IP, or later, an instance group).

```bash
ibmcloud is load-balancer-pool-create web-pool ha-app-alb \
  --algorithm round_robin \
  --protocol http \
  --health-delay 5 \
  --health-retries 2 \
  --health-timeout 2 \
  --health-type http \
  --health-monitor-url /healthz

# Add an existing instance as a pool member (private IP, not floating IP)
ibmcloud is load-balancer-pool-member-create web-pool ha-app-alb \
  --port 8080 \
  --target-address 10.10.1.10
```

**Gotcha:** the health-check URL (`/healthz` above) has to return a 2xx
status from the app itself — a load balancer that can reach port 8080 but
gets a 500 from `/healthz` will correctly mark that member unhealthy and
stop routing to it, which looks like an outage even though the instance is
technically "up."

## Listener: what the front end actually accepts

```bash
ibmcloud is load-balancer-listener-create ha-app-alb \
  --port 80 \
  --protocol http \
  --default-pool web-pool
```

For HTTPS, upload a certificate to **Secrets Manager** first and reference
its CRN — the ALB terminates TLS at the listener, so backend members can
stay on plain HTTP inside the private network:

```bash
ibmcloud is load-balancer-listener-create ha-app-alb \
  --port 443 \
  --protocol https \
  --certificate-instance crn:v1:bluemix:public:secrets-manager:... \
  --default-pool web-pool
```

## Instance groups: the auto-scaled backend

Provisioning VSIs by hand doesn't scale (literally). An **instance
template** describes one instance's configuration; an **instance group**
launches and manages a set of them from that template, and can register
its own members directly with the ALB pool.

```bash
# Template: the "stamp" every scaled-out instance is cloned from
ibmcloud is instance-template-create web-template \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --zone us-south-1 \
  --profile cx2-2x4 \
  --image r134-... \
  --keys mastery-key \
  --primary-network-interface subnet=private-subnet-z1

# Instance group built from the template, spanning both zones' subnets
ibmcloud is instance-group-create web-group \
  --instance-template web-template \
  --membership-count 2 \
  --subnets private-subnet-z1,private-subnet-z2 \
  --load-balancer ha-app-alb \
  --load-balancer-pool web-pool \
  --port 8080
```

`--membership-count 2` is the group's baseline size before any scaling
policy kicks in — the group creates that many instances immediately,
distributed across the subnets given.

## Auto scaling policy

```bash
ibmcloud is instance-group-manager-create web-group \
  --name cpu-scaler \
  --policy-type target \
  --min-membership-count 2 \
  --max-membership-count 6 \
  --aggregation-window 90 \
  --cooldown 300

ibmcloud is instance-group-manager-policy-create web-group cpu-scaler \
  --name scale-on-cpu \
  --metric-type cpu \
  --metric-value 70
```

- **`--aggregation-window`** — how many seconds of sustained metric data
  triggers a scaling decision (avoids reacting to a one-second spike).
- **`--cooldown`** — minimum seconds between scaling actions, so the group
  doesn't oscillate up and down while metrics settle after each change.
- **`--metric-value 70`** — target average CPU percent; the manager adds
  instances when the group's average is sustained above this and removes
  them when it's comfortably below.

**Gotcha:** the min/max bounds are a hard budget ceiling as much as a
capacity guarantee — `--max-membership-count 6` caps cost under a traffic
spike, but also caps how much load the service can actually absorb before
requests start queuing or failing. Size the max based on what you can
afford to run continuously, not just what you hope never gets hit.

## Verify it's actually balancing and scaling

```bash
# Confirm pool members are all "healthy"
ibmcloud is load-balancer-pool-members web-pool ha-app-alb

# Watch group membership over time
watch -n 30 ibmcloud is instance-group-memberships web-group

# Hostname to actually hit
ibmcloud is load-balancer ha-app-alb --output json | jq -r .hostname
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud is load-balancer-create <name> --subnets <s1,s2> --public` | Create a public ALB across zones |
| `ibmcloud is load-balancer-pool-create <pool> <lb> --health-monitor-url <path>` | Create a backend pool with health checks |
| `ibmcloud is load-balancer-listener-create <lb> --port <p> --default-pool <pool>` | Front-end listener |
| `ibmcloud is instance-template-create <name> ...` | Define the instance "stamp" for scaling |
| `ibmcloud is instance-group-create <name> --instance-template <t> --load-balancer <lb> --load-balancer-pool <pool>` | Create an auto-managed, load-balanced group |
| `ibmcloud is instance-group-manager-create <group> --policy-type target --min-membership-count <n> --max-membership-count <n>` | Attach an auto scaling manager |
| `ibmcloud is instance-group-manager-policy-create <group> <manager> --metric-type cpu --metric-value <pct>` | Scale on a target metric |
| `ibmcloud is instance-group-memberships <group>` | List current group membership |

## Exercise

Attach the ALB and instance group from this module to the two-zone VPC you
built in Module 1's exercise. Set `--min-membership-count 2
--max-membership-count 4` with a CPU target of 70%, then generate load
against one instance (`stress-ng --cpu 2 --timeout 300s` works if
installed) and watch `ibmcloud is instance-group-memberships web-group`
grow past the baseline within a few minutes. Scale back down by stopping
the load, confirm membership shrinks toward the minimum after the cooldown
window, then delete the instance group, template, and load balancer before
moving on.
