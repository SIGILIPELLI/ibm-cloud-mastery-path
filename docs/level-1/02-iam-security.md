# 02 · IAM & Security Basics

Identity and Access Management (IAM) is the layer that decides *who* (a
person or a piece of software) can do *what* to *which* resource on your
account. Get comfortable with it before Module 3 starts creating billable
infrastructure — it's much easier to design access correctly up front than
to retrofit it after ten resources exist.

## The four building blocks

| Concept | What it is |
|---|---|
| **User** | A person who logs in with an email/password or SSO identity. |
| **Service ID** | A non-human identity for an app, script, or CI pipeline — cannot log in interactively, only authenticates via API key. |
| **Access group** | A named bundle of policies you assign to multiple users/service IDs at once, instead of repeating policies per identity. |
| **Policy** | The actual grant: *this identity* gets *this role* on *this resource (or resource group, or whole account)*. |

## Roles

IBM Cloud has two role families that stack:

- **Platform roles** — control management-plane actions (create/delete/view
  the resource itself, manage its access policies): `Viewer`, `Operator`,
  `Editor`, `Administrator`.
- **Service access roles** — control what you can do *inside* the service's
  own data plane, and vary a bit by service: commonly `Reader`, `Writer`,
  `Manager` (e.g. for Cloud Object Storage, reading vs. writing objects is a
  service access role, not a platform role).

A policy typically grants both, e.g. "Editor + Writer on all Cloud Object
Storage instances in resource group `mastery-path`".

## Creating and using an access group

```bash
# Create the group
ibmcloud iam access-group-create mastery-path-developers

# Grant it Editor on every resource inside your course resource group
ibmcloud iam access-group-policy-create mastery-path-developers \
  --roles Editor \
  --resource-group-name mastery-path

# Add a user to the group
ibmcloud iam access-group-user-add mastery-path-developers you@example.com

# Review what a group can actually do
ibmcloud iam access-group-policies mastery-path-developers
```

## Service IDs and API keys (for apps, not people)

Every automated thing you build later this level — a Cloud Functions action,
a CI pipeline, a Schematics job — should authenticate as a **service ID**
scoped to only what it needs, never as your personal user.

```bash
# Create a service ID
ibmcloud iam service-id-create mastery-path-app \
  --description "Identity for the capstone app's backend"

# Grant it a narrow, resource-group-scoped policy (least privilege --
# NOT "Administrator" on the whole account)
ibmcloud iam service-policy-create mastery-path-app \
  --roles Writer \
  --service-name cloud-object-storage \
  --resource-group-name mastery-path

# Issue an API key for that service ID
ibmcloud iam service-api-key-create app-key mastery-path-app \
  --file ~/.ibmcloud/app-apikey.json
```

## Listing and auditing access

```bash
# Every policy on the account, across users, service IDs, and groups
ibmcloud iam access-group-policies mastery-path-developers
ibmcloud iam service-policies mastery-path-app
ibmcloud iam user-policies you@example.com

# Every API key that exists for a service ID (rotate/revoke old ones)
ibmcloud iam service-api-keys mastery-path-app
ibmcloud iam service-api-key-delete mastery-path-app <key-name>
```

## Least privilege in practice

- Scope policies to a **resource group** (`--resource-group-name`) or even a
  single resource, not the whole account, whenever you can.
- Give service IDs the narrowest service access role that works —
  `Reader`/`Writer` instead of `Manager`, `Writer` instead of `Editor`, and
  never `Administrator` for an automated identity.
- Use access groups instead of per-user policies once you have more than a
  couple of people — one policy change on the group updates everyone.
- Rotate and delete API keys you're not actively using; a key never expires
  on its own.

## How It Actually Works

- **Every access check IAM ever performs is the same evaluation, run
  fresh on every API call: gather every policy attached to the caller
  (directly, via access groups, and inherited from parent resource
  groups or the account), union the platform and service-access roles
  they grant on the target resource, and check whether the specific
  action being invoked (e.g. `cloud-object-storage.object.write`) is
  included.** There is no cached "you're an editor" flag stamped on a
  session — a policy change on an access group takes effect on that
  member's very next request, because the decision is recomputed from
  scratch every time, not read from a stale grant.
- **Roles are not features IBM Cloud invented per-service; they're
  standardized labels (`Viewer`/`Operator`/`Editor`/`Administrator` for
  the platform, `Reader`/`Writer`/`Manager` for service access) mapped
  internally to a fixed set of fine-grained actions that each service
  registers with IAM.** That's why granting "Writer" on Cloud Object
  Storage and "Writer" on Cloud Databases produce completely different
  actual permissions — each service defines its own action list behind
  the same role name.
- **An access group is not itself an identity IAM authenticates — it's a
  policy-attachment container.** When a user or service ID is added to a
  group, IAM doesn't create a new merged identity; at evaluation time it
  simply also pulls in every policy attached to any group that identity
  is a member of. This is why removing someone from a group revokes
  access instantly (there's nothing cached to expire) but why an
  identity's *direct* policies remain even after you remove every group
  membership — group and direct policies are additive, never exclusive.
- **A service ID's API key is bound to that ID's policies, not to
  whoever created it** — the key is just a bearer credential that proves
  "I am this service ID" to the token endpoint; IAM then evaluates
  policies against the service ID, never against the human who ran
  `service-api-key-create`. That's the actual mechanism behind least
  privilege: scoping the *service ID's* policy narrowly matters, and
  scoping the creating user's own access does nothing to constrain what
  the resulting key can do.

## Cheat sheet

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

## Exercise

Create an access group called `mastery-path-developers` with `Editor` on
your `mastery-path` resource group and add your own user to it. Then create
a service ID called `mastery-path-app` with only `Writer` on Cloud Object
Storage in that same resource group (not `Administrator`, not
account-wide), and issue it an API key. Run
`ibmcloud iam service-policies mastery-path-app` and confirm the output
shows exactly one narrow policy — that's the identity Module 10's capstone
app will use.
