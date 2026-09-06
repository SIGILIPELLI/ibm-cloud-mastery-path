# 02 · Landing Zone Architecture

Every network, IAM, and governance pattern from earlier modules —
multi-account structure, Transit Gateway hubs, SCC profiles, budgets — is
what IBM calls a **landing zone** when assembled as one deployable,
opinionated starting point for every new workload account. This module
builds a minimal landing zone using IBM's reference architecture and the
`terraform-ibm-modules` landing zone module.

## What a landing zone standardizes

| Layer | Standardized by |
|---|---|
| Account structure | Enterprise account groups (Module 01) |
| Network | Hub VPC + Transit Gateway + spoke VPCs (Level 3, Module 02) |
| Identity | Trusted profiles, dynamic access groups (Module 01) |
| Security posture | SCC profile attachment, Key Protect (Level 3, Module 03) |
| Observability | Monitoring/logging auto-attached to every new cluster |
| Cost | Budgets and mandatory cost tags (Level 3, Module 07) |

The point isn't inventing anything new — it's making every one of those
already-covered patterns apply automatically to a *new* account instead
of being re-decided (and re-forgotten) each time one is created.

## Use the IBM landing zone Terraform module

```hcl
module "landing_zone" {
  source  = "terraform-ibm-modules/landing-zone/ibm"
  version = "~> 5.0"

  prefix         = "acme"
  region         = "us-south"
  vpcs           = ["management", "workload"]
  enable_transit_gateway = true

  network_cidr   = "10.0.0.0/8"
  vpc_cidr_map = {
    management = "10.10.0.0/16"
    workload   = "10.20.0.0/16"
  }
}
```

```bash
terraform init
terraform validate
```

```text
Success! The configuration is valid.
```

This one module composes the hub-and-spoke VPC pattern, subnets tiered
per zone, and a Transit Gateway connecting `management` (bastion, CI
runners, shared services) to `workload` (the actual application VPC) —
the same shape built by hand across Level 2 and Level 3, now expressed as
one reusable, versioned module.

## Add the management VPC's shared services

```hcl
module "bastion" {
  source     = "./modules/bastion-host"
  vpc_id     = module.landing_zone.vpc_data["management"].vpc_id
  subnet_id  = module.landing_zone.vpc_data["management"].subnet_zone_list[0].id
}

module "shared_registry" {
  source            = "./modules/container-registry-namespace"
  resource_group_id = module.landing_zone.resource_group_id
  namespace         = "acme-shared"
}
```

## Bootstrap script: from zero to a compliant account

```bash
#!/usr/bin/env bash
set -euo pipefail

ACCOUNT_NAME="$1"

ibmcloud enterprise account-create --name "$ACCOUNT_NAME" \
  --parent "$(ibmcloud enterprise account-group product-teams --output json | jq -r .crn)" \
  --owner-email "team-lead@example.com"

echo "Waiting for account activation..."
until ibmcloud enterprise account "$ACCOUNT_NAME" --output json 2>/dev/null | jq -e '.state == "ACTIVE"' >/dev/null; do
  sleep 30
done

terraform -chdir=landing-zone init
terraform -chdir=landing-zone apply -var="account_name=$ACCOUNT_NAME" -auto-approve

ibmcloud scc profile-attach --profile "CIS IBM Cloud Foundations Benchmark" \
  --scope-id "$(ibmcloud account show --output json | jq -r .account_id)"
```

A script like this is what turns "provision a compliant new workload
account" from a multi-day, error-prone checklist into a single command —
the actual business value of a landing zone.

## Landing zone variants: IBM's reference patterns

IBM publishes a few named variants worth knowing rather than inventing
from scratch:

- **VSI (Classic)** — landing zone built on classic infrastructure VSIs,
  for teams migrating from classic rather than starting on VPC.
- **VPC** — the pattern this module builds: pure VPC Gen2 infrastructure.
- **ROKS** — VPC landing zone plus a pre-wired OpenShift cluster,
  monitoring, and logging attached from creation.
- **VSI Quickstart** — a smaller single-VPC variant for a fast proof of
  concept, intentionally skipping the hub/spoke topology.

## Guardrails as code, not tribal knowledge

```hcl
resource "ibm_iam_account_settings" "guardrails" {
  mfa                              = "TOTP4ALL"
  restrict_create_platform_apikey  = "RESTRICTED"
}

resource "ibm_resource_tag" "mandatory_cost_tag" {
  resource_id = module.landing_zone.workload_vpc_crn
  tags        = ["cost-center:unassigned"]
}
```

`cost-center:unassigned` as a *default* tag is deliberate: it makes an
untagged resource visible in a billing report (searchable, flaggable) 
instead of invisible, which a missing tag would otherwise be.

## Gotchas

- **A landing zone module version pin matters more than most Terraform
  dependencies** — `terraform-ibm-modules` releases are versioned and can
  introduce breaking variable renames; pin a specific minor version and
  upgrade deliberately, never track `latest`.
- **The module provisions infrastructure, not organizational buy-in** — a
  landing zone that mandates SCC and tagging but that teams route around
  (creating resources outside the module) doesn't achieve the
  standardization goal; pair the module with a policy that blocks
  non-compliant resource creation.
- **Bootstrap scripts that `apply -auto-approve` need to run somewhere
  auditable** (a CI pipeline, Schematics — see Level 3 Module 08) — running
  them from a laptop defeats the audit-trail benefit of having a landing
  zone in the first place.
- **Account activation polling loops need a timeout** — an owner who never
  accepts the invitation email leaves a naive `until ... ACTIVE` loop
  running forever; add a max-attempts guard in real automation.

## How It Actually Works

- **The landing zone module isn't a new IBM Cloud service — it's a
  composition layer that internally calls the same `ibm_is_vpc`,
  `ibm_tg_gateway`, and related resources you wrote by hand in Level 2 and
  Level 3, wired together with sensible defaults and cross-references.**
  `terraform plan` against it produces exactly the resource graph those
  earlier modules built manually (VPCs, subnets, address prefixes, a
  Transit Gateway, connections) — the module's value is that those
  cross-resource references (subnet IDs into worker pools, VPC CRNs into
  Transit Gateway connections) are already correctly wired and tested,
  not that it does anything the underlying provider couldn't.
- **A version pin on a shared module works the same mechanism as any
  Terraform module source pin (Level 3, Module 08): `terraform init`
  resolves `~> 5.0` once against the module registry and caches the
  fetched version, only re-resolving on `init -upgrade`.** That's why an
  unpinned or loosely pinned landing zone module is riskier than an
  ordinary app dependency — every account bootstrapped through it inherits
  whatever breaking variable rename or default change shipped in a newer
  minor version the next time someone runs `init -upgrade` on the shared
  bootstrap repo, not just the one account being built.
- **`cost-center:unassigned` as a default tag exploits the same tag-based
  billing attribution mechanism from Level 3, Module 07 — a resource
  without an explicit cost-center tag still emits metering records, and
  those records need *some* tag value to be queryable at all.** Applying
  the placeholder at creation time (via the Terraform resource, not a
  later manual step) guarantees every metering record from that resource
  carries a non-null, searchable value, turning "forgot to tag" into a
  visible, queryable billing-report line instead of spend that simply
  doesn't show up when filtering by tag.
- **The account-activation polling loop works because
  `ibmcloud enterprise account-create` only creates a *pending* invitation
  record — the account itself doesn't exist as a usable IAM/billing
  entity until the invited owner accepts it**, which is a human action
  Terraform and the CLI have no way to trigger or wait on synchronously.
  That's why the bootstrap script polls state rather than chaining
  `apply` directly after `account-create`: there's a genuine external
  dependency (an email being read and a link clicked) sitting between the
  two API calls.

## Cheat sheet

| Task | Command / reference |
|---|---|
| Landing zone module | `terraform-ibm-modules/landing-zone/ibm` |
| Create enterprise account | `ibmcloud enterprise account-create --name <n> --parent <crn>` |
| Check account activation state | `ibmcloud enterprise account <n> --output json \| jq .state` |
| Attach SCC profile post-bootstrap | `ibmcloud scc profile-attach --profile <name> --scope-id <account>` |
| Validate landing zone Terraform | `terraform validate` |

## Exercise

1. Write a landing zone Terraform configuration using the
   `terraform-ibm-modules/landing-zone/ibm` module with a management and
   workload VPC, and run `terraform validate`.
2. Add a bastion host module and a shared Container Registry namespace to
   the management VPC.
3. Write a bootstrap script that creates an enterprise account, waits for
   activation, applies the landing zone, and attaches an SCC profile —
   include a timeout on the activation-wait loop.
4. List, in your own words, which three guardrails from this module you'd
   consider non-negotiable for any new workload account, and why.
