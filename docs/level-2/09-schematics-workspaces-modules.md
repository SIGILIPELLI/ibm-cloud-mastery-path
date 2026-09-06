# 09 · Schematics Workspaces & Modules

Level 1's Schematics module (Module 9) used one flat `main.tf` in one
workspace — fine for a VPC and a subnet, but it doesn't survive a second
environment. The moment you need a `dev` and a `prod` version of the same
VPC, or a database module reused by three different apps, a single file
stops working. This module restructures that into reusable **Terraform
modules** and separate **Schematics workspaces per environment**.

## Why one workspace isn't enough

A Schematics workspace holds exactly one Terraform state. If `dev` and
`prod` share a workspace, they share a state file — a `terraform destroy`
meant for `dev` can tear down `prod` right along with it, because
Terraform doesn't know they were ever supposed to be separate. The fix:
one workspace per environment, both pointed at the *same* reusable module
code but with different input variables.

## Anatomy of a reusable module

A module is just a directory with the same three files by convention:

```text
modules/
  vpc/
    main.tf        # resources
    variables.tf   # inputs
    outputs.tf      # values other configs can consume
```

```hcl
# modules/vpc/variables.tf
variable "name" {
  type = string
}
variable "resource_group_id" {
  type = string
}
variable "zones" {
  type    = list(string)
  default = ["us-south-1", "us-south-2"]
}
variable "subnet_size" {
  type    = number
  default = 128
}
```

```hcl
# modules/vpc/main.tf
resource "ibm_is_vpc" "this" {
  name                       = var.name
  resource_group             = var.resource_group_id
  address_prefix_management  = "manual"
}

resource "ibm_is_vpc_address_prefix" "zones" {
  for_each = toset(var.zones)
  name     = "prefix-${each.value}"
  vpc      = ibm_is_vpc.this.id
  zone     = each.value
  cidr     = "10.${index(var.zones, each.value) * 16}.0.0/20"
}

resource "ibm_is_subnet" "private" {
  for_each                 = toset(var.zones)
  name                      = "${var.name}-private-${each.value}"
  vpc                       = ibm_is_vpc.this.id
  zone                      = each.value
  total_ipv4_address_count  = var.subnet_size
  depends_on                = [ibm_is_vpc_address_prefix.zones]
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = ibm_is_vpc.this.id
}
output "private_subnet_ids" {
  value = { for z, s in ibm_is_subnet.private : z => s.id }
}
```

This is the same VPC design from Module 1, generalized so `name`, `zones`,
and `subnet_size` are parameters instead of hardcoded values.

## Compose modules per environment

```hcl
# environments/dev/main.tf
module "vpc" {
  source            = "../../modules/vpc"
  name              = "dev-app-vpc"
  resource_group_id = data.ibm_resource_group.group.id
  zones             = ["us-south-1", "us-south-2"]
  subnet_size       = 64
}

# environments/prod/main.tf
module "vpc" {
  source            = "../../modules/vpc"
  name              = "prod-app-vpc"
  resource_group_id = data.ibm_resource_group.group.id
  zones             = ["us-south-1", "us-south-2", "us-south-3"]
  subnet_size       = 256
}
```

`prod` gets a third zone and larger subnets by changing four lines, not by
copy-pasting and hand-editing a whole VPC definition — the entire point of
extracting the module in the first place.

## One Schematics workspace per environment

```bash
ibmcloud schematics workspace new \
  --name mastery-dev \
  --template-repo https://github.com/<you>/mastery-schematics-demo \
  --template-type terraform_v1.5 \
  --resource-group mastery-path

ibmcloud schematics workspace new \
  --name mastery-prod \
  --template-repo https://github.com/<you>/mastery-schematics-demo \
  --template-type terraform_v1.5 \
  --resource-group mastery-path
```

Both workspaces point at the same Git repo — the difference between them
is entirely which subdirectory (`environments/dev` vs
`environments/prod`) you select as the template's working directory when
creating the workspace, configurable from the workspace's settings in the
console or `workspace update`.

```bash
ibmcloud schematics plan --id <mastery-dev-workspace-id>
ibmcloud schematics apply --id <mastery-dev-workspace-id>
```

**Gotcha:** running `plan`/`apply` against the wrong workspace ID applies
the wrong environment's variables against whatever directory that
workspace is bound to — always double check `ibmcloud schematics
workspace get --id <id>` shows the environment you expect before applying,
especially once `dev` and `prod` workspace IDs both exist and look similar
in a terminal history.

## Passing output from one workspace into another

A common pattern: an app workspace needs a value (a subnet ID, a database
connection string) produced by an infrastructure workspace it doesn't own.
Schematics doesn't auto-wire this across workspaces — pull the output and
feed it in explicitly:

```bash
SUBNET_ID=$(ibmcloud schematics output --id <vpc-workspace-id> --output json \
  | jq -r '.[] | select(.name=="private_subnet_ids") | .value["us-south-1"]')

ibmcloud schematics workspace update --id <app-workspace-id> \
  --var "subnet_id=$SUBNET_ID"

ibmcloud schematics apply --id <app-workspace-id>
```

This is intentionally manual (or scripted in a pipeline, tying back to
Module 8) rather than automatic — it keeps the dependency between
workspaces explicit instead of hidden inside Terraform state that only
Schematics can see.

## Pinning module versions

Once a module is used by more than one environment, an in-place edit to it
changes every consumer's next `apply` simultaneously. Pin consumers to a
Git ref instead of a mutable local path once the module is stable enough
to share broadly:

```hcl
module "vpc" {
  source = "git::https://github.com/<you>/mastery-schematics-demo.git//modules/vpc?ref=v1.2.0"
  # ... same variables as before
}
```

Bumping `ref` is then a deliberate, reviewable change (a one-line diff)
rather than every environment silently picking up whatever the module
directory currently contains.

## How It Actually Works

- **A Schematics workspace is really just a hosted Terraform state file
  plus a managed job runner** — `plan`/`apply` don't run on your laptop,
  they run in an IBM-managed container that clones the bound Git
  directory, downloads the exact `terraform_v1.5`-pinned binary, runs it
  against the workspace's own state, and streams logs back. Two workspaces
  pointed at the same repo but different subdirectories share nothing at
  runtime except the source code — their state files, variable sets, and
  job history are completely independent, which is the entire safety
  property behind "one workspace per environment."
- **A `module` block doesn't get compiled or copied at plan time from a
  remote source every run** — for a local `../../modules/vpc` path
  Terraform reads it straight off disk on each `plan`, so an edit to the
  module is live for every environment's very next plan with no explicit
  update step. A `git::...?ref=` source is different: Terraform clones
  that specific ref into `.terraform/modules/` once and reuses the cached
  copy until `terraform init -upgrade` is run again, which is the actual
  mechanism that turns "pin a module version" from a convention into an
  enforced fact.
- **`for_each = toset(var.zones)` builds Terraform's resource graph with
  one graph node per zone value, keyed by that value, instead of by
  numeric index the way `count` would.** That keying is why reordering
  `zones` in a variable doesn't cause Terraform to plan a destroy/recreate
  of unrelated subnets the way a `count`-based list reorder would — each
  subnet's identity in state is tied to its zone string, not its position.
- **Cross-workspace output passing is manual because Schematics workspace
  state is opaque to other workspaces by design** — `ibmcloud schematics
  output` is reading a JSON blob out of one workspace's completed apply,
  and `workspace update --var` is writing a plain string into another
  workspace's variable set; there's no `remote_state` data source linking
  them automatically the way a single Terraform config's own module
  outputs would. That gap is deliberate: it forces the dependency between
  infrastructure and app workspaces to show up as an explicit, reviewable
  pipeline step (Module 8) rather than an implicit reach into another
  workspace's state.

## Cheat sheet

| Command / Syntax | Purpose |
|---|---|
| `module "name" { source = "../../modules/x" ... }` | Compose a reusable module into a root config |
| `for_each = toset(var.zones)` | Repeat a resource block once per item in a variable |
| `output "name" { value = ... }` | Expose a value for another config (or another workspace) to consume |
| `ibmcloud schematics workspace new --name <n> --template-repo <url>` | Create one workspace per environment |
| `ibmcloud schematics output --id <id> --output json` | Read a workspace's outputs to feed into another workspace |
| `ibmcloud schematics workspace update --id <id> --var "<k>=<v>"` | Feed a value into a different workspace |
| `source = "git::<url>//<path>?ref=<tag>"` | Pin a module to an immutable Git ref |

## Exercise

Extract the VPC design from Module 1 into `modules/vpc` with `name`,
`zones`, and `subnet_size` variables, then write `environments/dev` and
`environments/prod` root configs that each call it with different values.
Create `mastery-dev` and `mastery-prod` Schematics workspaces pointed at
the two environment directories, apply both, and confirm via
`ibmcloud is vpcs` that `dev-app-vpc` has two zones and `prod-app-vpc` has
three. Destroy both before moving on — this exercise is about the module
pattern, not resources you need to keep running.
