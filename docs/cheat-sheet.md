---
title: "IBM Cloud Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free IBM Cloud cheat sheet: 38 quick-reference tables from every level of the IBM Cloud Mastery Path, each linked to its full lesson."
---

# IBM Cloud Cheat Sheet

Quick-reference tables from **38 lessons** of the free IBM Cloud Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; the `ibmcloud` CLI](level-1/01-setup-cli.md)

| Command | Purpose |
|---|---|
| `ibmcloud login --apikey @file -r <region>` | Non-interactive login |
| `ibmcloud plugin install <name>` | Add a service-family plugin |
| `ibmcloud target -r <region> -g <group>` | Set region + resource group for the session |
| `ibmcloud target` | Show current login/region/group context |
| `ibmcloud regions` | List available regions |
| `ibmcloud resource groups` | List resource groups |
| `ibmcloud resource group-create <name>` | Create a resource group |
| `ibmcloud iam api-key-create <name> --file <path>` | Create a reusable API key |
| `ibmcloud logout` | End the current CLI session |

### [02 · IAM &amp; Security Basics](level-1/02-iam-security.md)

| Command | Purpose |
|---|---|
| `ibmcloud iam access-group-create <name>` | Create an access group |
| `ibmcloud iam access-group-policy-create <group> --roles <r> --resource-group-name <g>` | Grant a group a scoped policy |
| `ibmcloud iam access-group-user-add <group> <email>` | Add a user to a group |
| `ibmcloud iam service-id-create <name>` | Create a non-human identity |
| `ibmcloud iam service-policy-create <id> --roles <r> --service-name <svc>` | Grant a service ID a scoped policy |
| `ibmcloud iam service-api-key-create <key-name> <id> --file <path>` | Issue an API key for a service ID |
| `ibmcloud iam service-policies <id>` | Review a service ID's policies |
| `ibmcloud iam service-api-key-delete <id> <key-name>` | Revoke an API key |

### [03 · Virtual Servers (VPC)](level-1/03-virtual-servers-vpc.md)

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

### [04 · Cloud Object Storage](level-1/04-cloud-object-storage.md)

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> cloud-object-storage <plan> global` | Create a COS instance |
| `ibmcloud resource service-key-create <name> Writer --instance-name <cos>` | Issue API/HMAC credentials |
| `ibmcloud cos bucket-create --bucket <name> --region <r> --class <c>` | Create a bucket |
| `ibmcloud cos object-put --bucket <b> --key <k> --body <file>` | Upload an object |
| `ibmcloud cos object-get --bucket <b> --key <k> <dest>` | Download an object |
| `ibmcloud cos objects --bucket <b>` | List objects in a bucket |
| `ibmcloud cos bucket-website-put --bucket <b> --website-configuration <json>` | Enable static website hosting |
| `ibmcloud cos bucket-delete --bucket <b> --force` | Delete a bucket and its contents |

### [05 · VPC Networking Basics](level-1/05-vpc-networking.md)

| Concept | Command / Rule |
|---|---|
| List zones in a region | `ibmcloud is zones <region>` |
| Create a subnet in a zone | `ibmcloud is subnet-create <name> <vpc> --zone <z> --ipv4-address-count <n>` |
| Give a subnet outbound internet | `ibmcloud is public-gateway-create` + `subnet-update --pgw` |
| Per-instance, stateful firewall | Security groups (`ibmcloud is security-group-*`) |
| Per-subnet, stateless firewall | Network ACLs (`ibmcloud is network-acl-*`) |
| View VPC routes | `ibmcloud is vpc-routing-tables <vpc>` |

### [06 · Cloud Databases](level-1/06-cloud-databases.md)

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> databases-for-postgresql <plan> <region>` | Provision a PostgreSQL deployment |
| `ibmcloud resource service-key-create <key> Administrator --instance-name <db>` | Get connection credentials |
| `ibmcloud cdb deployment-backup <name>` | Trigger an on-demand backup |
| `ibmcloud cdb deployment-backups <name>` | List backups |
| `ibmcloud cdb deployment-scale <name> --memory <mb> --disk <mb>` | Resize a deployment |
| `ibmcloud resource service-instance-delete <name>` | Deprovision the deployment |

### [07 · Cloud Functions](level-1/07-cloud-functions.md)

| Command | Purpose |
|---|---|
| `ibmcloud fn action create <name> <file.js>` | Deploy a new action |
| `ibmcloud fn action update <name> <file.js> --web true` | Update an action / expose it as a web action |
| `ibmcloud fn action invoke <name> --result --param k v` | Invoke synchronously |
| `ibmcloud fn activation logs <id>` | Fetch logs for an async invocation |
| `ibmcloud fn trigger create <name> --feed <feed> --param k v` | Create an event trigger |
| `ibmcloud fn rule create <name> <trigger> <action>` | Bind a trigger to an action |
| `ibmcloud fn action create <name> --sequence a,b,c` | Chain actions into a sequence |
| `ibmcloud fn package create <name>` | Group related actions |
| `ibmcloud fn action delete <name>` | Delete an action |

### [08 · Monitoring &amp; Log Analysis](level-1/08-monitoring-log-analysis.md)

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> sysdig-monitor <plan> <region>` | Provision Cloud Monitoring |
| `ibmcloud resource service-instance-create <name> logdna <plan> <region>` | Provision Log Analysis |
| `ibmcloud resource service-key-create <key> Manager --instance-name <svc>` | Get an ingestion/dashboard key |
| `ibmcloud logging tail --instance <name>` | Stream live logs |
| `ibmcloud logging query --instance <name> --query "<q>" --since <window>` | Search historical logs |
| Sysdig agent install script | Ship OS-level metrics from a VSI |
| LogDNA agent install script | Ship OS-level logs from a VSI |

### [09 · Schematics (Terraform-based IaC)](level-1/09-schematics-iac.md)

| Command | Purpose |
|---|---|
| `ibmcloud schematics workspace new --name <n> --template-repo <url> --template-type <t>` | Create a workspace from a Git-hosted Terraform template |
| `ibmcloud schematics workspace update --id <id> --var "<k>=<v>" --var-type sensitive` | Set a sensitive input variable |
| `ibmcloud schematics plan --id <id>` | Preview changes |
| `ibmcloud schematics apply --id <id>` | Apply changes |
| `ibmcloud schematics logs --id <id> --latest` | Stream the latest job's logs |
| `ibmcloud schematics output --id <id>` | Show Terraform outputs |
| `ibmcloud schematics destroy --id <id>` | Tear down everything the workspace manages |

### [10 · Capstone Project — Visit Counter](level-1/10-capstone-project.md)

| Step | Command |
|---|---|
| Delete a Functions action/package | `ibmcloud fn action delete <name>` / `ibmcloud fn package delete <name>` |
| Empty + delete a COS bucket | `ibmcloud cos bucket-delete --bucket <b> --force` |
| Delete a database/service instance | `ibmcloud resource service-instance-delete <name> -f --recursive` |
| Delete a VSI and its networking | `instance-delete`, `floating-ip-release`, `security-group-delete`, `subnet-delete`, `key-delete`, `vpc-delete` |
| Destroy Schematics-managed infra | `ibmcloud schematics destroy --id <id>` |
| Final audit | `ibmcloud resource service-instances --long` and `ibmcloud is instances` |

## Level 2 · Intermediate

### [01 · VPC Design Deep Dive](level-2/01-vpc-design-deep-dive.md)

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

### [02 · Load Balancing &amp; Auto Scaling](level-2/02-load-balancing-autoscaling.md)

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

### [03 · Kubernetes Service (IKS) Basics](level-2/03-kubernetes-service-basics.md)

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

### [04 · Code Engine (Containers &amp; Serverless)](level-2/04-code-engine.md)

| Command | Purpose |
|---|---|
| `ibmcloud ce project create --name <n>` | Create a Code Engine project |
| `ibmcloud ce app create --name <n> --build-source . --strategy buildpacks --port <p>` | Deploy an app, building from source |
| `ibmcloud ce app create --name <n> --image <img> --port <p>` | Deploy an app from a prebuilt image |
| `ibmcloud ce app update <n> --min-scale <n> --max-scale <n>` | Set scaling bounds |
| `ibmcloud ce secret create --name <n> --from-literal K=V` | Create a secret |
| `ibmcloud ce app update <n> --env-from-secret <secret>` | Inject a secret as env vars |
| `ibmcloud ce job create --name <n> --image <img> --array-indices <a-b>` | Create a parallel batch job |
| `ibmcloud ce jobrun submit --job <n>` | Trigger one run of a job |
| `ibmcloud ce app get --name <n>` | Show an app's URL and current scale |

### [05 · Cloud Databases Deep Dive](level-2/05-cloud-databases-deep-dive.md)

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> databases-for-postgresql standard <region> -p '{...}'` | Provision with explicit per-member sizing |
| `ibmcloud cdb deployment-replica-create <primary> --name <replica>` | Create a read replica |
| `ibmcloud cdb deployment-backups <name>` | List backups / recovery points |
| `ibmcloud cdb deployment-restore <name> --point-in-time <ts>` | Restore to a point in time (creates a new deployment) |
| `ibmcloud cdb deployment-connections <name> --endpoint-type private` | Switch to the private endpoint |
| `ibmcloud cdb deployment-scale <name> --memory <mb> --disk <mb> --cpu <n>` | Live rolling resize |
| `ibmcloud cdb deployment-maintenance-update <name> --day <d> --hour <h>` | Set the patch maintenance window |

### [06 · Object Storage Lifecycle &amp; Security](level-2/06-object-storage-lifecycle-security.md)

| Command | Purpose |
|---|---|
| `ibmcloud cos bucket-lifecycle-configuration-put --bucket <b> --lifecycle-configuration <json>` | Set expiration/transition rules |
| `ibmcloud cos bucket-lifecycle-configuration-get --bucket <b>` | View current lifecycle rules |
| `ibmcloud cos bucket-create --bucket <b> --object-lock` | Create a bucket with Object Lock enabled (creation-time only) |
| `ibmcloud cos object-lock-configuration-put --bucket <b> --object-lock-configuration <json>` | Set a default retention mode/period |
| `ibmcloud cos object-retention-put --bucket <b> --key <k> --retention <json>` | Set retention on one object |
| `ibmcloud cos object-legal-hold-put --bucket <b> --key <k> --legal-hold <json>` | Toggle a legal hold on/off |
| `ibmcloud cos bucket-create --bucket <b> --ibm-sse-kp-customer-root-key-crn <crn>` | Encrypt a bucket with your own Key Protect key |
| `ibmcloud iam service-policy-create <id> --service-name cloud-object-storage --resource-type bucket --resource <bucket>` | Scope IAM access to one bucket |

### [07 · IAM Access Policies Deep Dive](level-2/07-iam-access-policies-deep-dive.md)

| Command | Purpose |
|---|---|
| `ibmcloud iam service-roles --service-name <svc>` | List a service's available fine-grained actions |
| `ibmcloud iam role-create <name> --service-name <svc> --actions <a,b>` | Define a custom role |
| `ibmcloud resource tag-attach --tag-names <k:v> --resource-name <r>` | Tag a resource |
| `ibmcloud iam access-group-policy-create <group> --resource-tags <k:v>` | Grant access scoped by tag |
| `ibmcloud iam authorization-policy-create <source-svc> <target-svc> <role>` | Let one service call another without a manual API key |
| `ibmcloud cbr zone-create --name <n> --addresses type=vpc,value=<crn>` | Define an allowed network zone |
| `ibmcloud cbr rule-create --resource-attributes <attrs> --zone-id <id> --enforcement-mode report` | Restrict a service to a zone (test mode first) |
| `ibmcloud iam access-group-policies <group>` | Audit a group's effective policies |

### [08 · CI/CD with Toolchains](level-2/08-cicd-toolchains.md)

| Resource / Command | Purpose |
|---|---|
| `ibm_cd_toolchain` | The container for a toolchain's tools |
| `ibm_cd_toolchain_tool_hostedgit` | Link an existing Git repo into the toolchain |
| `ibm_cd_toolchain_tool_pipeline` + `ibm_cd_tekton_pipeline` | Attach a Tekton pipeline |
| `ibm_cd_tekton_pipeline_definition` | Point the pipeline at a repo's `.tekton/` folder |
| `ibm_cd_tekton_pipeline_trigger` (`type = "scm"`) | Fire the pipeline on a Git push |
| `ibmcloud iam oauth-tokens --output json` | Get a bearer token for the Continuous Delivery API |
| `curl .../pipeline_runs -d '{"trigger_name": "..."}'` | Trigger a run manually (useful before wiring a real webhook) |

### [09 · Schematics Workspaces &amp; Modules](level-2/09-schematics-workspaces-modules.md)

| Command / Syntax | Purpose |
|---|---|
| `module "name" { source = "../../modules/x" ... }` | Compose a reusable module into a root config |
| `for_each = toset(var.zones)` | Repeat a resource block once per item in a variable |
| `output "name" { value = ... }` | Expose a value for another config (or another workspace) to consume |
| `ibmcloud schematics workspace new --name <n> --template-repo <url>` | Create one workspace per environment |
| `ibmcloud schematics output --id <id> --output json` | Read a workspace's outputs to feed into another workspace |
| `ibmcloud schematics workspace update --id <id> --var "<k>=<v>"` | Feed a value into a different workspace |
| `source = "git::<url>//<path>?ref=<tag>"` | Pin a module to an immutable Git ref |

### [10 · Project — Highly Available Web App](level-2/10-project-ha-web-app.md)

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

## Level 3 · Advanced

### [01 · Red Hat OpenShift on IBM Cloud](level-3/01-openshift-on-ibm-cloud.md)

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

### [02 · Advanced Networking (Transit Gateway, VPN, Direct Link)](level-3/02-advanced-networking.md)

| Task | Command |
|---|---|
| Create Transit Gateway | `ibmcloud tg gateway-create --name <n> --location <region>` |
| Attach a VPC | `ibmcloud tg connection-add <gw> --network-type vpc --network-id <crn>` |
| Create VPN gateway | `ibmcloud is vpn-gateway-create <n> <vpc> --subnet <s> --mode route` |
| Add VPN tunnel | `ibmcloud is vpn-gateway-connection-add <gw> <name> --peer-address <ip> ...` |
| Create Direct Link gateway | `ibmcloud dl gateway-create --name <n> --speed <mbps> --bgp-asn <asn>` |
| List TGW connections | `ibmcloud tg connections <gw>` |
| Route report | `ibmcloud tg route-report-create <gw>` |

### [03 · Security &amp; Compliance (Security Advisor, Key Protect)](level-3/03-security-compliance.md)

| Task | Command |
|---|---|
| Create Key Protect instance | `ibmcloud resource service-instance-create <n> kms tiered-pricing <region>` |
| Create root key | `ibmcloud kp key create <n> --instance-id <id> --standard-key false` |
| Rotate a key | `ibmcloud kp key rotate <key> --instance-id <id>` |
| Enable dual auth | `ibmcloud kp instance dual-auth-enable --instance-id <id>` |
| Attach SCC profile | `ibmcloud scc profile-attach --profile <name> --scope-id <account>` |
| View SCC results | `ibmcloud scc results --attachment-id <id>` |
| Create Activity Tracker route | `ibmcloud atracker route create --name <n> --rules '[...]'` |

### [04 · Event-Driven Architecture (Event Streams / Kafka)](level-3/04-event-driven-architecture.md)

| Task | Command |
|---|---|
| Create instance | `ibmcloud resource service-instance-create <n> messagehub standard <region>` |
| Create topic | `ibmcloud es topic-create <name> --instance <n> --partitions <n>` |
| List topics | `ibmcloud es topics --instance <n>` |
| Create service key | `ibmcloud resource service-key-create <n> Manager --instance-name <inst>` |
| Set mirroring topics | `ibmcloud es mirroring-topic-selection-set --instance <dr> --source-instance <src> --topics <pattern>` |
| Delete topic | `ibmcloud es topic-delete <name> --instance <n>` |

### [05 · Observability at Scale (Activity Tracker, Sysdig)](level-3/05-observability-at-scale.md)

| Task | Command |
|---|---|
| Create monitoring instance | `ibmcloud resource service-instance-create <n> sysdig-monitor graduated-tier <region>` |
| Attach cluster to monitoring | `ibmcloud ob monitoring config create --instance <n> --cluster <c>` |
| Attach cluster to logging | `ibmcloud ob logging config create --instance <n> --cluster <c>` |
| Tail logs with filter | `ibmcloud logging tail --instance <n> --filter "<query>"` |
| Create alert | `ibmcloud ob monitoring alert-create --instance <n> --condition '<expr>'` |
| List available metrics | `ibmcloud ob monitoring metrics --instance <n>` |

### [06 · Multi-Region &amp; Disaster Recovery](level-3/06-multi-region-dr.md)

| Task | Command |
|---|---|
| Create cross-region replica | `ibmcloud cdb deployment-create <name> --datacenter <region> --replica-of <id>` |
| Promote replica | `ibmcloud cdb deployment-promote-read-replica <name>` |
| Create cross-region bucket | `ibmcloud cos bucket-create --bucket <n> --region us-geo` |
| Create GLB | `ibmcloud cis glb-create --instance <cis> --name <fqdn> --default-pools <pool>` |
| Create origin pool | `ibmcloud cis glb-pool-create --instance <cis> --name <pool> --origins '[...]'` |

### [07 · Cost Management &amp; Governance](level-3/07-cost-management-governance.md)

| Task | Command |
|---|---|
| View account usage | `ibmcloud billing account-usage` |
| View resource-group usage | `ibmcloud billing resource-group-usage --resource-group-name <rg>` |
| Attach a cost tag | `ibmcloud resource tag-attach --resource-id <crn> --tag-names <k:v,...>` |
| Create a budget | `ibmcloud billing budget-create --name <n> --amount <usd> --resource-group-name <rg>` |
| List VPC quotas | `ibmcloud is quota-list` |
| Enable worker pool autoscaling | `ibmcloud ks worker-pool autoscale set --cluster <c> --worker-pool <p> --enable` |

### [08 · Advanced Schematics &amp; Terraform Modules](level-3/08-advanced-schematics-terraform.md)

| Task | Command |
|---|---|
| Create workspace | `ibmcloud schematics workspace new --file <config.json>` |
| Plan | `ibmcloud schematics plan --id <workspace-id>` |
| Apply | `ibmcloud schematics apply --id <workspace-id>` |
| View logs | `ibmcloud schematics logs --id <workspace-id> --act-id <activity-id>` |
| Refresh state (drift check) | `ibmcloud schematics workspace refresh-state --id <workspace-id>` |
| Destroy | `ibmcloud schematics destroy --id <workspace-id>` |

### [09 · API Management (API Connect)](level-3/09-api-management.md)

| Task | Command |
|---|---|
| Create API Connect instance | `ibmcloud resource service-instance-create <n> api-connect professional <region>` |
| Import an OpenAPI spec | `ibmcloud apic draft-apis:create --server <inst> <file.yaml>` |
| Create a product | `ibmcloud apic products:create --server <inst> --apis <api:ver> --plan <n>` |
| Publish to a catalog | `ibmcloud apic products:publish --server <inst> <product:ver> --catalog <cat>` |
| Create a developer org | `ibmcloud apic developer-orgs:create --server <inst> --title <name>` |
| List published products | `ibmcloud apic products:list --server <inst> --catalog <cat>` |

## Level 4 · Master

### [01 · Enterprise Account Structure &amp; IAM at Scale](level-4/01-enterprise-account-iam-at-scale.md)

| Task | Command |
|---|---|
| Create account group | `ibmcloud enterprise account-group-create --name <n> --parent <crn>` |
| Create account under a group | `ibmcloud enterprise account-create --name <n> --parent <crn>` |
| Create trusted profile | `ibmcloud iam trusted-profile-create <n>` |
| Add identity to profile | `ibmcloud iam trusted-profile-identity-add <profile> --identity-type crn --identifier <crn>` |
| Create dynamic access-group rule | `ibmcloud iam access-group-dynamic-rule-create <group> --identity-provider <url> --conditions '<expr>'` |
| Update account-wide IAM settings | `ibmcloud iam account-settings-update --mfa TOTP4ALL` |

### [02 · Landing Zone Architecture](level-4/02-landing-zone-architecture.md)

| Task | Command / reference |
|---|---|
| Landing zone module | `terraform-ibm-modules/landing-zone/ibm` |
| Create enterprise account | `ibmcloud enterprise account-create --name <n> --parent <crn>` |
| Check account activation state | `ibmcloud enterprise account <n> --output json \| jq .state` |
| Attach SCC profile post-bootstrap | `ibmcloud scc profile-attach --profile <name> --scope-id <account>` |
| Validate landing zone Terraform | `terraform validate` |

### [03 · Hybrid Cloud with Satellite](level-4/03-hybrid-cloud-satellite.md)

| Task | Command |
|---|---|
| Create a Satellite location | `ibmcloud sat location create --name <n> --managed-from <region> --zone <z>` |
| Generate host attach script | `ibmcloud sat host attach --location <loc> --host-provider ibm-satellite --script` |
| List hosts and health | `ibmcloud sat host ls --location <loc>` |
| Create Satellite OpenShift cluster | `ibmcloud sat cluster create --name <n> --location <loc> --kube-version <v>` |
| Assign storage class | `ibmcloud sat storage assign --location <loc> --storage-class <class>` |
| Attach location to Transit Gateway | `ibmcloud tg connection-add <gw> --network-type gre_tunnel --network-id <crn>` |

### [04 · Data &amp; AI Services (watsonx, Db2 Warehouse)](level-4/04-data-ai-services.md)

| Task | Command |
|---|---|
| Create Db2 Warehouse instance | `ibmcloud resource service-instance-create <n> dashdb-for-transactions performance <region>` |
| Create watsonx.ai instance | `ibmcloud resource service-instance-create <n> pm-20 lite <region>` |
| Create watsonx.governance instance | `ibmcloud resource service-instance-create <n> aiopenscale lite <region>` |
| List service instance plans | `ibmcloud catalog service-marketplace <service> --output json` |

### [05 · Performance &amp; Scalability Engineering](level-4/05-performance-scalability-engineering.md)

| Task | Command |
|---|---|
| View pod resource usage | `oc adm top pods -n <namespace>` |
| Create HPA | `oc autoscale deployment/<name> --min <n> --max <n> --cpu-percent <pct>` |
| Enable worker pool autoscaling | `ibmcloud ks worker-pool autoscale set --cluster <c> --worker-pool <p> --enable` |
| Create database read replica | `ibmcloud cdb deployment-create <name> --replica-of <id>` |
| Run a k6 load test | `k6 run --vus <n> --duration <t> <script>.js` |
| Create a CDN cache page rule | `ibmcloud cis page-rule-create --instance <cis> --target <pattern> --actions '<json>'` |

### [06 · Security Deep Dive (Hyper Protect, Key Management)](level-4/06-security-deep-dive.md)

| Task | Command |
|---|---|
| Create HPCS instance | `ibmcloud resource service-instance-create <n> hs-crypto standard <region> --parameters '{"units": <n>}'` |
| Check crypto unit status | `ibmcloud hpcs crypto-unit-info --instance <n>` |
| Create a KYOK root key | `ibmcloud hpcs key create <n> --instance <inst> --standard-key false` |
| Create a confidential-compute VSI | `ibmcloud is instance-create <n> --confidential-compute-mode sgx --profile <profile>` |
| Order a TLS certificate | `ibmcloud certificate-manager certificate-order --instance-id <id> --domains <fqdn>` |

### [07 · Automation Pipelines at Scale (GitOps + Schematics)](level-4/07-automation-pipelines-at-scale.md)

| Task | Command |
|---|---|
| Trigger a Schematics plan from CI | `ibmcloud schematics plan --id <workspace-id>` |
| Apply from CD (with force for automation) | `ibmcloud schematics apply --id <workspace-id> --force` |
| Create an Argo CD Application | `oc apply -f application.yaml` (kind: Application) |
| Render a kustomize overlay locally | `kubectl kustomize <overlay-path>` |
| Pause Argo CD self-heal for a hotfix | `argocd app set <app> --sync-policy none` |
| Revert a bad deploy | `git revert <sha> && git push` |

### [08 · FinOps &amp; Enterprise Cost Optimization](level-4/08-finops-cost-optimization.md)

| Task | Command |
|---|---|
| Enterprise usage report | `ibmcloud enterprise account-group-usage-report --account-group-id <id> --billing-month <YYYY-MM>` |
| List unattached floating IPs | `ibmcloud is floating-ips --output json \| jq '.[] \| select(.target == null)'` |
| Resize an instance profile | `ibmcloud is instance-update <name> --profile <profile>` |
| List subscription/reserved offerings | `ibmcloud billing offering-list` |
| View pod resource usage cluster-wide | `oc adm top pods -A` |

### [09 · Site Reliability Engineering on IBM Cloud](level-4/09-sre-on-ibm-cloud.md)

| Task | Command / concept |
|---|---|
| List available metrics for SLI queries | `ibmcloud ob monitoring metrics --instance <n>` |
| Create a burn-rate alert | `ibmcloud ob monitoring alert-create --condition '<promql-like expr>'` |
| Roll back via GitOps | `git revert <sha> && git push` |
| Error budget formula | `1 - SLO` over the SLO's measurement window |
| Fast-burn multiplier (common default) | 14.4x over 1h ≈ exhausts a 28-day budget in ~2 days |

---

Go deeper with the full [IBM Cloud Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
