# 03 · Virtual Servers (VPC)

A **Virtual Server Instance (VSI)** is IBM Cloud's basic unit of compute
inside a **Virtual Private Cloud (VPC)** — a full virtual machine you can SSH
into, running in a network you control. This module provisions one from
scratch: the VPC, a subnet, an SSH key, a security group, and the instance
itself.

!!! warning "This module is billable"
    Unlike Cloud Object Storage or Cloud Functions, VSIs are not covered by
    a permanent free allowance on most plans. Use the smallest profile
    (`cx2-2x4`) and follow the teardown at the end of this lesson (or wait
    for Module 10's full cleanup) so it doesn't run longer than you intend.

## Create the VPC and a subnet

```bash
# A VPC is just an isolated network container -- no cost by itself
ibmcloud is vpc-create mastery-vpc --resource-group-name mastery-path

# List zones in your target region
ibmcloud is zones us-south

# A subnet lives in exactly one zone and gets an address range
ibmcloud is subnet-create mastery-subnet mastery-vpc \
  --zone us-south-1 \
  --ipv4-address-count 256
```

## Upload an SSH key

```bash
# Generate a keypair locally if you don't already have one
ssh-keygen -t ed25519 -f ~/.ssh/mastery-path -C "mastery-path"

# Upload the public half to IBM Cloud
ibmcloud is key-create mastery-key @~/.ssh/mastery-path.pub \
  --resource-group-name mastery-path
```

## Create a security group and open SSH

New VPCs come with a default security group that denies all inbound
traffic. Create a dedicated one for this instance instead of loosening the
default.

```bash
ibmcloud is security-group-create mastery-sg mastery-vpc \
  --resource-group-name mastery-path

# Allow inbound SSH only from your current IP, not 0.0.0.0/0
MY_IP=$(curl -s ifconfig.me)
ibmcloud is security-group-rule-add mastery-sg inbound tcp \
  --port-min 22 --port-max 22 \
  --remote "${MY_IP}/32"

# Allow all outbound so the instance can reach package repos etc.
ibmcloud is security-group-rule-add mastery-sg outbound all
```

## Create the instance

```bash
ibmcloud is instance-create mastery-vsi mastery-vpc us-south-1 cx2-2x4 mastery-subnet \
  --image r134-... \
  --keys mastery-key \
  --security-groups mastery-sg \
  --resource-group-name mastery-path

# Find a current Ubuntu image ID if you don't already have one
ibmcloud is images --visibility public | grep ubuntu
```

`cx2-2x4` is a compute-optimized profile with 2 vCPUs / 4 GB RAM — the
smallest practical size for a general-purpose test VSI.

## Attach a floating IP and connect

VSIs only get a private address by default. To reach one from your laptop
you need a **floating IP** — a public address you attach to the instance's
network interface.

```bash
ibmcloud is floating-ip-reserve mastery-fip \
  --nic mastery-vsi/eth0 \
  --resource-group-name mastery-path

# Read the address back out
ibmcloud is instance mastery-vsi --output json | grep -A2 floating

ssh -i ~/.ssh/mastery-path root@<floating-ip>
```

## How It Actually Works

- **A VSI is not a dedicated physical box — it's a KVM-based virtual
  machine multiplexed onto a shared hypervisor host, with your VPC's
  network fabric implementing isolation at the software-defined-networking
  layer rather than with physical cabling.** Every packet your VSI sends
  is intercepted by the hypervisor's virtual switch, which tags it with
  your VPC's network identifier before it ever reaches shared physical
  wiring — that's the actual mechanism guaranteeing tenant isolation, not
  the mere existence of separate account boundaries.
- **A floating IP is a NAT mapping maintained in the VPC's network
  edge, not an address configured on the VM's own network interface.**
  The VSI's real (private) network interface only ever knows its private
  subnet IP; the floating IP lives in a translation table on the fabric
  router, which rewrites the destination address of inbound packets and
  the source address of outbound ones. This is exactly why detaching and
  reattaching a floating IP to a different instance is instantaneous —
  you're editing one row of a routing table, not reconfiguring a NIC.
- **Boot volumes are backed by IBM Cloud Block Storage, itself replicated
  across independent physical drives inside the availability zone** — the
  VSI's disk I/O is actually network-attached storage traffic over the
  data center's internal fabric, not local disk. That's why you can
  resize or snapshot a boot volume without physically touching the
  instance: a snapshot is a copy-on-write pointer into the same
  distributed storage backend, not a byte-for-byte disk image.
- **Security groups are evaluated per network interface, stateful, and
  additive across every group attached to that interface** — a packet is
  allowed the instant any one rule permits it, and a reply to an already
  permitted outbound connection is auto-allowed without a matching
  inbound rule, because the fabric tracks connection state rather than
  re-evaluating every packet independently.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud is vpc-create <name>` | Create a VPC |
| `ibmcloud is subnet-create <name> <vpc> --zone <z> --ipv4-address-count <n>` | Create a subnet in a zone |
| `ibmcloud is key-create <name> @<pubkey>` | Upload an SSH public key |
| `ibmcloud is security-group-create <name> <vpc>` | Create a security group |
| `ibmcloud is security-group-rule-add <sg> inbound tcp --port-min <p> --port-max <p> --remote <cidr>` | Add an inbound firewall rule |
| `ibmcloud is instance-create <name> <vpc> <zone> <profile> <subnet> --image <id> --keys <key>` | Launch a VSI |
| `ibmcloud is floating-ip-reserve <name> --nic <instance>/eth0` | Attach a public IP |
| `ibmcloud is instance-delete <name>` | Delete a VSI |
| `ibmcloud is floating-ip-release <name>` | Release a floating IP |

## Teardown (run this once you're done experimenting)

```bash
ibmcloud is instance-delete mastery-vsi -f
ibmcloud is floating-ip-release mastery-fip -f
ibmcloud is security-group-delete mastery-sg -f
ibmcloud is subnet-delete mastery-subnet -f
ibmcloud is key-delete mastery-key -f
ibmcloud is vpc-delete mastery-vpc -f
```

## Exercise

Provision the VPC, subnet, SSH key, security group (SSH restricted to your
own IP), and a `cx2-2x4` instance as above, attach a floating IP, and SSH
in. Run `uname -a` over that connection to confirm it's really a fresh VM,
then run the full teardown block before moving on to the next module so the
instance isn't left running.
