# 01 · VPC Design Deep Dive

Level 1's VPC module (Module 3) built one subnet in one zone — enough to
launch a single VSI, not enough to survive a zone outage. This module
designs the topology every later Level 2 module assumes: a VPC with
public-facing and private-only tiers spread across multiple zones, address
prefixes managed on purpose instead of by default, and route tables that
control egress explicitly.

## Why a single subnet isn't a real design

A production VPC almost never looks like Module 3's one-subnet setup. Two
problems with a single subnet in a single zone:

- **No zone redundancy.** If that zone has an outage, everything in the VPC
  is unreachable — there's no second zone to fail over to.
- **No tiering.** Web servers, application servers, and databases all share
  one broadcast domain and one set of routing rules, so a security group
  mistake on one tier can expose another.

The fix is a **multi-tier, multi-zone** layout: a public subnet per zone for
anything that needs a public gateway (bastion hosts, NAT-bound workloads),
and one or more private subnets per zone for everything else, with routing
that keeps private subnets off the public internet by default.

## Plan the address space with prefixes

An IBM Cloud VPC gets a default address prefix per zone (`10.x.0.0/18`-ish
ranges) unless you turn that off and manage prefixes yourself. For a design
you actually understand later, manage them explicitly:

```bash
# Create the VPC without IBM's default address prefixes
ibmcloud is vpc-create ha-app-vpc \
  --resource-group-name mastery-path \
  --address-prefix-management manual

# Add one /20 prefix per zone you'll use
ibmcloud is vpc-address-prefix-create prefix-zone1 ha-app-vpc us-south-1 10.10.0.0/20
ibmcloud is vpc-address-prefix-create prefix-zone2 ha-app-vpc us-south-2 10.10.16.0/20
ibmcloud is vpc-address-prefix-create prefix-zone3 ha-app-vpc us-south-3 10.10.32.0/20
```

Carving a `/20` per zone up front leaves headroom for a public and several
private subnets in each zone without needing to touch the address plan
again later.

## Lay out public and private subnets per zone

```bash
# Public subnet per zone -- for anything that needs a public gateway route
ibmcloud is subnet-create public-subnet-z1 ha-app-vpc \
  --zone us-south-1 --ipv4-address-count 64

ibmcloud is subnet-create public-subnet-z2 ha-app-vpc \
  --zone us-south-2 --ipv4-address-count 64

# Private subnet per zone -- app/database tier, no direct public route
ibmcloud is subnet-create private-subnet-z1 ha-app-vpc \
  --zone us-south-1 --ipv4-address-count 128

ibmcloud is subnet-create private-subnet-z2 ha-app-vpc \
  --zone us-south-2 --ipv4-address-count 128
```

Two zones is the practical minimum for "highly available" — three is
better if the workload (and budget) supports it, since IBM Cloud regions
that offer three zones (like `us-south`) let you lose one and keep quorum
for anything using majority-vote replication.

## Public gateways: attach per zone, not per VPC

A **public gateway** is what lets instances in a subnet reach the internet
outbound (for package updates, calling external APIs) without themselves
having a floating IP. Gateways are zone-scoped, so a VPC spanning three
zones needs its own gateway per zone if every zone needs outbound access.

```bash
ibmcloud is public-gateway-create pgw-z1 ha-app-vpc --zone us-south-1
ibmcloud is public-gateway-create pgw-z2 ha-app-vpc --zone us-south-2

# Attach a gateway to the subnets that should get outbound internet access
ibmcloud is subnet-update public-subnet-z1 --public-gateway-id pgw-z1
ibmcloud is subnet-update private-subnet-z1 --public-gateway-id pgw-z1
```

**Gotcha:** attaching a public gateway to a "private" subnet gives every
instance in it outbound internet access (not inbound — that still needs a
floating IP or load balancer), which is often exactly what you want for
package installs but easy to forget you turned on. If a tier should have
*zero* internet reachability in either direction, don't attach a gateway to
it at all and route egress through a NAT appliance or Virtual Private
Endpoint instead.

## Route tables: control egress explicitly

Every VPC ships with a default routing table. For a real design, create a
dedicated table for the private tier so its egress rules are auditable on
their own, instead of buried in the default table alongside the public
tier's rules.

```bash
ibmcloud is vpc-routing-table-create ha-app-vpc \
  --name private-tier-rt

# Route all outbound traffic from the private tier through a NAT appliance
# instead of the public gateway, if you're keeping a stricter boundary
ibmcloud is vpc-routing-table-route-create ha-app-vpc private-tier-rt \
  --destination 0.0.0.0/0 \
  --action deliver \
  --next-hop 10.10.0.4 \
  --zone us-south-1

# Attach the table to the private subnets
ibmcloud is subnet-update private-subnet-z1 --routing-table private-tier-rt
```

Custom routing tables are what later modules lean on — Module 2's load
balancer needs a predictable path from its backend pool to app instances,
and Module 3's Kubernetes Service cluster expects VPC subnets that already
separate worker nodes from anything public-facing.

## Terraform equivalent (Schematics-ready)

Everything above maps directly onto the `ibm` Terraform provider, which is
how you'll manage this VPC once Module 9 introduces Schematics workspaces
built from modules:

```hcl
resource "ibm_is_vpc" "ha_app" {
  name                        = "ha-app-vpc"
  resource_group              = data.ibm_resource_group.group.id
  address_prefix_management   = "manual"
}

resource "ibm_is_vpc_address_prefix" "zone1" {
  name = "prefix-zone1"
  zone = "us-south-1"
  vpc  = ibm_is_vpc.ha_app.id
  cidr = "10.10.0.0/20"
}

resource "ibm_is_subnet" "private_z1" {
  name                     = "private-subnet-z1"
  vpc                      = ibm_is_vpc.ha_app.id
  zone                     = "us-south-1"
  ipv4_cidr_block          = "10.10.1.0/24"
  depends_on               = [ibm_is_vpc_address_prefix.zone1]
}
```

`depends_on` matters here — Terraform can't infer that a subnet's CIDR
needs an address prefix to exist first, since the prefix isn't referenced
by ID anywhere in the subnet resource.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud is vpc-create <name> --address-prefix-management manual` | Create a VPC without auto-assigned prefixes |
| `ibmcloud is vpc-address-prefix-create <name> <vpc> <zone> <cidr>` | Add a manually-managed address range in a zone |
| `ibmcloud is subnet-create <name> <vpc> --zone <z> --ipv4-address-count <n>` | Create a subnet sized by host count |
| `ibmcloud is public-gateway-create <name> <vpc> --zone <z>` | Create a zone-scoped internet gateway |
| `ibmcloud is subnet-update <subnet> --public-gateway-id <gw>` | Attach a gateway to a subnet |
| `ibmcloud is vpc-routing-table-create <vpc> --name <name>` | Create a custom routing table |
| `ibmcloud is vpc-routing-table-route-create <vpc> <table> --destination <cidr> --action <a> --next-hop <ip>` | Add a route |
| `ibmcloud is subnet-update <subnet> --routing-table <table>` | Attach a routing table to a subnet |

## Exercise

Build `ha-app-vpc` with manually-managed address prefixes across two zones,
a public and private subnet in each zone, one public gateway per zone
attached only to the public subnets, and a dedicated routing table for the
private subnets. Confirm with `ibmcloud is subnets --vpc ha-app-vpc` that
each subnet lands in the zone you expect, then tear it down with
`ibmcloud is vpc-delete ha-app-vpc -f` (this cascades to subnets, prefixes,
and gateways created directly under it, but delete gateways and routing
tables explicitly first if the cascade errors on dependencies).
