# 02 · Advanced Networking (Transit Gateway, VPN, Direct Link)

Level 2 built one VPC with public/private tiers. Real deployments rarely
stop at one VPC — a landing-zone-style estate has a hub VPC for shared
services, spoke VPCs per workload, on-prem data centers, and sometimes a
second cloud. This module connects them with **Transit Gateway**, **VPN
for VPC**, and **Direct Link**.

## Transit Gateway: hub-and-spoke between VPCs

Transit Gateway is IBM Cloud's routing hub — it connects multiple VPCs
(and, optionally, classic infrastructure and Direct Link/VPN connections)
without full-mesh peering.

```bash
ibmcloud tg gateway-create \
  --name hub-tgw \
  --location us-south \
  --global false \
  --resource-group-name mastery-path
```

```text
Gateway hub-tgw is being created
ID           crn:v1:bluemix:public:transit:us-south:...
Status       pending
```

Attach the VPCs (hub, plus each workload spoke):

```bash
ibmcloud tg connection-add hub-tgw --network-type vpc \
  --network-id $(ibmcloud is vpc hub-vpc --output json | jq -r .crn)

ibmcloud tg connection-add hub-tgw --network-type vpc \
  --network-id $(ibmcloud is vpc spoke-orders-vpc --output json | jq -r .crn)

ibmcloud tg connection-add hub-tgw --network-type vpc \
  --network-id $(ibmcloud is vpc spoke-inventory-vpc --output json | jq -r .crn)
```

```text
Connection is being added to gateway 'hub-tgw'...
OK
```

Once all three connections show `attached`, any subnet in any attached VPC
can route to any other by default — Transit Gateway does not enforce
segmentation for you. Segmentation is still your job, via security groups
and, for anything sensitive, **route filters** that restrict which prefixes
a spoke advertises or accepts:

```bash
ibmcloud tg route-report-create hub-tgw
```

## Gotcha: overlapping address space

Transit Gateway will happily attach two VPCs with overlapping CIDR blocks
— and then routing between them is undefined/broken. Plan address space
centrally (a shared IPAM sheet or Terraform module) *before* attaching
VPCs to a shared gateway, not after.

## VPN for VPC: encrypted site-to-site to on-prem

For connecting a single VPC back to an on-prem network over the public
internet with IPsec:

```bash
ibmcloud is vpn-gateway-create onprem-vpn ha-app-vpc \
  --subnet private-subnet-z1 --mode route

ibmcloud is vpn-gateway-connection-add onprem-vpn tunnel-to-hq \
  --peer-address 203.0.113.10 \
  --ike-policy-id $(ibmcloud is ike-policy-create hq-ike --ike-version 2 \
      --authentication-algorithm sha256 --encryption-algorithm aes256 \
      --dh-group 19 --output json | jq -r .id) \
  --ipsec-policy-id $(ibmcloud is ipsec-policy-create hq-ipsec \
      --authentication-algorithm sha256 --encryption-algorithm aes256 \
      --pfs group_19 --output json | jq -r .id) \
  --local-cidrs 10.10.0.0/16 --peer-cidrs 192.168.0.0/16
```

```text
Connection tunnel-to-hq is being created
Status   pending
```

Route mode (BGP-less static routing via CIDR lists) is simplest for a
single on-prem site; policy-based mode is deprecated on newer VPC gateways
— use route mode for new builds.

## Direct Link: private, non-internet connectivity

VPN encrypts over the public internet; **Direct Link** is a private
circuit (via a colocation provider or direct cross-connect) for
predictable latency and higher throughput — the choice for production
hybrid workloads, not just dev/test.

```bash
ibmcloud dl gateway-create \
  --name dl-hq \
  --speed 1000 \
  --bgp-asn 65010 \
  --bgp-base-cidr 169.254.0.0/16 \
  --global false \
  --metered false \
  --type dedicated \
  --location dal10 \
  --resource-group-name mastery-path
```

```text
Direct Link gateway dl-hq is being created.
Status   pending_approval
```

`pending_approval` is normal for `dedicated` connections — IBM's network
team and the colocation provider must provision the physical
cross-connect, which can take days, not minutes. `connect`-type Direct
Link (through a supported network provider's existing infrastructure) is
faster to stand up when a dedicated port isn't already in place.

## Wire Direct Link into Transit Gateway

```bash
ibmcloud tg connection-add hub-tgw --network-type directlink \
  --network-id $(ibmcloud dl gateway dl-hq --output json | jq -r .crn)
```

Now on-prem traffic through Direct Link, VPN-connected sites, and every
attached VPC share one routing domain through the hub gateway — the
landing-zone pattern IBM's reference architectures use.

## Terraform for the hub gateway

```hcl
resource "ibm_tg_gateway" "hub" {
  name           = "hub-tgw"
  location       = "us-south"
  global         = false
  resource_group = data.ibm_resource_group.mastery_path.id
}

resource "ibm_tg_connection" "hub_vpc" {
  gateway      = ibm_tg_gateway.hub.id
  network_type = "vpc"
  name         = "hub-vpc-conn"
  network_id   = ibm_is_vpc.hub_vpc.crn
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Other networking gotchas

- **Transit Gateway does not span regions by default** — set `--global
  true` (and expect a cost difference) to connect VPCs across regions
  through one gateway.
- **VPN gateway throughput caps**: a `route`-mode VPN gateway on VPC is
  capped well below Direct Link speeds (roughly 1–2 Gbps depending on
  policy); don't plan bulk-transfer workloads over VPN.
- **Security groups still apply** across a Transit Gateway connection —
  attaching a VPC doesn't bypass its security groups or network ACLs.
- **BGP ASN collisions**: reusing the same ASN on both ends of a Direct
  Link/VPN BGP session is a common setup mistake; keep a documented ASN
  range per environment.

## Cheat sheet

| Task | Command |
|---|---|
| Create Transit Gateway | `ibmcloud tg gateway-create --name <n> --location <region>` |
| Attach a VPC | `ibmcloud tg connection-add <gw> --network-type vpc --network-id <crn>` |
| Create VPN gateway | `ibmcloud is vpn-gateway-create <n> <vpc> --subnet <s> --mode route` |
| Add VPN tunnel | `ibmcloud is vpn-gateway-connection-add <gw> <name> --peer-address <ip> ...` |
| Create Direct Link gateway | `ibmcloud dl gateway-create --name <n> --speed <mbps> --bgp-asn <asn>` |
| List TGW connections | `ibmcloud tg connections <gw>` |
| Route report | `ibmcloud tg route-report-create <gw>` |

## Exercise

1. Create two VPCs with non-overlapping CIDR blocks and attach both to a
   Transit Gateway; verify a route report shows both prefixes.
2. Stand up a route-mode VPN gateway on one VPC with a placeholder peer
   address and IKE/IPsec policies, and inspect the connection status.
3. Write the Transit Gateway and one VPC attachment as Terraform and run
   `terraform validate`.
4. Document, in a short table, which of Transit Gateway, VPN, and Direct
   Link you'd choose for: two VPCs in the same region, a dev laptop VPN
   into a VPC, and a production database replicating to an on-prem data
   center.
