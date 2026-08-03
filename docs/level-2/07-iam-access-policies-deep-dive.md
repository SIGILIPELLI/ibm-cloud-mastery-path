# 07 · IAM Access Policies Deep Dive

Level 1's IAM module (Module 2) covered users, service IDs, access groups,
and resource-group-scoped policies — enough for a single developer working
alone. This module adds what a real team or automated pipeline needs:
custom roles narrower than the built-in ones, tag-based access that scales
without editing policies every time a resource is created, service-to-
service authorization so one IBM Cloud service can call another without a
human-managed API key, and context-based restrictions that lock access
down to specific networks regardless of what any policy grants.

## Custom roles: narrower than the built-in ones

The built-in service access roles (`Reader`, `Writer`, `Manager`, ...) are
often coarser than you want. A **custom role** picks individual actions
out of a service's full action list:

```bash
# See what actions a service exposes before building a role around them
ibmcloud iam service-roles --service-name cloud-object-storage --output json \
  | jq '.serviceRoles[].actions'

ibmcloud iam role-create cos-read-only-metadata \
  --service-name cloud-object-storage \
  --display-name "COS Read Metadata Only" \
  --actions "cloud-object-storage.object.get,cloud-object-storage.bucket.get_bucket_head"

ibmcloud iam access-group-policy-create mastery-path-developers \
  --roles "cos-read-only-metadata" \
  --service-name cloud-object-storage \
  --resource-group-name mastery-path
```

A custom role that grants only `object.get` and a HEAD-style metadata read
can list and inspect objects but can't download full object bodies or
write anything — useful for a monitoring or auditing identity that has no
business handling actual object content.

## Tag-based access: scale without touching policies per resource

Resource-group scoping (Level 1) is coarse; per-resource scoping (Module 6
of this level) doesn't scale past a handful of resources. **Access tags**
sit in between: tag resources by attribute, then write one policy against
the tag instead of one policy per resource.

```bash
# Tag resources as you create them (or after the fact)
ibmcloud resource tag-attach --tag-names "env:staging" \
  --resource-name mastery-postgres-ha

ibmcloud resource tag-attach --tag-names "env:staging" \
  --resource-name mastery-iks

# One policy grants access to everything tagged env:staging, present or future
ibmcloud iam access-group-policy-create mastery-path-developers \
  --roles Viewer \
  --service-name all-identity-services \
  --resource-tags "env:staging"
```

**Gotcha:** access tags are not the same as the cost/billing tags you
might already be attaching for reporting — both use `resource tag-attach`,
but only tags actually referenced in a policy's `--resource-tags` do
anything for access control. Attaching a tag to a resource grants nobody
anything by itself; the policy referencing that tag is what matters.

## Service-to-service authorization: no human-managed key required

When one IBM Cloud service needs to call another on your behalf — Code
Engine pulling from Container Registry, Kubernetes Service reading a
Secrets Manager certificate — the clean answer is an **authorization
policy** between the two services, not an API key stored as a Kubernetes
secret.

```bash
ibmcloud iam authorization-policy-create \
  codeengine \
  cloud-object-storage \
  Reader \
  --source-resource-group-name mastery-path \
  --target-resource-group-name mastery-path

ibmcloud iam authorization-policies
```

Once created, any Code Engine app or job in that resource group can read
from any COS bucket in the same resource group using its own platform
identity — no `PGPASSWORD`-style secret to create, rotate, or leak. This
is exactly the mechanism that lets IKS worker nodes pull images from
Container Registry in Module 3 without an explicit pull secret.

## Context-based restrictions: lock access to a network, not just an identity

A policy answers "can this identity do this action." A **context-based
restriction (CBR)** adds "...but only from this network" on top, and
applies even to identities with an otherwise valid `Administrator` policy.

```bash
# Define a network zone -- here, only the VPC from Module 1
ibmcloud cbr zone-create \
  --name mastery-path-vpc-zone \
  --description "Only ha-app-vpc" \
  --addresses type=vpc,value=$(ibmcloud is vpc ha-app-vpc --output json | jq -r .crn)

# Restrict Cloud Object Storage so its API is only reachable from that zone
ibmcloud cbr rule-create \
  --description "COS reachable only from ha-app-vpc" \
  --resource-attributes serviceName=cloud-object-storage,resourceGroupId=<mastery-path-rg-id> \
  --zone-id <zone-id-from-above> \
  --enforcement-mode enabled
```

**Gotcha:** `--enforcement-mode enabled` blocks non-conforming requests
immediately; test new rules with `--enforcement-mode report` first, which
logs what *would* have been blocked without actually blocking it — CBR
mistakes lock out legitimate traffic (including your own laptop, if it's
outside the zone you defined) just as effectively as they block attackers.

## Auditing effective access

```bash
# What can this identity actually do, combining IAM policy + CBR + tags?
ibmcloud iam user-policies you@example.com
ibmcloud iam access-group-policies mastery-path-developers
ibmcloud cbr rules

# Simulate whether a specific request would be allowed, without making it
ibmcloud iam access-report --output json
```

## Cheat sheet

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

## Exercise

Create a custom role on Cloud Object Storage that only permits reading
object metadata, and grant it to `mastery-path-developers`. Tag
`mastery-postgres-ha` and `mastery-iks` with `env:staging` and add one
policy granting `Viewer` across everything with that tag. Then create a
CBR zone scoped to `ha-app-vpc` and a rule restricting Cloud Object
Storage to it in `report` mode; check `ibmcloud cbr rules` and confirm the
rule exists before ever flipping it to `enabled` against production data.
