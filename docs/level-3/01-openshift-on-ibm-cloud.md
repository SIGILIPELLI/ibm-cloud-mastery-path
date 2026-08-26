# 01 · Red Hat OpenShift on IBM Cloud

Level 3 shifts from raw VPC infrastructure to a managed Kubernetes
distribution: **Red Hat OpenShift on IBM Cloud (ROKS)**. It's the same VPC,
subnets, and security groups from Level 2 underneath, but IBM operates the
control plane, patches the nodes, and layers on OpenShift's developer and
operator tooling — routes, build configs, and the OperatorHub — on top of
vanilla Kubernetes.

## OpenShift vs. plain Kubernetes on IBM Cloud

IBM Cloud offers two managed cluster products side by side:

| | IKS (Kubernetes Service) | ROKS (OpenShift) |
|---|---|---|
| Control plane | IBM-managed | IBM-managed |
| Ingress | NGINX ALB by default | HAProxy Router + `Route` objects |
| Image registry auth | IBM Cloud Container Registry | Built-in internal registry + ICR |
| Admin console | `kubectl` / IKS dashboard | `oc` CLI + OpenShift web console |
| Licensing | Free control plane | Included in worker node price on IBM Cloud |
| Security defaults | Standard RBAC | SecurityContextConstraints (SCC), stricter by default |

Pick ROKS when the team already knows OpenShift, needs SCCs for compliance,
or wants Operators/OperatorHub for day-2 management of databases and
middleware. Pick IKS for a lighter footprint. This module assumes ROKS.

## Provision a cluster into the Level 2 VPC

Reuse the multi-zone VPC and private subnets from Level 2's VPC design
module instead of building a new network:

```bash
ibmcloud ks cluster create vpc-gen2 \
  --name roks-mastery \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-subnet-z1 --output json | jq -r .id) \
  --zone us-south-1 \
  --flavor bx2.4x16 \
  --workers 2 \
  --version 4.14_openshift \
  --resource-group-name mastery-path \
  --disable-public-service-endpoint
```

```text
Creating cluster...
OK
The cluster is being created now.
```

`--disable-public-service-endpoint` keeps the Kubernetes/OpenShift API
reachable only from inside the VPC (or over VPN/Direct Link) — the default
posture for anything beyond a sandbox. Add a second zone's subnet with
`ibmcloud ks worker-pool zone add` once the cluster exists so worker nodes
survive a zone outage, the same reasoning as Level 2's multi-zone subnets.

## Point the CLI at the cluster

```bash
ibmcloud ks cluster config --cluster roks-mastery --admin
oc login -u apikey -p $(ibmcloud iam api-key-create cli-login --output json | jq -r .apikey) \
  --server=https://c1.us-south.containers.cloud.ibm.com:31234
oc get nodes
```

```text
NAME           STATUS   ROLES    AGE   VERSION
10.10.16.4     Ready    master   45m   v1.27.6+...
10.10.16.5     Ready    worker   40m   v1.27.6+...
10.10.16.6     Ready    worker   40m   v1.27.6+...
```

`ibmcloud ks cluster config --admin` writes a kubeconfig with a
cluster-admin certificate; day-to-day work should authenticate with an IAM
API key scoped to a project namespace instead, following the least-privilege
habit from Level 1's IAM module.

## Deploy and expose an app the OpenShift way

```bash
oc new-project store-frontend

oc new-app --name frontend \
  registry.us.icr.io/mastery-path/store-frontend:1.4 \
  --allow-missing-images

oc expose service/frontend --hostname frontend.roks-mastery.us-south.containers.appdomain.cloud
```

```text
route.route.openshift.io/frontend exposed
```

A `Route` is OpenShift's equivalent of a Kubernetes `Ingress`, but it's
handled by the built-in HAProxy router and gets a `*.containers.appdomain.cloud`
wildcard TLS certificate for free — no cert-manager setup needed for a
first pass, though production traffic should still bring its own certificate
via `oc create route edge`.

## SecurityContextConstraints: the OpenShift gotcha

The single most common "why won't my pod start" issue moving from vanilla
Kubernetes to ROKS is **SCCs**. By default, ROKS pods cannot run as root or
bind to privileged ports, even if the deployment YAML worked fine on plain
Kubernetes elsewhere:

```text
Error creating: pods "frontend-6f9d" is forbidden: unable to validate
against any security context constraint: provider "restricted": .spec.containers[0].securityContext.runAsUser:
Invalid value: 0: must be in the ranges: [1000700000, 1000709999]
```

Fix it in the image/deployment, not by loosening the cluster:

```yaml
securityContext:
  runAsNonRoot: true
  # leave runAsUser unset — let OpenShift assign from the namespace's
  # allocated UID range instead of hardcoding one
```

Only grant a broader SCC (`anyuid`, `privileged`) to a specific service
account when an image genuinely needs it, and treat that grant as an
auditable exception:

```bash
oc adm policy add-scc-to-user anyuid -z legacy-app-sa -n store-frontend
```

## Terraform for repeatable cluster creation

```hcl
resource "ibm_container_vpc_cluster" "roks" {
  name              = "roks-mastery"
  vpc_id            = ibm_is_vpc.ha_app_vpc.id
  kube_version      = "4.14_openshift"
  flavor            = "bx2.4x16"
  worker_count      = 2
  resource_group_id = data.ibm_resource_group.mastery_path.id

  zones {
    subnet_id = ibm_is_subnet.private_subnet_z1.id
    name      = "us-south-1"
  }
  zones {
    subnet_id = ibm_is_subnet.private_subnet_z2.id
    name      = "us-south-2"
  }
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## More OpenShift-specific gotchas

- **Worker pool version drift**: ROKS patches minor OpenShift versions on a
  schedule; `oc get clusterversion` and `ibmcloud ks cluster get` can show
  different "current" versions during a rolling upgrade — that's expected,
  not a fault.
- **Registry pull secrets**: images in IBM Cloud Container Registry need an
  image pull secret per namespace (`oc create secret docker-registry`), not
  just per cluster — a new namespace with no secret gets `ImagePullBackOff`.
- **Master API idle timeout**: the managed control plane sits behind a load
  balancer with an idle timeout around 5 minutes; long-lived `oc exec`
  sessions or `kubectl proxy` tunnels can silently drop and need a reconnect.
- **Node flavor changes require a new pool**: you cannot resize an existing
  worker pool's flavor in place — create a new pool with the target flavor,
  drain, then delete the old one.

## Cheat sheet

| Task | Command |
|---|---|
| Create ROKS cluster | `ibmcloud ks cluster create vpc-gen2 --vpc-id <id> --subnet-id <id> --version 4.14_openshift` |
| Fetch admin kubeconfig | `ibmcloud ks cluster config --cluster <name> --admin` |
| List nodes | `oc get nodes` |
| New app from image | `oc new-app --name <app> <image>` |
| Expose a Route | `oc expose service/<svc> --hostname <fqdn>` |
| Grant an SCC | `oc adm policy add-scc-to-user <scc> -z <sa>` |
| Add a worker pool zone | `ibmcloud ks worker-pool zone add` |
| List cluster versions | `ibmcloud ks versions` |

## Exercise

1. Provision a ROKS cluster into an existing (or newly created) multi-zone
   VPC, keeping the API endpoint private.
2. Deploy a container image and expose it via a `Route` with an
   edge-terminated TLS certificate.
3. Deliberately deploy a pod that sets `runAsUser: 0`, capture the SCC
   denial, then fix the deployment to run under the namespace's allocated
   UID range instead of granting `anyuid`.
4. Write the cluster as Terraform (`ibm_container_vpc_cluster`) referencing
   the VPC and subnet resources from your Level 2 state, and run
   `terraform validate` against it.
