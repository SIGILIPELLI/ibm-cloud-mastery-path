# 08 · Advanced Schematics & Terraform Modules

Every prior module ran `terraform validate` locally against snippets.
Real teams don't run `terraform apply` from a laptop against shared
infrastructure — state gets lost, credentials get scattered, and nobody
can see who ran what. This module covers **IBM Cloud Schematics**
(managed Terraform-as-a-service) and structuring Terraform into reusable
modules.

## Why Schematics instead of local Terraform

| | Local Terraform | Schematics |
|---|---|---|
| State storage | Wherever you configure a backend | Managed, versioned automatically |
| Credentials | Local env vars / files | IAM-scoped workspace, no local secrets |
| Who ran what | Shell history, if you're lucky | Full plan/apply audit log |
| Drift detection | Manual `terraform plan` | Built-in drift detection jobs |
| Team access | Shared backend + shared secrets | IAM roles on the workspace itself |

## Create a Schematics workspace

```bash
ibmcloud schematics workspace new \
  --file workspace.json
```

```json
{
  "name": "orders-platform-prod",
  "type": ["terraform_v1.7"],
  "resource_group": "mastery-path",
  "template_repo": {
    "url": "https://github.com/example-org/orders-platform-infra",
    "branch": "main"
  },
  "template_data": [{
    "folder": "environments/prod",
    "variablestore": [
      {"name": "region", "value": "us-south"},
      {"name": "worker_count", "value": "3"}
    ]
  }]
}
```

```text
Workspace orders-platform-prod created.
ID: us-south.workspace.orders-platform-prod.a1b2c3d4
```

## Plan and apply through Schematics, not locally

```bash
ibmcloud schematics plan --id us-south.workspace.orders-platform-prod.a1b2c3d4
```

```text
Activity ID: 9f8e7d...
Plan job submitted. Check status with 'ibmcloud schematics logs'.
```

```bash
ibmcloud schematics logs --id us-south.workspace.orders-platform-prod.a1b2c3d4 --act-id 9f8e7d...
```

```text
Terraform will perform the following actions:
  + ibm_is_vpc.ha_app_vpc
  + ibm_container_vpc_cluster.roks
Plan: 12 to add, 0 to change, 0 to destroy.
```

```bash
ibmcloud schematics apply --id us-south.workspace.orders-platform-prod.a1b2c3d4
```

Applying requires explicit confirmation in interactive mode, or
`--force` for automation pipelines — a deliberate friction point so a
`prod` workspace apply is never accidental.

## Structure Terraform into modules, not one giant file

```text
infra/
  modules/
    ha-vpc/
      main.tf
      variables.tf
      outputs.tf
    roks-cluster/
      main.tf
      variables.tf
      outputs.tf
  environments/
    prod/
      main.tf        # composes modules with prod-specific inputs
      backend.tf
    dev/
      main.tf        # same modules, smaller sizing
```

```hcl
# environments/prod/main.tf
module "vpc" {
  source         = "../../modules/ha-vpc"
  region         = "us-south"
  address_prefix = "10.10.0.0/16"
}

module "cluster" {
  source       = "../../modules/roks-cluster"
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids
  worker_count = 3
  flavor       = "bx2.4x16"
}
```

```hcl
# modules/roks-cluster/variables.tf
variable "vpc_id"     { type = string }
variable "subnet_ids" { type = list(string) }

variable "worker_count" {
  type    = number
  default = 2
}

variable "flavor" {
  type    = string
  default = "bx2.4x16"
}
```

```bash
terraform -chdir=infra/environments/prod validate
# Success! The configuration is valid.
```

One module per logical unit (VPC, cluster, database) reused across
`dev`/`prod` environment folders keeps drift between environments
intentional (different variable values) instead of accidental (copy-pasted
files that silently diverge).

## Drift detection

```bash
ibmcloud schematics workspace refresh-state --id us-south.workspace.orders-platform-prod.a1b2c3d4
ibmcloud schematics plan --id us-south.workspace.orders-platform-prod.a1b2c3d4
```

A plan showing unexpected changes against a workspace nobody has touched
means something was modified outside Terraform — usually a console
click-fix during an incident that never got backported into code. Treat
every such drift as a required follow-up: either codify the manual change
or revert it, don't leave it silently diverged.

## Policy-as-code gate before apply

Schematics supports a pre-apply policy check via IBM Cloud's Compliance
policies or a custom Sentinel-style check wired through a CI pipeline
(covered fully in Level 4's automation module) — worth previewing here:

```yaml
# .schematics/policy.yaml (conceptual — enforced via CI, not a native Schematics field)
rules:
  - name: no-public-database
    deny_if: "ibm_database.*.access_tags does not include 'private'"
  - name: require-cost-tags
    deny_if: "resource lacks tag 'cost-center'"
```

Running a policy check as a required CI step before `ibmcloud schematics
apply` (rather than trusting reviewers to catch every violation by eye) is
the practice Level 4's pipelines module builds out fully.

## Gotchas

- **Schematics workspace Terraform version is pinned per workspace** — a
  module written against a newer Terraform syntax feature can fail in a
  workspace still pinned to an older `terraform_v1.x`; check
  `ibmcloud schematics workspace get` before assuming a new HCL feature
  will work.
- **`terraform_v1.7` (or whichever is current) workspace types get
  deprecated on a schedule** — Schematics announces sunset dates for old
  Terraform versions; an old workspace left unmigrated eventually can't be
  applied at all.
- **Module source pinning matters**: `source = "../../modules/x"` (local
  path) works fine within one repo, but a module pulled from a separate
  Git repo should pin a tag or commit SHA (`?ref=v1.2.0`), not a floating
  branch, or a workspace's next apply can silently pick up unreviewed
  module changes.
- **State file secrets**: Terraform state can contain sensitive values
  (e.g., generated database passwords) in plaintext within the state JSON
  — Schematics encrypts state at rest, but exporting state locally for
  debugging reintroduces that exposure.

## How It Actually Works

- **A Schematics workspace runs `plan`/`apply` inside a job container IBM
  provisions per activity, using a token minted for the workspace's own
  IAM identity — never your local credentials.** That's why "no local
  secrets" is a real property, not just a policy: the job authenticates as
  a service identity scoped by the workspace's IAM policies, does its
  clone-init-plan/apply cycle, streams logs back, and the container is
  discarded afterward — nothing about your laptop's credentials or
  environment variables is ever in the loop.
- **Drift detection is just `terraform refresh` followed by `terraform
  plan`, run by Schematics on your behalf rather than a distinct
  feature.** `refresh-state` re-reads the real, current configuration of
  every resource in state directly from each service's API and updates
  Terraform's recorded view of "what exists" without changing actual
  infrastructure; the following `plan` then diffs that refreshed state
  against your HCL. A plan showing unexpected changes after a refresh
  means the live resource's API-reported state no longer matches what
  your code says it should be — which is precisely what a console
  click-fix produces.
- **Module source pinning changes what `terraform init` does at the file
  level, which is the entire mechanism behind the risk described above.**
  A local path source is read live off disk on every init; a Git source
  with `?ref=<tag-or-sha>` is fetched once into `.terraform/modules/` and
  cached — `init` only re-fetches it if the ref itself changes or
  `-upgrade` is passed. A floating branch ref re-resolves to whatever
  commit is currently at the branch tip on the next `init`, so two applies
  minutes apart can silently use different module code with an identical
  Terraform config.
- **State encryption at rest protects the stored JSON file, but every
  value a provider marks sensitive is still fully present in plaintext
  inside that JSON — encryption controls who can read the file, not what's
  in it.** Schematics encrypts the workspace's persisted state using a
  managed key, which is why the risk described only appears once you
  `terraform state pull`/export it locally: at that point the decrypted
  JSON, generated passwords included, exists as plaintext on whatever
  machine you exported it to.

## Cheat sheet

| Task | Command |
|---|---|
| Create workspace | `ibmcloud schematics workspace new --file <config.json>` |
| Plan | `ibmcloud schematics plan --id <workspace-id>` |
| Apply | `ibmcloud schematics apply --id <workspace-id>` |
| View logs | `ibmcloud schematics logs --id <workspace-id> --act-id <activity-id>` |
| Refresh state (drift check) | `ibmcloud schematics workspace refresh-state --id <workspace-id>` |
| Destroy | `ibmcloud schematics destroy --id <workspace-id>` |

## Exercise

1. Refactor a VPC + cluster Terraform config from earlier modules into
   two reusable modules under `modules/`, composed from a `prod`
   environment folder.
2. Create a Schematics workspace configuration (JSON) pointing at that
   repo layout and validate the JSON with `python3 -m json.tool`.
3. Write out the `terraform plan` output you'd expect to see on first
   apply (resource counts by type) and compare against an actual
   `terraform plan` run locally against the same modules.
4. Describe, in prose, the drift-detection workflow you'd run monthly
   against a production workspace and what you'd do if drift is found.
