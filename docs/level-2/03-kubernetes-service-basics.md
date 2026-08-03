# 03 · Kubernetes Service (IKS) Basics

The instance group in Module 2 auto-scales identical VSIs, but it has no
idea what's running on each one beyond "boot this image." **IBM Cloud
Kubernetes Service (IKS)** is the managed alternative for anything
container-based: you describe workloads declaratively, and Kubernetes
schedules, restarts, and scales the containers themselves, on top of worker
nodes that are still VPC VSIs under the hood.

## VPC clusters vs. classic clusters

IKS has two cluster types: **classic** (IBM's older, non-VPC networking)
and **VPC** (worker nodes are VSIs inside a VPC you control, like the one
from Module 1). New clusters should always be VPC clusters — classic exists
mainly for infrastructure provisioned years ago. This module uses VPC
clusters exclusively.

```bash
ibmcloud ks cluster create vpc-gen2 \
  --name mastery-iks \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-subnet-z1 --output json | jq -r .id) \
  --zone us-south-1 \
  --flavor bx2.4x16 \
  --workers 2 \
  --resource-group-id $(ibmcloud resource group mastery-path --output json | jq -r .[0].id)
```

**Gotcha:** cluster creation takes 20-40 minutes for the master alone —
much longer than any Level 1 resource. Don't assume something's stuck;
poll status instead of re-running the create command (which will just
fail with "cluster already exists").

```bash
ibmcloud ks cluster get --cluster mastery-iks
# State: deploying -> ... -> normal
```

## Worker pools across zones

A **worker pool** is a set of nodes with the same flavor and zone. For
resilience, add a pool spanning a second zone rather than putting every
node in one:

```bash
ibmcloud ks worker-pool create vpc-gen2 \
  --cluster mastery-iks \
  --name pool-z2 \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-subnet-z2 --output json | jq -r .id) \
  --zone us-south-2 \
  --flavor bx2.4x16 \
  --size-per-zone 2
```

Kubernetes' scheduler spreads pods across nodes on its own, but it only
spreads across *zones* if the worker pools themselves span zones — a
single-zone cluster is a single point of failure no matter how many
replicas a Deployment specifies.

## Configure kubectl

```bash
ibmcloud ks cluster config --cluster mastery-iks

kubectl get nodes
# NAME           STATUS   ROLES    AGE   VERSION
# 10.10.1.4      Ready    <none>   12m   v1.29.x+IKS
# 10.10.17.5     Ready    <none>   3m    v1.29.x+IKS
```

`ibmcloud ks cluster config` merges the cluster's credentials into
`~/.kube/config` — it doesn't print them, it configures `kubectl` directly,
which is easy to forget if you're expecting console output.

## Deploy a workload

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
        - name: hello-web
          image: icr.io/mastery-path/hello-web:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -o wide
# hello-web-... spread across both zones' nodes automatically
```

Always set `resources.requests` — the scheduler uses requests to decide
which node has room for a pod; without them, Kubernetes can over-pack a
node and every pod on it starves under load simultaneously.

## Expose it: Service and Ingress

A `Deployment` alone isn't reachable. A `Service` gives it a stable
in-cluster (or external) address:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f service.yaml
kubectl get service hello-web
# EXTERNAL-IP shows a VPC load balancer hostname once provisioned
```

`type: LoadBalancer` on IKS provisions an **Application Load Balancer for
VPC** behind the scenes — the same resource type from Module 2, just
managed by Kubernetes instead of by hand. For multiple services sharing one
public entry point, use an `Ingress` resource with the cluster's built-in
ALB Ingress controller instead of one `LoadBalancer` Service per app (each
`LoadBalancer` Service provisions its own billable ALB).

## Push your own image to IBM Cloud Container Registry

```bash
ibmcloud cr region-set us-south
ibmcloud cr namespace-add mastery-path

docker build -t icr.io/mastery-path/hello-web:1.0 .
docker push icr.io/mastery-path/hello-web:1.0

# Confirm it landed, and check for known vulnerabilities
ibmcloud cr images --restrict mastery-path
ibmcloud cr image-scan icr.io/mastery-path/hello-web:1.0
```

IKS worker nodes can pull from Container Registry in the same account and
region without extra image-pull secrets, as long as the cluster's default
service account has the `Reader` role on the registry namespace (granted
automatically for clusters and registries in the same account).

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud ks cluster create vpc-gen2 --vpc-id <id> --subnet-id <id> --zone <z> --flavor <f> --workers <n>` | Create a VPC-based cluster |
| `ibmcloud ks worker-pool create vpc-gen2 --cluster <c> --zone <z> --subnet-id <id>` | Add a worker pool in another zone |
| `ibmcloud ks cluster config --cluster <c>` | Merge cluster credentials into kubectl |
| `kubectl apply -f <file>` | Create/update resources from YAML |
| `kubectl get pods -o wide` | List pods and the node each landed on |
| `kubectl get service <name>` | Show a Service's external address |
| `ibmcloud cr namespace-add <ns>` / `docker push icr.io/<ns>/<image>` | Push a private image |
| `ibmcloud ks cluster rm --cluster <c>` | Delete the cluster (and its worker nodes) |

## Exercise

Create a two-zone IKS cluster with one worker pool per zone, deploy the
`hello-web` Deployment with 3 replicas, and expose it with a `LoadBalancer`
Service. Confirm with `kubectl get pods -o wide` that replicas landed on
nodes in both zones, then `kubectl delete node <one-node-name>` is *not*
the way to test failover on a managed cluster — instead cordon and drain
one node (`kubectl cordon <node>` then `kubectl drain <node>
--ignore-daemonsets`) and confirm the Deployment's pods reschedule onto the
remaining nodes. Delete the cluster with `ibmcloud ks cluster rm` when
done — worker nodes bill hourly for as long as the cluster exists.
