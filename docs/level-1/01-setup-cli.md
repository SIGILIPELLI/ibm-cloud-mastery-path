# 01 · Setup & the `ibmcloud` CLI

Every lesson in this level assumes two things: you have an IBM Cloud account,
and you can drive it from a terminal with the `ibmcloud` CLI instead of
clicking through the console. This module gets both in place, plus the
habit — targeting a region and resource group before you provision
anything — that keeps a multi-project account from turning into a mess.

## Create an IBM Cloud account

1. Go to [cloud.ibm.com/registration](https://cloud.ibm.com/registration) and
   sign up with an email address.
2. You land on the **Lite** plan by default — a permanently free tier (not a
   time-limited trial) that gives you always-free allowances for services
   like Cloud Object Storage, Cloud Functions, and a small Kubernetes
   cluster, plus a $0 balance for pay-as-you-go services until you
   explicitly add a credit card.
3. Everything in this level is designed to fit inside Lite-tier allowances.
   Where a lesson provisions something that *can* incur cost (like a Virtual
   Server Instance), it calls that out explicitly and the capstone
   (Module 10) ends with a full teardown checklist.

!!! warning "Lite vs. Pay-As-You-Go"
    Upgrading your account unlocks paid services and higher quotas, but also
    means resources you forget to delete can generate a bill. Stay on Lite
    while working through this level unless a specific lesson tells you
    otherwise.

## Install the CLI

```bash
# macOS / Linux (installer script)
curl -fsSL https://clis.cloud.ibm.com/install/osx | sh      # macOS
curl -fsSL https://clis.cloud.ibm.com/install/linux | sh    # Linux

# macOS (Homebrew)
brew install --cask ibm-cloud-cli

# Windows (PowerShell, run as Administrator)
iex(New-Object Net.WebClient).DownloadString('https://clis.cloud.ibm.com/install/powershell')
```

Verify it installed:

```bash
ibmcloud --version
# ibmcloud version 2.x.x+...
```

## Install the plugins you'll need this level

The base CLI is intentionally small; each service family ships as a plugin.

```bash
ibmcloud plugin install vpc-infrastructure
ibmcloud plugin install cloud-object-storage
ibmcloud plugin install cloud-databases
ibmcloud plugin install cloud-functions
ibmcloud plugin install schematics
ibmcloud plugin install observe-service

ibmcloud plugin list
```

## Log in

```bash
# Interactive email/password login (prompts for a one-time passcode if
# multi-factor auth is enabled on your account)
ibmcloud login

# Non-interactive login with an API key -- the pattern you'll use in scripts
# and CI, and the one used for the rest of this site
ibmcloud login --apikey @~/.ibmcloud/apikey.json -r us-south

# Single sign-on (federated accounts)
ibmcloud login --sso
```

Create an API key once, up front, so every later lesson can log in
non-interactively:

```bash
ibmcloud iam api-key-create my-cli-key \
  --description "Local CLI access" \
  --file ~/.ibmcloud/apikey.json

chmod 600 ~/.ibmcloud/apikey.json
```

!!! danger "Treat the API key file like a password"
    Anyone with this file can act as you on the account. Never commit it to
    git — add `~/.ibmcloud/` to your global gitignore, not the project's.

## Target a region and resource group

IBM Cloud organizes billable resources under **resource groups** (a
lightweight folder for access control and cost tracking) inside a
**region** (where the resource physically runs). Almost every provisioning
command needs both set, or passed explicitly.

```bash
# List what's available
ibmcloud regions
ibmcloud resource groups

# Set both for the current CLI session
ibmcloud target -r us-south -g Default

# Confirm
ibmcloud target
```

```text
API endpoint:      https://cloud.ibm.com
Region:            us-south
User:              you@example.com
Account:           My Account (abcd1234...) <-> (unbound)
Resource group:    Default
```

Create a dedicated resource group for this course so everything you build is
easy to find (and easy to tear down) later:

```bash
ibmcloud resource group-create mastery-path
ibmcloud target -g mastery-path
```

## How It Actually Works

- **`ibmcloud login` doesn't send your password on every later command — it
  exchanges credentials once for a short-lived IAM access token (an hour or
  so) plus a longer-lived refresh token, both cached in
  `~/.bluemix/config.json`.** Every subsequent CLI call reads that cached
  access token and attaches it as an `Authorization: Bearer` header to the
  IBM Cloud IAM-fronted API it's hitting; when the access token expires the
  CLI silently uses the refresh token to mint a new one, which is why a
  session that's been idle for days sometimes forces a fresh `login` — the
  refresh token itself has finally expired too.
- **An API key is a static, non-expiring credential in a different tier
  from that ephemeral access token — it's what you trade in for a new
  token, not the token itself.** `ibmcloud login --apikey @file` reads the
  key from the JSON file, POSTs it to IAM's token endpoint
  (`iam.cloud.ibm.com/identity/token`), and receives back the same
  short-lived access token an interactive login would get. This is exactly
  why the key file, not the token cache, is the thing worth protecting:
  anyone holding it can regenerate valid tokens indefinitely, while a
  leaked access token self-expires within the hour.
- **Region and resource-group targeting are local CLI state, not account
  configuration — `ibmcloud target` writes your choice into
  `~/.bluemix/config.json` and every later command reads it from there to
  fill in the region/resource-group fields a REST call requires.** Nothing
  about the account itself changes; a second terminal, or the console UI,
  has no idea what you targeted. That's also why CI pipelines re-target
  explicitly on every run instead of relying on a previous session's
  state — there's no shared, server-side "current context" to inherit.
- **Each `ibmcloud plugin install` pulls a self-contained Go binary for
  that service family and registers its subcommands with the core CLI's
  command router — the base CLI ships thin on purpose so it stays small and
  installs fast, deferring the actual API client code for VPC, COS,
  Cloud Databases, etc. until you ask for it.**

## Cheat sheet

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

## Exercise

Create an IBM Cloud account if you don't have one, install the CLI and the
six plugins above, generate an API key and save it to
`~/.ibmcloud/apikey.json`, then create a `mastery-path` resource group and
target it in region `us-south` (or the region closest to you). Run
`ibmcloud target` and confirm the output shows your API key login, chosen
region, and the `mastery-path` group — you'll reuse this exact context for
every remaining Level 1 module.
