# 05 · VPC Networking Basics

Module 3 used a VPC and subnet to launch a VM without explaining the
networking model underneath. This module fills that gap: how traffic
actually gets in and out of a VPC, and the difference between the two
firewall-like controls IBM Cloud gives you — security groups and network
ACLs.

## VPCs, subnets, and zones

- A **VPC** is an isolated, region-scoped virtual network. Nothing inside it
  is reachable from outside by default.
- A **subnet** is a slice of the VPC's address space, and lives in exactly
  one **zone** (an isolated failure domain within the region — comparable to
  an "availability zone" elsewhere). Spreading subnets across zones is how
  you build zone-redundant applications.
- Every subnet gets an implicit local route to every other subnet in the
  same VPC — no extra config needed for same-VPC traffic.

```bash
ibmcloud is vpc-create app-vpc --resource-group-name mastery-path
ibmcloud is zones us-south

ibmcloud is subnet-create app-subnet-1 app-vpc --zone us-south-1 --ipv4-address-count 256
ibmcloud is subnet-create app-subnet-2 app-vpc --zone us-south-2 --ipv4-address-count 256
```

## Public gateways: how a private subnet reaches the internet

By default, instances in a subnet have no route to the public internet —
even with a floating IP attached, outbound-only internet access for
instances *without* a floating IP requires a **public gateway** attached to
that subnet.

```bash
ibmcloud is public-gateway-create app-pgw app-vpc --zone us-south-1 \
  --resource-group-name mastery-path

ibmcloud is subnet-update app-subnet-1 --pgw app-pgw
```

Think of it as: floating IPs give an instance an *inbound* address;
public gateways give a whole subnet *outbound* internet (like a NAT
gateway in other clouds).

## Security groups vs. network ACLs

Both filter traffic, but at different scopes and with different models:

| | Security Groups | Network ACLs |
|---|---|---|
| Applies to | Individual network interfaces (instances) | Every interface in a subnet |
| Default behavior | Deny all inbound, allow all outbound until you attach rules | Includes explicit default allow-all rules you can edit |
| Rule type | **Stateful** — a matched inbound rule automatically allows the matching return traffic | **Stateless** — you must write matching inbound *and* outbound rules yourself |
| Typical use | "This app server accepts HTTPS from anywhere, SSH from my IP" | "Nothing in this subnet talks to that subnet, no exceptions" |

```bash
# Security group: stateful, per-instance
ibmcloud is security-group-create web-sg app-vpc
ibmcloud is security-group-rule-add web-sg inbound tcp --port-min 443 --port-max 443 --remote 0.0.0.0/0

# Network ACL: stateless, per-subnet -- note the explicit outbound rule
# is required even though the security group above already allows it
ibmcloud is network-acl-create app-acl app-vpc
ibmcloud is network-acl-rule-add app-acl allow inbound tcp \
  --destination-port-min 443 --destination-port-max 443 --source 0.0.0.0/0
ibmcloud is network-acl-rule-add app-acl allow outbound tcp \
  --source-port-min 443 --source-port-max 443 --destination 0.0.0.0/0

ibmcloud is subnet-update app-subnet-1 --network-acl app-acl
```

In practice: use security groups as your primary, per-workload firewall, and
reserve network ACLs for coarse, subnet-wide guardrails (e.g. "block this
whole subnet from ever reaching that whole subnet") layered on top.

## Routing between subnets and out to the internet

```bash
# See the implicit + custom routes for a VPC
ibmcloud is vpc-routing-tables app-vpc

# A custom route, e.g. sending traffic for an on-prem range through a
# virtual network function instance instead of the default path
ibmcloud is vpc-routing-table-route-create app-vpc <routing-table-id> \
  --destination 10.50.0.0/16 \
  --action deliver \
  --next-hop-address 10.0.1.4 \
  --zone us-south-1
```

## Cheat sheet

| Concept | Command / Rule |
|---|---|
| List zones in a region | `ibmcloud is zones <region>` |
| Create a subnet in a zone | `ibmcloud is subnet-create <name> <vpc> --zone <z> --ipv4-address-count <n>` |
| Give a subnet outbound internet | `ibmcloud is public-gateway-create` + `subnet-update --pgw` |
| Per-instance, stateful firewall | Security groups (`ibmcloud is security-group-*`) |
| Per-subnet, stateless firewall | Network ACLs (`ibmcloud is network-acl-*`) |
| View VPC routes | `ibmcloud is vpc-routing-tables <vpc>` |

## Exercise

Create a VPC with two subnets in two different zones, attach a public
gateway to one of them, and create both a security group (allow inbound
443 from anywhere) and a network ACL (matching inbound + outbound 443
rules) and attach both to the gateway-enabled subnet. Explain in your own
words, in one sentence each, why the network ACL needs an explicit outbound
rule but the security group doesn't.
