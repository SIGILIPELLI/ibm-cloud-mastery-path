# 10 · Project — Highly Available Web App

The capstone for this level: one web app that survives losing an entire
availability zone. It combines the VPC design from Module 1, IBM Cloud
Kubernetes Service from Module 3 (which reuses the Application Load
Balancer concept from Module 2 automatically), and a multi-member
Databases for PostgreSQL deployment from Module 5 — connected over the
private endpoint, not the public one.

## Architecture

```text
                         Internet
                            |
              ha-app-alb (VPC Application Load Balancer,
              auto-provisioned by the Kubernetes Service below)
                            |
        +-------------------+-------------------+
        |                                       |
  Zone us-south-1                         Zone us-south-2
  IKS worker pool (2 nodes)                IKS worker pool (2 nodes)
  hello-web pods (replica set)             hello-web pods (replica set)
        |                                       |
        +-------------------+-------------------+
                            |
              private endpoint (no public internet hop)
                            |
              mastery-postgres-ha (Databases for PostgreSQL,
              multi-member, Module 5)
```

Every layer is the one you already built and tore down in an earlier
module's exercise — this project's job is standing them up together and
proving the composition survives a failure, not learning new services.

## 1. VPC: reuse (or rebuild) `ha-app-vpc`

```bash
ibmcloud is vpc-create ha-app-vpc \
  --resource-group-name mastery-path \
  --address-prefix-management manual

ibmcloud is vpc-address-prefix-create prefix-zone1 ha-app-vpc us-south-1 10.10.0.0/20
ibmcloud is vpc-address-prefix-create prefix-zone2 ha-app-vpc us-south-2 10.10.16.0/20

ibmcloud is subnet-create private-subnet-z1 ha-app-vpc \
  --zone us-south-1 --ipv4-address-count 128
ibmcloud is subnet-create private-subnet-z2 ha-app-vpc \
  --zone us-south-2 --ipv4-address-count 128
```

Two zones is the floor for "highly available" here; add a third
`us-south-3` subnet/worker pool/prefix throughout if you want the stretch
goal below.

## 2. Database: multi-member Postgres on the private endpoint

```bash
ibmcloud resource service-instance-create mastery-postgres-ha \
  databases-for-postgresql standard us-south \
  --resource-group-name mastery-path \
  -p '{
    "members_memory_allocation_mb": 4096,
    "members_disk_allocation_mb": 20480,
    "members_cpu_allocation_count": 2
  }'

ibmcloud cdb deployment-connections mastery-postgres-ha --endpoint-type private

ibmcloud resource service-key-create mastery-postgres-ha-cred Administrator \
  --instance-name mastery-postgres-ha
```

```sql
CREATE TABLE visit_counter (
    id INT PRIMARY KEY DEFAULT 1,
    count INT NOT NULL DEFAULT 0,
    CONSTRAINT single_row CHECK (id = 1)
);
INSERT INTO visit_counter (id, count) VALUES (1, 0);
```

## 3. IAM: a service ID scoped to this app only

```bash
ibmcloud iam service-id-create mastery-path-app \
  --description "hello-web capstone backend identity"

ibmcloud iam service-policy-create mastery-path-app \
  --roles Viewer \
  --service-name databases-for-postgresql \
  --resource-group-name mastery-path
```

Database credentials themselves still come from the service key above
(the platform role above only governs managing the deployment, not
connecting to it) — inject the connection details as a Kubernetes
`Secret`, never a hardcoded value in the Deployment manifest:

```bash
kubectl create secret generic postgres-creds \
  --from-literal=PGHOST=<host-from-service-key> \
  --from-literal=PGPORT=<port> \
  --from-literal=PGUSER=<user> \
  --from-literal=PGPASSWORD=<password>
```

## 4. Kubernetes Service: cluster spanning two zones

```bash
ibmcloud ks cluster create vpc-gen2 \
  --name mastery-iks \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-subnet-z1 --output json | jq -r .id) \
  --zone us-south-1 \
  --flavor bx2.4x16 \
  --workers 2 \
  --resource-group-id $(ibmcloud resource group mastery-path --output json | jq -r .[0].id)

ibmcloud ks worker-pool create vpc-gen2 \
  --cluster mastery-iks \
  --name pool-z2 \
  --vpc-id $(ibmcloud is vpc ha-app-vpc --output json | jq -r .id) \
  --subnet-id $(ibmcloud is subnet private-subnet-z2 --output json | jq -r .id) \
  --zone us-south-2 \
  --flavor bx2.4x16 \
  --size-per-zone 2

ibmcloud ks cluster config --cluster mastery-iks
```

## 5. Deploy the app: Deployment, Service, and HPA

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 4
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
          envFrom:
            - secretRef:
                name: postgres-creds
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {cpu: 250m, memory: 256Mi}
---
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
kubectl apply -f deployment.yaml
kubectl get service hello-web
# EXTERNAL-IP: a VPC Application Load Balancer hostname -- Module 2's ALB,
# provisioned automatically because the Service type is LoadBalancer
```

Auto scaling here is Kubernetes-native rather than the VSI instance group
from Module 2 — a **HorizontalPodAutoscaler** plays the same role at the
pod level:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-web
  minReplicas: 4
  maxReplicas: 12
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa hello-web --watch
```

## 6. Prove it's actually highly available

```bash
# Confirm replicas landed across both zones' nodes
kubectl get pods -o wide

# Simulate a zone-level failure: cordon and drain every node in zone 1
kubectl cordon <node-in-zone-1>
kubectl drain <node-in-zone-1> --ignore-daemonsets

# The Deployment should reschedule those pods onto zone 2's nodes;
# confirm the LoadBalancer's EXTERNAL-IP kept responding throughout
watch -n 5 curl -s -o /dev/null -w "%{http_code}\n" \
  "http://$(kubectl get service hello-web -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

kubectl uncordon <node-in-zone-1>
```

If the `curl` loop above never returns anything but `200` while zone 1's
node is drained, the app survived a simulated zone outage — the actual
point of this whole project.

## Cheat sheet

| Step | Command |
|---|---|
| Multi-zone VPC | `ibmcloud is vpc-create` + `subnet-create` per zone (Module 1) |
| Private-only database | `ibmcloud cdb deployment-connections <name> --endpoint-type private` |
| Scoped app identity | `ibmcloud iam service-id-create` + `service-policy-create` |
| Cluster across zones | `ibmcloud ks cluster create` + `ibmcloud ks worker-pool create` |
| Expose the app | `kubectl apply` a `Service` with `type: LoadBalancer` |
| Pod-level autoscaling | `kubectl apply` a `HorizontalPodAutoscaler` |
| Simulate a zone failure | `kubectl cordon` + `kubectl drain --ignore-daemonsets` |
| Confirm zero downtime | loop `curl` against the Service's external hostname during the drain |

## Teardown

```bash
kubectl delete -f hpa.yaml -f deployment.yaml
ibmcloud ks cluster rm --cluster mastery-iks -f
ibmcloud resource service-key-delete mastery-postgres-ha-cred -f
ibmcloud resource service-instance-delete mastery-postgres-ha -f --recursive
ibmcloud is subnet-delete private-subnet-z1 -f
ibmcloud is subnet-delete private-subnet-z2 -f
ibmcloud is vpc-delete ha-app-vpc -f
```

`mastery-iks` and `mastery-postgres-ha` both bill continuously — delete
them as soon as you've confirmed the failover test, the same discipline
from Level 1's capstone teardown.

## Exercise

Build the full architecture above, run the zone-drain failover test, and
confirm zero failed requests in the `curl` loop. Then generate CPU load
against the pods (`kubectl exec` into one and run a CPU-bound loop, or use
a load-testing tool against the Service's external hostname) and confirm
the `HorizontalPodAutoscaler` scales replicas up past 4 before settling
back down once load stops.

## Stretch goals

- **Third zone.** Add `us-south-3` — a subnet, a worker pool, and an
  address prefix — and repeat the failover test by draining an entire
  zone's worth of nodes instead of just one node.
- **CI/CD it.** Wire Module 8's Tekton pipeline to build `hello-web` and
  run `kubectl apply` on push, so this project deploys the same way a real
  team's would instead of via manual `kubectl` commands.
- **Manage it all with Schematics.** Reimplement Modules 1-2-5 (VPC,
  networking, database) as the reusable module pattern from Module 9,
  with one workspace per environment, and keep only the Kubernetes
  manifests as `kubectl apply` — the realistic split between
  infrastructure-as-code and application deployment on most real teams.
- **Add a read replica.** Point a reporting/health-check job at a Module
  5-style read replica instead of the primary, so analytical queries never
  compete with the app's own traffic for connections.
