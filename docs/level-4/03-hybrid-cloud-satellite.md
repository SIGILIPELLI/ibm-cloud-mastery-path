# 03 · Hybrid Cloud with Satellite

Level 3's Direct Link and VPN modules connected IBM Cloud networking to
on-prem infrastructure. **IBM Cloud Satellite** goes further: it extends
IBM Cloud *services* — including a managed OpenShift/Kubernetes control
plane — to run on infrastructure you host, anywhere with a network path
back to IBM Cloud, including on-prem data centers or another cloud
provider.

## Why Satellite instead of just Direct Link + self-managed Kubernetes

Direct Link solves connectivity; it doesn't solve who patches, upgrades,
and manages a Kubernetes cluster running on your own hardware. Satellite
lets IBM Cloud manage the control plane and lifecycle of infrastructure
that physically never leaves your data center (or another cloud) —
useful for data residency requirements, existing hardware investment, or
edge locations with no practical path to full cloud migration.

## Create a Satellite location

```bash
ibmcloud sat location create \
  --name onprem-dc1 \
  --managed-from wdc \
  --zone dc1-zone-a --zone dc1-zone-b --zone dc1-zone-c \
  --resource-group-name mastery-path
```

```text
Creating location 'onprem-dc1'...
OK
Location onprem-dc1 created. Status: pending
```

`--managed-from wdc` picks the IBM Cloud region (Washington DC) hosting
the control-plane management components — the location itself can be
anywhere, but Satellite's control operations run from the chosen managing
region.

## Attach host machines

Satellite needs real (or virtual) machines registered as hosts before any
workload can run — this is the one place in the whole curriculum where
you attach non-IBM-Cloud-provisioned infrastructure:

```bash
ibmcloud sat host attach \
  --location onprem-dc1 \
  --host-provider ibm-satellite \
  --labels zone=dc1-zone-a \
  --script >attach-host.sh
```

```text
Attach script written to attach-host.sh
```

Run the generated script on each on-prem machine (RHEL/CentOS with
Satellite's agent requirements met) — it registers the machine with the
location's control plane over an outbound-only connection, so no inbound
firewall hole is needed on the on-prem side:

```bash
sudo bash attach-host.sh
```

```text
Registering host with location 'onprem-dc1'...
Host registered. Assigned to zone: dc1-zone-a
```

## Create a Satellite-hosted OpenShift cluster

```bash
ibmcloud sat cluster create \
  --name onprem-roks \
  --location onprem-dc1 \
  --kube-version 4.14_openshift \
  --zone dc1-zone-a --zone dc1-zone-b --zone dc1-zone-c \
  --host-labels zone=dc1-zone-a
```

```text
Creating cluster 'onprem-roks'...
This may take up to 60 minutes depending on host readiness.
```

From this point forward, `oc get nodes`, `oc new-app`, and every OpenShift
pattern from Level 3, Module 01 works identically — the cluster's API
server, upgrades, and health monitoring are managed by IBM Cloud even
though every node is physical (or virtual) hardware sitting in your data
center.

## Extend other IBM Cloud services to a Satellite location

```bash
ibmcloud sat config create \
  --name onprem-config \
  --location onprem-dc1

ibmcloud sat storage assign \
  --location onprem-dc1 \
  --storage-class satellite-storage-nfs
```

Databases for PostgreSQL, Event Streams, and Key Protect all support
deployment onto a Satellite location's Satellite Config-managed
infrastructure with the same CLI subcommands used elsewhere in this
curriculum, just pointed at the Satellite location instead of a public
region — the same `ibmcloud cdb deployment-create` command from Level 1,
with a `--satellite-location` flag instead of `--datacenter`.

## Link a Satellite cluster into the landing zone network

```bash
ibmcloud tg connection-add hub-tgw --network-type gre_tunnel \
  --network-id $(ibmcloud sat location onprem-dc1 --output json | jq -r .crn) \
  --base-network-type classic
```

A GRE tunnel (or Direct Link, for higher throughput needs) connects the
Satellite location's traffic into the same Transit Gateway hub built in
Level 3 — the on-prem OpenShift cluster and cloud-native VPC workloads
end up in one routable network, following the same hub-and-spoke
principle regardless of where the compute physically lives.

## Terraform for a Satellite location

```hcl
resource "ibm_satellite_location" "onprem_dc1" {
  location    = "onprem-dc1"
  managed_from = "wdc"
  zones        = ["dc1-zone-a", "dc1-zone-b", "dc1-zone-c"]
}

resource "ibm_satellite_cluster" "onprem_roks" {
  name          = "onprem-roks"
  location      = ibm_satellite_location.onprem_dc1.id
  kube_version  = "4.14_openshift"
  zones {
    id = "dc1-zone-a"
  }
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Host count minimums**: a cluster needs a minimum number of
  attached, healthy hosts per zone (control plane plus workers) — a
  location with too few registered hosts will sit at `pending` waiting
  for capacity that never arrives, with a non-obvious error.
- **Outbound-only connectivity is a feature, not a limitation to work
  around** — Satellite is deliberately designed so on-prem never needs an
  inbound rule from IBM Cloud; opening one defeats the security model and
  isn't necessary.
- **Host OS and hardware requirements are specific** (kernel version,
  minimum CPU/RAM per host) — a host that fails silent prerequisites will
  register but never become schedulable; check `ibmcloud sat host ls
  --location <loc> --output json | jq '.[] | {name, health}'` rather than
  assuming registration equals readiness.
- **Satellite Config drift**: config changes made directly on a host
  (bypassing Satellite Config) don't roll back automatically and can
  cause the managed control plane's view of cluster state to disagree
  with actual host state — treat Satellite hosts like any other IaC-
  managed resource, no manual edits.

## Cheat sheet

| Task | Command |
|---|---|
| Create a Satellite location | `ibmcloud sat location create --name <n> --managed-from <region> --zone <z>` |
| Generate host attach script | `ibmcloud sat host attach --location <loc> --host-provider ibm-satellite --script` |
| List hosts and health | `ibmcloud sat host ls --location <loc>` |
| Create Satellite OpenShift cluster | `ibmcloud sat cluster create --name <n> --location <loc> --kube-version <v>` |
| Assign storage class | `ibmcloud sat storage assign --location <loc> --storage-class <class>` |
| Attach location to Transit Gateway | `ibmcloud tg connection-add <gw> --network-type gre_tunnel --network-id <crn>` |

## Exercise

1. Create a Satellite location definition (three zones) and generate a
   host attach script — read through it and explain, in prose, what
   outbound connections it establishes.
2. Sketch (Terraform, unapplied) a Satellite-hosted OpenShift cluster
   using that location.
3. Describe how you'd connect a Satellite location's traffic into an
   existing Transit Gateway hub from Level 3, and why that keeps routing
   consistent regardless of where compute runs.
4. List two business reasons a team might choose Satellite over fully
   migrating to public-region VPC infrastructure.
