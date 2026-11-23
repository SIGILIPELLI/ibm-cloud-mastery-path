---
title: "IBM Cloud Interview Questions and Answers (179 Questions)"
description: "179 IBM Cloud interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the IBM Cloud Mastery Path."
---

# IBM Cloud Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free IBM Cloud Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; the `ibmcloud` CLI](level-1/01-setup-cli.md)

**Q1.** What does `ibmcloud login` do?

**Answer:** Authenticates the CLI to your IBM Cloud account

Use SSO or API keys.

**Q2.** What does `ibmcloud target -g <group> -r <region>` do?

**Answer:** Sets the resource group and region for commands

Context.

**Q3.** What's a resource group?

**Answer:** A logical container for organizing resources and access control

Required for most resources.

**Q4.** What does the free/Lite tier caution include?

**Answer:** Resources can bill if upgraded; delete practice resources

Check plans.

### [02 · IAM &amp; Security Basics](level-1/02-iam-security.md)

**Q5.** What does IAM decide?

**Answer:** Who (user, service ID) can do what (role) on which resource

Policies.

**Q6.** What are service IDs for?

**Answer:** Identities for applications/automation rather than people

With API keys.

**Q7.** What are the platform vs service access roles?

**Answer:** Platform roles manage resources; service roles control data access inside services

Viewer/Editor/Administrator and Reader/Writer/Manager.

**Q8.** Why use access groups?

**Answer:** Assign policies to groups and add users, instead of per-user policies

Scalable.

### [03 · Virtual Servers (VPC)](level-1/03-virtual-servers-vpc.md)

**Q9.** What is a VSI?

**Answer:** Virtual Server Instance in a VPC

IaaS compute.

**Q10.** What do you need before launching a VSI?

**Answer:** A VPC, subnet, SSH key and image/profile

Network first.

**Q11.** What's a profile?

**Answer:** The instance's vCPU/RAM size family (e.g. bx2-2x8)

Balanced/compute/memory.

**Q12.** How reach a private VSI?

**Answer:** Floating IP, bastion/VPN or Transit Gateway

Least exposure.

### [04 · Cloud Object Storage](level-1/04-cloud-object-storage.md)

**Q13.** What is COS?

**Answer:** S3-compatible object storage with buckets and objects

Cloud Object Storage.

**Q14.** What do storage classes represent?

**Answer:** Cost/performance for access frequency (Smart, Standard, Vault, Cold)

Lifecycle.

**Q15.** How should apps access COS?

**Answer:** Using service credentials/IAM with HMAC or API keys, least privilege

Rotate keys.

**Q16.** What does bucket versioning protect against?

**Answer:** Accidental deletes and overwrites

Immutable history.

### [05 · VPC Networking Basics](level-1/05-vpc-networking.md)

**Q17.** What's a VPC subnet?

**Answer:** An IP range in one zone for placing instances

Zone-scoped.

**Q18.** What's a security group?

**Answer:** A stateful firewall at the instance/interface level

Allow rules.

**Q19.** What's a network ACL?

**Answer:** A stateless subnet-level rule list (allow/deny)

Ordered rules.

**Q20.** What does a public gateway do?

**Answer:** Gives instances in a subnet outbound internet access

NAT-like.

### [06 · Cloud Databases](level-1/06-cloud-databases.md)

**Q21.** What does a managed database service handle?

**Answer:** Provisioning, backups, patching and scaling for you

Databases for PostgreSQL.

**Q22.** Which connection practice is right?

**Answer:** TLS with credentials from service credentials/Secrets Manager

Private endpoints preferred.

**Q23.** What do private endpoints do?

**Answer:** Keep traffic on IBM's private network

Reduce exposure.

**Q24.** Why enable backups/PITR?

**Answer:** Recover from corruption or mistakes

Test restores.

### [07 · Cloud Functions](level-1/07-cloud-functions.md)

**Q25.** What is Cloud Functions' programming model?

**Answer:** Event-driven actions triggered by HTTP, events or schedules

OpenWhisk based.

**Q26.** What's a cold start?

**Answer:** Latency when a new runtime container starts

Keep warm if needed.

**Q27.** What are triggers and rules?

**Answer:** Triggers emit events; rules connect them to actions

Wiring.

**Q28.** What limits actions?

**Answer:** Memory, timeouts and payload size

Design stateless.

### [08 · Monitoring &amp; Log Analysis](level-1/08-monitoring-log-analysis.md)

**Q29.** What does IBM Cloud Logs/Log Analysis collect?

**Answer:** Logs from services and apps for search and alerting

Agent or integration.

**Q30.** What do Monitoring metrics show?

**Answer:** Resource and app performance over time

Dashboards/alerts.

**Q31.** Why alert on symptoms?

**Answer:** Users feel symptoms; avoid noise from causes

Actionable.

**Q32.** What does Activity Tracker record?

**Answer:** Who did what to which resource (audit events)

Compliance.

### [09 · Schematics (Terraform-based IaC)](level-1/09-schematics-iac.md)

**Q33.** What is Schematics?

**Answer:** IBM's managed Terraform service (workspaces)

IaC.

**Q34.** What does a workspace hold?

**Answer:** A Terraform configuration, variables and state

Plan/apply.

**Q35.** Why use IaC?

**Answer:** Repeatable, reviewable, versioned infrastructure

No console drift.

**Q36.** What does `plan` show?

**Answer:** Changes Terraform would make

Review first.

### [10 · Capstone Project — Visit Counter](level-1/10-capstone-project.md)

**Q37.** What does the visit counter capstone combine?

**Answer:** Cloud Functions/Code, a database, COS and Schematics with monitoring

End to end.

**Q38.** How protect credentials?

**Answer:** Service IDs with least privilege and Secrets Manager

Hygiene.

**Q39.** How verify it works?

**Answer:** Test the endpoint and check logs/metrics

Observe.

**Q40.** What after finishing?

**Answer:** Destroy resources via Schematics to avoid cost

Clean up.

## Level 2 · Intermediate

### [01 · VPC Design Deep Dive](level-2/01-vpc-design-deep-dive.md)

**Q41.** Why use multiple zones in a VPC?

**Answer:** Survive a zone failure with subnets and instances spread across zones

HA.

**Q42.** What does address prefix management control?

**Answer:** The CIDR ranges available to subnets in each zone

Plan ranges.

**Q43.** What's Transit Gateway for?

**Answer:** Connecting multiple VPCs and on-prem networks through a hub

Scale connectivity.

**Q44.** What's a good practice for CIDR planning?

**Answer:** Avoid overlaps across VPCs and on-prem

Connectivity later.

### [02 · Load Balancing &amp; Auto Scaling](level-2/02-load-balancing-autoscaling.md)

**Q45.** What do VPC load balancers provide?

**Answer:** Distributing traffic across healthy instances (ALB/NLB)

Health checks.

**Q46.** What's an instance group?

**Answer:** A set of identical VSIs with autoscaling from a template

Elastic.

**Q47.** What triggers autoscaling policies?

**Answer:** Metrics like CPU or memory utilization

Targets.

**Q48.** What's draining?

**Answer:** Letting in-flight requests finish before removing an instance

Graceful.

### [03 · Kubernetes Service (IKS) Basics](level-2/03-kubernetes-service-basics.md)

**Q49.** What is IKS?

**Answer:** IBM Cloud Kubernetes Service, a managed Kubernetes

Workers in your VPC.

**Q50.** What does the managed control plane mean?

**Answer:** IBM runs and patches the Kubernetes masters

You manage workers/workloads.

**Q51.** What's a worker pool?

**Answer:** A set of worker nodes with the same flavor

Scale pools.

**Q52.** How expose apps?

**Answer:** Ingress/ALB or LoadBalancer services

Routing.

### [04 · Code Engine (Containers &amp; Serverless)](level-2/04-code-engine.md)

**Q53.** What's Code Engine?

**Answer:** A serverless platform for containers, jobs and apps scaling to zero

Simple deploy.

**Q54.** What are Code Engine jobs?

**Answer:** Run-to-completion batch workloads

Batch.

**Q55.** What's built into it?

**Answer:** Source-to-image builds from Git or Dockerfile

Builds.

**Q56.** When prefer Code Engine over IKS?

**Answer:** When you want less cluster operation for simple workloads

Trade-offs.

### [05 · Cloud Databases Deep Dive](level-2/05-cloud-databases-deep-dive.md)

**Q57.** What's high availability for a managed DB?

**Answer:** Multi-member deployments across zones with automatic failover

Replication.

**Q58.** What do read replicas help?

**Answer:** Scale reads and offload reporting

Lag.

**Q59.** What does scaling a DB involve?

**Answer:** Adjusting memory/disk/CPU per member (with possible rolling restarts)

Plan windows.

**Q60.** Why test restores?

**Answer:** Backups are unproven until restored

DR confidence.

### [06 · Object Storage Lifecycle &amp; Security](level-2/06-object-storage-lifecycle-security.md)

**Q61.** What do COS lifecycle rules do?

**Answer:** Transition or expire objects automatically by age

Cost control.

**Q62.** What does immutable object storage (WORM) provide?

**Answer:** Retention locks preventing deletion/modification for a period

Compliance.

**Q63.** How encrypt COS data?

**Answer:** Default at rest; use Key Protect/HPCS for customer-managed keys

Control.

**Q64.** What does bucket firewall do?

**Answer:** Restricts access to allowed IP ranges/networks

Defense.

### [07 · IAM Access Policies Deep Dive](level-2/07-iam-access-policies-deep-dive.md)

**Q65.** What are access policies composed of?

**Answer:** Subject, role(s) and resource (with optional attributes/conditions)

Fine grained.

**Q66.** What's the best practice for admin access?

**Answer:** Least privilege and separate admin identities with MFA

Reduce risk.

**Q67.** What do trusted profiles do?

**Answer:** Let compute resources/federated users assume roles without API keys

Keyless.

**Q68.** What does context-based restriction do?

**Answer:** Limit access by network location/context

Zero trust.

### [08 · CI/CD with Toolchains](level-2/08-cicd-toolchains.md)

**Q69.** What does an IBM Cloud toolchain automate?

**Answer:** Build, test and deploy via Continuous Delivery pipelines

Tekton pipelines.

**Q70.** What's Tekton?

**Answer:** Kubernetes-native pipeline resources (tasks, pipelines)

Cloud-native CI/CD.

**Q71.** What should pipelines use for secrets?

**Answer:** Secrets Manager/credentials store, not plain variables

Protect.

**Q72.** What gate should deploys have?

**Answer:** Tests and approvals for production

Safe delivery.

### [09 · Schematics Workspaces &amp; Modules](level-2/09-schematics-workspaces-modules.md)

**Q73.** Why split Schematics workspaces by environment/layer?

**Answer:** Isolate state and blast radius

Network vs app.

**Q74.** What are Terraform modules?

**Answer:** Reusable parameterized configuration blocks

DRY.

**Q75.** What shares values between workspaces?

**Answer:** Outputs consumed via remote state or variables

Layering.

**Q76.** What does drift detection show?

**Answer:** Differences between deployed resources and configuration

Fix drift.

### [10 · Project — Highly Available Web App](level-2/10-project-ha-web-app.md)

**Q77.** What does the HA web app project combine?

**Answer:** Multi-zone VPC, load balancer, autoscaling instance group and managed DB

Level 2 project.

**Q78.** How prove zone-failure survival?

**Answer:** Disable/stop a zone's instances and verify service continues

Chaos test.

**Q79.** What health checks matter?

**Answer:** LB checks on the app's real health endpoint

Meaningful.

**Q80.** What monitors availability?

**Answer:** Synthetic checks and alerts on error rate/latency

SLOs.

## Level 3 · Advanced

### [01 · Red Hat OpenShift on IBM Cloud](level-3/01-openshift-on-ibm-cloud.md)

**Q81.** What is Red Hat OpenShift on IBM Cloud (ROKS)?

**Answer:** A managed OpenShift (enterprise Kubernetes) service

Operators, routes.

**Q82.** What does OpenShift add to Kubernetes?

**Answer:** Built-in CI/CD, routes, security context constraints and an operator ecosystem

Platform.

**Q83.** What are Routes?

**Answer:** OpenShift's way to expose services externally

Ingress alternative.

**Q84.** What are SCCs?

**Answer:** Security Context Constraints controlling pod privileges

Hardening.

### [02 · Advanced Networking (Transit Gateway, VPN, Direct Link)](level-3/02-advanced-networking.md)

**Q85.** What do Direct Link and VPN provide?

**Answer:** Private or encrypted connectivity between on-prem and IBM Cloud

Hybrid.

**Q86.** What's a VPC peering alternative at scale?

**Answer:** Transit Gateway hub-and-spoke

Transitive.

**Q87.** What does Cloud Internet Services (CIS) provide?

**Answer:** DNS, global load balancing, WAF and DDoS protection

Edge.

**Q88.** What limits blast radius in networks?

**Answer:** Segmentation with subnets, ACLs and security groups

Defense.

### [03 · Security &amp; Compliance (Security Advisor, Key Protect)](level-3/03-security-compliance.md)

**Q89.** What does Key Protect/HPCS provide?

**Answer:** Key management with BYOK/KYOK for envelope encryption

Control.

**Q90.** What does Security and Compliance Center do?

**Answer:** Continuous posture and compliance scanning against profiles

Findings.

**Q91.** What's shared responsibility?

**Answer:** IBM secures the platform; you secure configuration and data

Know the line.

**Q92.** What does Secrets Manager do?

**Answer:** Stores and rotates secrets and certificates

Central.

### [04 · Event-Driven Architecture (Event Streams / Kafka)](level-3/04-event-driven-architecture.md)

**Q93.** What does Event Streams provide?

**Answer:** Managed Kafka for event streaming

Pub/sub.

**Q94.** Why event-driven architecture?

**Answer:** Loose coupling and asynchronous scalability

Decoupled services.

**Q95.** What must consumers be?

**Answer:** Idempotent because delivery can repeat

At-least-once.

**Q96.** What's a dead-letter pattern?

**Answer:** Routing repeatedly failing messages aside for inspection

Safe handling.

### [05 · Observability at Scale (Activity Tracker, Sysdig)](level-3/05-observability-at-scale.md)

**Q97.** What does observability at scale require?

**Answer:** Logs, metrics and traces correlated across services

Context.

**Q98.** What's distributed tracing for?

**Answer:** Following a request across microservices to find latency

Instana/OTel.

**Q99.** What's alert fatigue?

**Answer:** Too many low-value alerts causing real issues to be missed

Tune.

**Q100.** What do SLOs drive?

**Answer:** Alerting and reliability targets tied to users

Error budgets.

### [06 · Multi-Region &amp; Disaster Recovery](level-3/06-multi-region-dr.md)

**Q101.** What does multi-region DR need?

**Answer:** Replicated data, deployable infrastructure and tested failover

RTO/RPO.

**Q102.** What are RTO and RPO?

**Answer:** Time to recover and acceptable data loss window

Drive design.

**Q103.** Which DR strategy is cheapest?

**Answer:** Backup and restore (slowest RTO)

Trade-offs.

**Q104.** How validate DR?

**Answer:** Regular failover tests/game days

Prove it.

### [07 · Cost Management &amp; Governance](level-3/07-cost-management-governance.md)

**Q105.** How track IBM Cloud cost?

**Answer:** Billing/usage reports, tags and budgets with alerts

FinOps.

**Q106.** What does tagging enable?

**Answer:** Cost allocation by team/project

Showback.

**Q107.** What are quick savings?

**Answer:** Delete idle resources and right-size

Hygiene.

**Q108.** What governance control limits spend?

**Answer:** Budgets, resource quotas and policies

Guardrails.

### [08 · Advanced Schematics &amp; Terraform Modules](level-3/08-advanced-schematics-terraform.md)

**Q109.** What does a Terraform CI check include?

**Answer:** fmt, validate, lint/security scan and plan review

Pipeline.

**Q110.** What does remote state locking prevent?

**Answer:** Concurrent applies corrupting state

Safety.

**Q111.** Why pin provider versions?

**Answer:** Reproducible plans

Stability.

**Q112.** What are policy-as-code checks?

**Answer:** Automated guardrails evaluated on plans (OPA/Sentinel-like)

Compliance.

### [09 · API Management (API Connect)](level-3/09-api-management.md)

**Q113.** What does API Connect/API Gateway provide?

**Answer:** Publishing, securing and governing APIs with plans, quotas and analytics

API management.

**Q114.** How secure APIs?

**Answer:** AuthN/Z, rate limits, validation and TLS

Defense.

**Q115.** What are API plans?

**Answer:** Usage tiers/quotas for consumers

Monetize/limit.

**Q116.** Why version APIs?

**Answer:** Evolve without breaking consumers

Compatibility.

### [10 · Project — Multi-Tier Microservices Platform](level-3/10-project-microservices-platform.md)

**Q117.** What does the microservices platform combine?

**Answer:** ROKS, networking, security, events, observability and DR

Level 3 project.

**Q118.** How prove resilience?

**Answer:** Failure injection and monitored recovery

Evidence.

**Q119.** What's the delivery model?

**Answer:** GitOps/CI-CD with automated tests

Repeatable.

**Q120.** What should be documented?

**Answer:** Architecture, runbooks and ownership

Operate.

## Level 4 · Master

### [01 · Enterprise Account Structure &amp; IAM at Scale](level-4/01-enterprise-account-iam-at-scale.md)

**Q121.** Why enterprise accounts?

**Answer:** Central governance and billing across multiple accounts

Account hierarchy.

**Q122.** What does IAM at scale rely on?

**Answer:** Access groups, trusted profiles and consistent policy templates

Automation.

**Q123.** What's account separation for?

**Answer:** Isolating environments/teams with guardrails

Blast radius.

**Q124.** How manage across accounts?

**Answer:** Enterprise-level policies and service ID/automation

Consistency.

### [02 · Landing Zone Architecture](level-4/02-landing-zone-architecture.md)

**Q125.** What's a landing zone?

**Answer:** A secured, pre-configured multi-account foundation with networking and IAM

Deployable architecture.

**Q126.** What does it include?

**Answer:** Network segmentation, IAM, logging, key management and guardrails

Secure by default.

**Q127.** Why deploy it with IaC?

**Answer:** Repeatable and auditable foundations

Terraform.

**Q128.** What's a management/workload VPC split?

**Answer:** Separate admin and app networks connected via Transit Gateway

Isolation.

### [03 · Hybrid Cloud with Satellite](level-4/03-hybrid-cloud-satellite.md)

**Q129.** What is IBM Cloud Satellite?

**Answer:** Extends IBM Cloud services to on-prem/edge locations managed from the cloud

Hybrid.

**Q130.** Why hybrid cloud?

**Answer:** Data residency, latency or existing infrastructure

Constraints.

**Q131.** What's a Satellite location?

**Answer:** Your own infrastructure registered as a control-plane-managed location

Hosts connect.

**Q132.** What's the operational trade-off?

**Answer:** You manage the hardware and connectivity

Responsibility.

### [04 · Data &amp; AI Services (watsonx, Db2 Warehouse)](level-4/04-data-ai-services.md)

**Q133.** What do data and AI services include?

**Answer:** Databases, analytics and watsonx services

Platform.

**Q134.** What does watsonx.ai provide?

**Answer:** Foundation models and tools to build generative AI

Prompt/tune.

**Q135.** What's data governance in AI?

**Answer:** Lineage, access control and quality for training data

Trust.

**Q136.** How protect sensitive data in AI?

**Answer:** Minimize, mask and control access with encryption

Privacy.

### [05 · Performance &amp; Scalability Engineering](level-4/05-performance-scalability-engineering.md)

**Q137.** What drives performance engineering?

**Answer:** Measure, find bottlenecks, then scale or optimize

Evidence.

**Q138.** What does load testing reveal?

**Answer:** Capacity and failure behavior under stress

Before launch.

**Q139.** What's horizontal scaling?

**Answer:** Adding instances instead of bigger ones

Elastic.

**Q140.** What does caching reduce?

**Answer:** Latency and backend load

Careful invalidation.

### [06 · Security Deep Dive (Hyper Protect, Key Management)](level-4/06-security-deep-dive.md)

**Q141.** What does defense in depth mean here?

**Answer:** Layers: IAM, network controls, encryption, monitoring and response

Overlap.

**Q142.** What does CBR add?

**Answer:** Network-context rules limiting where API calls may originate

Perimeter.

**Q143.** How detect threats?

**Answer:** Security services, logs and anomaly alerts into a SIEM

Visibility.

**Q144.** What's key rotation for?

**Answer:** Limit exposure from compromised keys

Lifecycle.

### [07 · Automation Pipelines at Scale (GitOps + Schematics)](level-4/07-automation-pipelines-at-scale.md)

**Q145.** What does pipeline automation at scale need?

**Answer:** Reusable templates, policy checks and secure credentials

Standardize.

**Q146.** What's a golden pipeline?

**Answer:** A standard pipeline teams reuse with built-in checks

Paved road.

**Q147.** Why gate on security scans?

**Answer:** Stop vulnerable artifacts from deploying

Shift left.

**Q148.** How handle approvals?

**Answer:** Automated checks plus human gates for prod

Balance.

### [08 · FinOps &amp; Enterprise Cost Optimization](level-4/08-finops-cost-optimization.md)

**Q149.** What's FinOps?

**Answer:** Cross-team practice to optimize cloud spend with visibility and accountability

Culture.

**Q150.** What lowers cost safely?

**Answer:** Right-sizing, reservations/savings plans and cleanup

Evidence.

**Q151.** What's showback vs chargeback?

**Answer:** Showing teams their costs vs billing them

Accountability.

**Q152.** What's a cost anomaly?

**Answer:** Unexpected spend spike needing investigation

Alerts.

### [09 · Site Reliability Engineering on IBM Cloud](level-4/09-sre-on-ibm-cloud.md)

**Q153.** What does SRE on IBM Cloud emphasize?

**Answer:** SLOs, error budgets, automation and blameless postmortems

Reliability.

**Q154.** What's an error budget?

**Answer:** The allowed unreliability that balances velocity and stability

Policy.

**Q155.** What makes a good postmortem?

**Answer:** Timeline, root cause and follow-up actions without blame

Learn.

**Q156.** What reduces toil?

**Answer:** Automating repetitive operational tasks

SRE goal.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** Landing zone, security, hybrid, automation, FinOps and SRE practices

Enterprise platform.

**Q158.** How prove readiness?

**Answer:** DR drills, security review, cost report and SLO dashboards

Evidence.

**Q159.** What documentation helps?

**Answer:** Architecture decisions, runbooks and ownership maps

Operate.

**Q160.** What sustains the platform?

**Answer:** Governance, automation and continuous improvement

Lifecycle.

## Mixed questions across all levels

**Q161.** A VSI in a private subnet needs to be managed from your laptop. What's the least-exposed approach?

**Answer:** A VPN/bastion (or Transit Gateway/private access) rather than a public IP on the instance

Minimize attack surface.

**Q162.** An application needs to write to Cloud Object Storage. What identity and role?

**Answer:** A service ID with a service access role Writer on that bucket/instance, using its API key from Secrets Manager

Least privilege.

**Q163.** What's the difference between platform and service access roles?

**Answer:** Platform roles manage the resource (create/configure); service roles control data/operations inside the service

Both are needed in different situations.

**Q164.** Which structure lets you survive the loss of one availability zone?

**Answer:** Subnets and instances spread across at least two zones behind a load balancer

Multi-zone VPC.

**Q165.** A VPC's subnets overlap with the on-prem network. What problem will you hit?

**Answer:** You can't route between them via Direct Link/VPN/Transit Gateway; plan non-overlapping CIDRs

Plan addressing early.

**Q166.** How many usable addresses does a /24 IBM Cloud VPC subnet have (5 reserved)?

```python
print(2**8-5)
```

**Answer:** 251

Five addresses are reserved per subnet. Verified by running it.

**Q167.** A security group and a network ACL both apply. Which is stateful?

**Answer:** The security group (the ACL is stateless)

Return traffic handling.

**Q168.** Where should database credentials for a Code Engine app come from?

**Answer:** Service credentials bound to the app or Secrets Manager, never in the image or Git

Rotate and scope.

**Q169.** A function must react to new objects in COS. Which model fits?

**Answer:** Event-driven: a trigger/notification invoking an action

Loose coupling.

**Q170.** Queue consumers sometimes get the same message twice. What must handlers be?

**Answer:** Idempotent

At-least-once delivery.

**Q171.** Traffic is spiky and mostly idle. Which compute option minimizes cost?

**Answer:** Serverless (Code Engine/Functions) scaling to zero

Pay for use.

**Q172.** A deployment drifted because someone edited a resource in the console. How do you prevent this?

**Answer:** Manage infrastructure through Schematics/Terraform and detect/fix drift

Infrastructure as code.

**Q173.** Terraform applies run concurrently and corrupt state. What helps?

**Answer:** Remote state with locking (Schematics workspaces serialize runs)

Safety.

**Q174.** Which encryption approach gives you control of the keys?

**Answer:** Bring your own key with Key Protect or HPCS (customer-managed keys)

Envelope encryption.

**Q175.** Which target defines how much data loss is acceptable after a disaster?

**Answer:** RPO (RTO is the recovery time)

Design replication to meet it.

**Q176.** An audit asks 'who deleted this bucket?'. Which service answers it?

**Answer:** Activity Tracker (audit events)

Who did what.

**Q177.** Monthly cost doubled. What do you check first?

**Answer:** Usage/billing reports by tag and resource, then idle or oversized resources

FinOps basics.

**Q178.** Which practice makes a landing zone repeatable and auditable?

**Answer:** Deploy it as code with reviewed pipelines

Deployable architectures.

**Q179.** What does an SRE error budget do?

**Answer:** Balances release velocity against reliability, pausing risky changes when the budget is spent

Policy tool.

## Practical coding questions

### Plan a multi-zone VPC

Implement plan_vpc(zones, tiers): carve a /18 address prefix per zone from 10.240.0.0/16 and allocate one /24 subnet per tier inside each zone's prefix, in order. Verify there are no overlaps and compute usable addresses per subnet (IBM Cloud reserves 5).

Starter code:

```
import ipaddress

def plan_vpc(zones, tiers, prefix_len=24, base="10.240.0.0/16"):
    # give each zone its own /18 address prefix carved from base; inside each zone's prefix allocate one
    # /prefix_len subnet per tier (in order). Return {zone: {tier: cidr}}; raise ValueError if
    # there are too many zones or tiers to fit.
    return {}

plan = plan_vpc(["us-south-1", "us-south-2", "us-south-3"], ["web", "app", "db"])
for z, tiers in plan.items():
    print(z, tiers)
nets = [ipaddress.ip_network(c) for t in plan.values() for c in t.values()]
print("subnets:", len(nets), "overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
print("usable per subnet:", nets[0].num_addresses - 5)
```

Expected output:

```text
us-south-1 {'web': '10.240.0.0/24', 'app': '10.240.1.0/24', 'db': '10.240.2.0/24'}
us-south-2 {'web': '10.240.64.0/24', 'app': '10.240.65.0/24', 'db': '10.240.66.0/24'}
us-south-3 {'web': '10.240.128.0/24', 'app': '10.240.129.0/24', 'db': '10.240.130.0/24'}
subnets: 9 overlaps: False
usable per subnet: 251
```

??? success "Show a solution"

    ```
    import ipaddress

    def plan_vpc(zones, tiers, prefix_len=24, base="10.240.0.0/16"):
        """One subnet per tier per zone, taken from per-zone address prefixes of the VPC (like IBM Cloud
        address prefixes), never overlapping. Returns {zone: {tier: cidr}}."""
        base_net = ipaddress.ip_network(base)
        zone_prefix = 18                                   # each zone owns a /18 address prefix
        zone_nets = list(base_net.subnets(new_prefix=zone_prefix))
        if len(zones) > len(zone_nets):
            raise ValueError("too many zones")
        plan = {}
        for zone, znet in zip(zones, zone_nets):
            subs = list(znet.subnets(new_prefix=prefix_len))
            if len(tiers) > len(subs):
                raise ValueError("too many tiers")
            plan[zone] = {tier: str(sub) for tier, sub in zip(tiers, subs)}
        return plan

    plan = plan_vpc(["us-south-1", "us-south-2", "us-south-3"], ["web", "app", "db"])
    for z, tiers in plan.items():
        print(z, tiers)
    nets = [ipaddress.ip_network(c) for t in plan.values() for c in t.values()]
    print("subnets:", len(nets), "overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
    print("usable per subnet:", nets[0].num_addresses - 5)
    ```

    Giving each zone its own address prefix and subnetting inside it keeps zones isolated and the plan non-overlapping. IBM Cloud reserves five addresses per subnet, so a /24 has 251 usable. Verified by running the solution.

### Evaluate IAM access policies

Implement allowed(user, kind, service, group, needed_role): subjects are the user's access groups (or the user/service ID itself); a policy matches by subject, kind (platform vs service role), service and resource group ("*" matches anything) and grants if its role's rank is at least the needed role's rank.

Starter code:

```
ROLE_RANK = {"Viewer": 1, "Reader": 1, "Operator": 2, "Writer": 2, "Editor": 3, "Manager": 3, "Administrator": 4}

POLICIES = [
    # (subject, kind, role, service, resource_group)  kind: platform manages the service, service accesses its data
    ("group:devs",   "service",  "Writer",  "cloud-object-storage", "dev"),
    ("group:devs",   "platform", "Editor",  "is",                   "dev"),
    ("group:ops",    "platform", "Administrator", "is",              "*"),
    ("group:audit",  "platform", "Viewer",  "*",                    "*"),
    ("serviceid:ci", "service",  "Writer",  "cloud-object-storage", "dev"),
]
MEMBERS = {"ada": ["group:devs"], "bo": ["group:ops"], "cy": ["group:audit"], "dan": []}

def allowed(user, kind, service, group, needed_role):
    # a user's subjects are MEMBERS[user] (their access groups) or [user] itself (e.g. a service ID).
    # allow if any POLICY matches subject, kind, service ("*" matches any) and resource group ("*" any)
    # with a role whose rank is >= the needed role's rank (platform and service roles are separate "kinds")
    return False

checks = [
    ("ada", "service", "cloud-object-storage", "dev", "Writer"),
    ("ada", "service", "cloud-object-storage", "prod", "Writer"),
    ("ada", "platform", "is", "dev", "Editor"),
    ("ada", "platform", "is", "dev", "Administrator"),
    ("bo", "platform", "is", "prod", "Operator"),
    ("cy", "platform", "cloud-object-storage", "prod", "Viewer"),
    ("cy", "service", "cloud-object-storage", "prod", "Reader"),
    ("dan", "platform", "is", "dev", "Viewer"),
    ("serviceid:ci", "service", "cloud-object-storage", "dev", "Reader"),
]
for c in checks:
    print(("ALLOW" if allowed(*c) else "DENY ").ljust(6), *c)
```

Expected output:

```text
ALLOW  ada service cloud-object-storage dev Writer
DENY   ada service cloud-object-storage prod Writer
ALLOW  ada platform is dev Editor
DENY   ada platform is dev Administrator
ALLOW  bo platform is prod Operator
ALLOW  cy platform cloud-object-storage prod Viewer
DENY   cy service cloud-object-storage prod Reader
DENY   dan platform is dev Viewer
ALLOW  serviceid:ci service cloud-object-storage dev Reader
```

??? success "Show a solution"

    ```
    ROLE_RANK = {"Viewer": 1, "Reader": 1, "Operator": 2, "Writer": 2, "Editor": 3, "Manager": 3, "Administrator": 4}

    POLICIES = [
        # (subject, kind, role, service, resource_group)  kind: platform manages the service, service accesses its data
        ("group:devs",   "service",  "Writer",  "cloud-object-storage", "dev"),
        ("group:devs",   "platform", "Editor",  "is",                   "dev"),
        ("group:ops",    "platform", "Administrator", "is",              "*"),
        ("group:audit",  "platform", "Viewer",  "*",                    "*"),
        ("serviceid:ci", "service",  "Writer",  "cloud-object-storage", "dev"),
    ]
    MEMBERS = {"ada": ["group:devs"], "bo": ["group:ops"], "cy": ["group:audit"], "dan": []}

    def allowed(user, kind, service, group, needed_role):
        subjects = MEMBERS.get(user, [user])
        for subject, k, role, svc, rg in POLICIES:
            if subject not in subjects or k != kind:
                continue
            if svc not in ("*", service) or rg not in ("*", group):
                continue
            if ROLE_RANK[role] >= ROLE_RANK[needed_role]:
                return True
        return False

    checks = [
        ("ada", "service", "cloud-object-storage", "dev", "Writer"),
        ("ada", "service", "cloud-object-storage", "prod", "Writer"),
        ("ada", "platform", "is", "dev", "Editor"),
        ("ada", "platform", "is", "dev", "Administrator"),
        ("bo", "platform", "is", "prod", "Operator"),
        ("cy", "platform", "cloud-object-storage", "prod", "Viewer"),
        ("cy", "service", "cloud-object-storage", "prod", "Reader"),
        ("dan", "platform", "is", "dev", "Viewer"),
        ("serviceid:ci", "service", "cloud-object-storage", "dev", "Reader"),
    ]
    for c in checks:
        print(("ALLOW" if allowed(*c) else "DENY ").ljust(6), *c)
    ```

    Policies attach to groups and service IDs, scope to a service and resource group, and platform roles are separate from service roles. A user with no group gets nothing by default (deny by default). This is a simplified model of IBM Cloud IAM. Verified by running the solution.

---

More free practice: the [IBM Cloud Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
