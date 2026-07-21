# 09 · Schematics (Terraform-based IaC)

Every resource in Modules 3-8 was created with one-off CLI commands.
**Schematics** is IBM Cloud's managed Terraform service: you describe
infrastructure declaratively, and Schematics plans and applies it for you
(no local Terraform install or state file management required, though you
can also run the same templates with your own Terraform CLI). This module
introduces the workflow that Level 2 builds on heavily.

## Why declarative IaC over CLI one-liners

CLI commands like the ones in earlier modules are easy to run once, but
have no memory of what you already created — re-running them either fails
(name already exists) or duplicates resources, and there's no single file
describing "this is what our infrastructure should look like." A Terraform
configuration fixes that: it's a versionable description of desired state,
and Terraform (via Schematics or locally) computes and applies the diff.

## A minimal configuration

```hcl
# main.tf
terraform {
  required_providers {
    ibm = {
      source  = "IBM-Cloud/ibm"
      version = "~> 1.60"
    }
  }
}

variable "ibmcloud_api_key" {
  type      = string
  sensitive = true
}

provider "ibm" {
  ibmcloud_api_key = var.ibmcloud_api_key
  region           = "us-south"
}

data "ibm_resource_group" "group" {
  name = "mastery-path"
}

resource "ibm_is_vpc" "vpc" {
  name           = "schematics-vpc"
  resource_group = data.ibm_resource_group.group.id
}

resource "ibm_is_subnet" "subnet" {
  name                     = "schematics-subnet"
  vpc                      = ibm_is_vpc.vpc.id
  zone                     = "us-south-1"
  total_ipv4_address_count = 256
  resource_group           = data.ibm_resource_group.group.id
}

output "vpc_id" {
  value = ibm_is_vpc.vpc.id
}
```

Push this to a Git repo (Schematics pulls templates from a repo URL, not
local files) — a public repo is fine for this exercise since it contains no
secrets (the API key comes in as a variable, never hardcoded).

## Create a Schematics workspace

```bash
ibmcloud schematics workspace new \
  --name mastery-schematics \
  --template-repo https://github.com/<you>/mastery-schematics-demo \
  --template-type terraform_v1.5 \
  --resource-group mastery-path
```

Set the sensitive `ibmcloud_api_key` variable **through the workspace**, not
in the repo:

```bash
ibmcloud schematics workspace update --id <workspace-id> \
  --var "ibmcloud_api_key=$(cat ~/.ibmcloud/apikey.json | jq -r .apikey)" \
  --var-type sensitive
```

## Plan and apply

```bash
# Generates an execution plan without changing anything
ibmcloud schematics plan --id <workspace-id>

ibmcloud schematics apply --id <workspace-id>

# Watch it run
ibmcloud schematics logs --id <workspace-id> --latest
```

## Inspecting and destroying

```bash
# See what Terraform believes exists right now
ibmcloud schematics state pull --id <workspace-id>

ibmcloud schematics output --id <workspace-id>

# Tear everything the workspace created back down
ibmcloud schematics destroy --id <workspace-id>
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud schematics workspace new --name <n> --template-repo <url> --template-type <t>` | Create a workspace from a Git-hosted Terraform template |
| `ibmcloud schematics workspace update --id <id> --var "<k>=<v>" --var-type sensitive` | Set a sensitive input variable |
| `ibmcloud schematics plan --id <id>` | Preview changes |
| `ibmcloud schematics apply --id <id>` | Apply changes |
| `ibmcloud schematics logs --id <id> --latest` | Stream the latest job's logs |
| `ibmcloud schematics output --id <id>` | Show Terraform outputs |
| `ibmcloud schematics destroy --id <id>` | Tear down everything the workspace manages |

## Exercise

Push the `main.tf` above to a Git repo of your own, create a Schematics
workspace pointing at it, set `ibmcloud_api_key` as a sensitive variable,
and run `plan` then `apply`. Confirm the VPC and subnet appear via
`ibmcloud is vpc-list`, then run `ibmcloud schematics destroy` and confirm
they're gone — the same plan/apply/destroy loop you'll use for every
project in Level 2 and beyond.
