---
title: "Final Quiz"
description: "Test what you have learned across all four levels of IBM Cloud Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A VSI in a private subnet needs to be managed from your laptop. What's the least-exposed approach?",
    "options": [
      "Floating IP open to the world",
      "Open SSH to 0.0.0.0/0",
      "Share the key",
      "A VPN/bastion (or Transit Gateway/private access) rather than a public IP on the instance"
    ],
    "correctIndex": 3,
    "explanation": "Minimize attack surface."
  },
  {
    "question": "An application needs to write to Cloud Object Storage. What identity and role?",
    "options": [
      "Your personal API key",
      "Public bucket",
      "Administrator on the account",
      "A service ID with a service access role Writer on that bucket/instance, using its API key from Secrets Manager"
    ],
    "correctIndex": 3,
    "explanation": "Least privilege."
  },
  {
    "question": "What's the difference between platform and service access roles?",
    "options": [
      "Same",
      "Platform roles manage the resource (create/configure); service roles control data/operations inside the service",
      "None",
      "Reverse"
    ],
    "correctIndex": 1,
    "explanation": "Both are needed in different situations."
  },
  {
    "question": "Which structure lets you survive the loss of one availability zone?",
    "options": [
      "One zone",
      "Bigger VM",
      "One big subnet",
      "Subnets and instances spread across at least two zones behind a load balancer"
    ],
    "correctIndex": 3,
    "explanation": "Multi-zone VPC."
  },
  {
    "question": "A VPC's subnets overlap with the on-prem network. What problem will you hit?",
    "options": [
      "You can't route between them via Direct Link/VPN/Transit Gateway; plan non-overlapping CIDRs",
      "None",
      "Slow",
      "DNS"
    ],
    "correctIndex": 0,
    "explanation": "Plan addressing early."
  },
  {
    "question": "How many usable addresses does a /24 IBM Cloud VPC subnet have (5 reserved)?\n```python\nprint(2**8-5)\n```\n",
    "options": [
      "256",
      "251",
      "252",
      "254"
    ],
    "correctIndex": 1,
    "explanation": "Five addresses are reserved per subnet. Verified by running it."
  },
  {
    "question": "A security group and a network ACL both apply. Which is stateful?",
    "options": [
      "Both",
      "Neither",
      "ACL",
      "The security group (the ACL is stateless)"
    ],
    "correctIndex": 3,
    "explanation": "Return traffic handling."
  },
  {
    "question": "Where should database credentials for a Code Engine app come from?",
    "options": [
      "Service credentials bound to the app or Secrets Manager, never in the image or Git",
      "Git",
      "README",
      "Dockerfile"
    ],
    "correctIndex": 0,
    "explanation": "Rotate and scope."
  },
  {
    "question": "A function must react to new objects in COS. Which model fits?",
    "options": [
      "Polling cron",
      "VM",
      "Manual",
      "Event-driven: a trigger/notification invoking an action"
    ],
    "correctIndex": 3,
    "explanation": "Loose coupling."
  },
  {
    "question": "Queue consumers sometimes get the same message twice. What must handlers be?",
    "options": [
      "Idempotent",
      "Stateful",
      "Single",
      "Faster"
    ],
    "correctIndex": 0,
    "explanation": "At-least-once delivery."
  },
  {
    "question": "Traffic is spiky and mostly idle. Which compute option minimizes cost?",
    "options": [
      "Large always-on VSI",
      "Reserved cluster",
      "Bare metal",
      "Serverless (Code Engine/Functions) scaling to zero"
    ],
    "correctIndex": 3,
    "explanation": "Pay for use."
  },
  {
    "question": "A deployment drifted because someone edited a resource in the console. How do you prevent this?",
    "options": [
      "Ignore",
      "Ban the console",
      "Snapshots",
      "Manage infrastructure through Schematics/Terraform and detect/fix drift"
    ],
    "correctIndex": 3,
    "explanation": "Infrastructure as code."
  },
  {
    "question": "Terraform applies run concurrently and corrupt state. What helps?",
    "options": [
      "Git state",
      "Local state",
      "Remote state with locking (Schematics workspaces serialize runs)",
      "More engineers"
    ],
    "correctIndex": 2,
    "explanation": "Safety."
  },
  {
    "question": "Which encryption approach gives you control of the keys?",
    "options": [
      "Base64",
      "Bring your own key with Key Protect or HPCS (customer-managed keys)",
      "Default only",
      "None"
    ],
    "correctIndex": 1,
    "explanation": "Envelope encryption."
  },
  {
    "question": "Which target defines how much data loss is acceptable after a disaster?",
    "options": [
      "RTO",
      "SLA",
      "SLO",
      "RPO (RTO is the recovery time)"
    ],
    "correctIndex": 3,
    "explanation": "Design replication to meet it."
  },
  {
    "question": "An audit asks 'who deleted this bucket?'. Which service answers it?",
    "options": [
      "Billing",
      "Log Analysis",
      "Monitoring",
      "Activity Tracker (audit events)"
    ],
    "correctIndex": 3,
    "explanation": "Who did what."
  },
  {
    "question": "Monthly cost doubled. What do you check first?",
    "options": [
      "Usage/billing reports by tag and resource, then idle or oversized resources",
      "Add regions",
      "Buy more",
      "Add users"
    ],
    "correctIndex": 0,
    "explanation": "FinOps basics."
  },
  {
    "question": "Which practice makes a landing zone repeatable and auditable?",
    "options": [
      "Deploy it as code with reviewed pipelines",
      "Wiki",
      "Scripts on laptops",
      "Console clicks"
    ],
    "correctIndex": 0,
    "explanation": "Deployable architectures."
  },
  {
    "question": "What does an SRE error budget do?",
    "options": [
      "Sets prices",
      "Counts bugs",
      "Balances release velocity against reliability, pausing risky changes when the budget is spent",
      "Funds hardware"
    ],
    "correctIndex": 2,
    "explanation": "Policy tool."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Plan a multi-zone VPC

Implement plan_vpc(zones, tiers): carve a /18 address prefix per zone from 10.240.0.0/16 and allocate one /24 subnet per tier inside each zone's prefix, in order. Verify there are no overlaps and compute usable addresses per subnet (IBM Cloud reserves 5).

**Starter code**

```bash
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

**Expected output**

```text
us-south-1 {'web': '10.240.0.0/24', 'app': '10.240.1.0/24', 'db': '10.240.2.0/24'}
us-south-2 {'web': '10.240.64.0/24', 'app': '10.240.65.0/24', 'db': '10.240.66.0/24'}
us-south-3 {'web': '10.240.128.0/24', 'app': '10.240.129.0/24', 'db': '10.240.130.0/24'}
subnets: 9 overlaps: False
usable per subnet: 251
```

<details><summary>Show solution and explanation</summary>

<pre><code>import ipaddress

def plan_vpc(zones, tiers, prefix_len=24, base=&quot;10.240.0.0/16&quot;):
    &quot;&quot;&quot;One subnet per tier per zone, taken from per-zone address prefixes of the VPC (like IBM Cloud
    address prefixes), never overlapping. Returns {zone: {tier: cidr}}.&quot;&quot;&quot;
    base_net = ipaddress.ip_network(base)
    zone_prefix = 18                                   # each zone owns a /18 address prefix
    zone_nets = list(base_net.subnets(new_prefix=zone_prefix))
    if len(zones) &gt; len(zone_nets):
        raise ValueError(&quot;too many zones&quot;)
    plan = {}
    for zone, znet in zip(zones, zone_nets):
        subs = list(znet.subnets(new_prefix=prefix_len))
        if len(tiers) &gt; len(subs):
            raise ValueError(&quot;too many tiers&quot;)
        plan[zone] = {tier: str(sub) for tier, sub in zip(tiers, subs)}
    return plan

plan = plan_vpc([&quot;us-south-1&quot;, &quot;us-south-2&quot;, &quot;us-south-3&quot;], [&quot;web&quot;, &quot;app&quot;, &quot;db&quot;])
for z, tiers in plan.items():
    print(z, tiers)
nets = [ipaddress.ip_network(c) for t in plan.values() for c in t.values()]
print(&quot;subnets:&quot;, len(nets), &quot;overlaps:&quot;, any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
print(&quot;usable per subnet:&quot;, nets[0].num_addresses - 5)
</code></pre>

<p>Giving each zone its own address prefix and subnetting inside it keeps zones isolated and the plan non-overlapping. IBM Cloud reserves five addresses per subnet, so a /24 has 251 usable. Verified by running the solution.</p>
</details>

### Evaluate IAM access policies

Implement allowed(user, kind, service, group, needed_role): subjects are the user's access groups (or the user/service ID itself); a policy matches by subject, kind (platform vs service role), service and resource group ("*" matches anything) and grants if its role's rank is at least the needed role's rank.

**Starter code**

```bash
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

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>ROLE_RANK = {&quot;Viewer&quot;: 1, &quot;Reader&quot;: 1, &quot;Operator&quot;: 2, &quot;Writer&quot;: 2, &quot;Editor&quot;: 3, &quot;Manager&quot;: 3, &quot;Administrator&quot;: 4}

POLICIES = [
    # (subject, kind, role, service, resource_group)  kind: platform manages the service, service accesses its data
    (&quot;group:devs&quot;,   &quot;service&quot;,  &quot;Writer&quot;,  &quot;cloud-object-storage&quot;, &quot;dev&quot;),
    (&quot;group:devs&quot;,   &quot;platform&quot;, &quot;Editor&quot;,  &quot;is&quot;,                   &quot;dev&quot;),
    (&quot;group:ops&quot;,    &quot;platform&quot;, &quot;Administrator&quot;, &quot;is&quot;,              &quot;*&quot;),
    (&quot;group:audit&quot;,  &quot;platform&quot;, &quot;Viewer&quot;,  &quot;*&quot;,                    &quot;*&quot;),
    (&quot;serviceid:ci&quot;, &quot;service&quot;,  &quot;Writer&quot;,  &quot;cloud-object-storage&quot;, &quot;dev&quot;),
]
MEMBERS = {&quot;ada&quot;: [&quot;group:devs&quot;], &quot;bo&quot;: [&quot;group:ops&quot;], &quot;cy&quot;: [&quot;group:audit&quot;], &quot;dan&quot;: []}

def allowed(user, kind, service, group, needed_role):
    subjects = MEMBERS.get(user, [user])
    for subject, k, role, svc, rg in POLICIES:
        if subject not in subjects or k != kind:
            continue
        if svc not in (&quot;*&quot;, service) or rg not in (&quot;*&quot;, group):
            continue
        if ROLE_RANK[role] &gt;= ROLE_RANK[needed_role]:
            return True
    return False

checks = [
    (&quot;ada&quot;, &quot;service&quot;, &quot;cloud-object-storage&quot;, &quot;dev&quot;, &quot;Writer&quot;),
    (&quot;ada&quot;, &quot;service&quot;, &quot;cloud-object-storage&quot;, &quot;prod&quot;, &quot;Writer&quot;),
    (&quot;ada&quot;, &quot;platform&quot;, &quot;is&quot;, &quot;dev&quot;, &quot;Editor&quot;),
    (&quot;ada&quot;, &quot;platform&quot;, &quot;is&quot;, &quot;dev&quot;, &quot;Administrator&quot;),
    (&quot;bo&quot;, &quot;platform&quot;, &quot;is&quot;, &quot;prod&quot;, &quot;Operator&quot;),
    (&quot;cy&quot;, &quot;platform&quot;, &quot;cloud-object-storage&quot;, &quot;prod&quot;, &quot;Viewer&quot;),
    (&quot;cy&quot;, &quot;service&quot;, &quot;cloud-object-storage&quot;, &quot;prod&quot;, &quot;Reader&quot;),
    (&quot;dan&quot;, &quot;platform&quot;, &quot;is&quot;, &quot;dev&quot;, &quot;Viewer&quot;),
    (&quot;serviceid:ci&quot;, &quot;service&quot;, &quot;cloud-object-storage&quot;, &quot;dev&quot;, &quot;Reader&quot;),
]
for c in checks:
    print((&quot;ALLOW&quot; if allowed(*c) else &quot;DENY &quot;).ljust(6), *c)
</code></pre>

<p>Policies attach to groups and service IDs, scope to a service and resource group, and platform roles are separate from service roles. A user with no group gets nothing by default (deny by default). This is a simplified model of IBM Cloud IAM. Verified by running the solution.</p>
</details>
