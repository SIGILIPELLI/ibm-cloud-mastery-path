# 01 · Enterprise Account Structure & IAM at Scale

Level 1's IAM module covered users, roles, and access groups inside a
single account. That model breaks down past a handful of teams — dozens
of accounts, hundreds of service IDs, and access reviews that can't be
done by eye. This module covers IBM Cloud **Enterprise** account
structure and IAM patterns that scale.

## Enterprise, account groups, and accounts

```text
Enterprise (billing + policy root)
 ├── Account Group: platform-engineering
 │     ├── Account: platform-shared-services
 │     └── Account: platform-networking-hub
 └── Account Group: product-teams
       ├── Account: orders-prod
       ├── Account: orders-dev
       └── Account: catalog-prod
```

```bash
ibmcloud enterprise account-group-create \
  --name product-teams \
  --parent $(ibmcloud enterprise list --output json | jq -r '.resources[0].crn')
```

```text
Account group 'product-teams' created.
ID: abcdef1234567890
```

```bash
ibmcloud enterprise account-create \
  --name orders-prod \
  --parent $(ibmcloud enterprise account-group product-teams --output json | jq -r .crn) \
  --owner-email orders-team-lead@example.com
```

```text
Account creation request submitted.
The invited owner must accept the invitation to activate the account.
```

Separate accounts (not just resource groups) per environment/team gives a
harder isolation boundary than resource groups alone — IAM policies,
quotas, and even a compromised API key are scoped to one account, not the
whole enterprise.

## Enterprise-wide IAM: trusted profiles across accounts

A trusted profile lets a workload in one account assume access in another
without a shared static credential — the enterprise-scale replacement for
copying API keys between accounts:

```bash
ibmcloud iam trusted-profile-create cross-account-ci \
  --description "CI pipeline in platform-shared-services deploying to orders-prod"

ibmcloud iam trusted-profile-identity-add cross-account-ci \
  --identity-type crn \
  --identifier crn:v1:bluemix:public:iam-identity::a/<shared-services-account>::serviceid:ServiceId-abc123

ibmcloud iam trusted-profile-policy-create cross-account-ci \
  --roles Editor \
  --service-name containers-kubernetes \
  --account-id <orders-prod-account-id>
```

```text
Policy created for trusted profile 'cross-account-ci'.
```

The CI service ID in `platform-shared-services` now assumes the profile
to get `Editor` on `orders-prod`'s Kubernetes service — scoped, auditable,
and revocable by deleting the trusted profile, without ever creating a
credential that lives in `orders-prod`.

## Access groups at enterprise scale: dynamic rules

Manually adding hundreds of users to access groups doesn't scale. Dynamic
rules add users automatically based on identity provider (SAML/OIDC)
attributes:

```bash
ibmcloud iam access-group-create orders-team-engineers

ibmcloud iam access-group-dynamic-rule-create orders-team-engineers \
  --name "sso-orders-team" \
  --expiration 24 \
  --identity-provider "https://sso.example.com/saml" \
  --conditions 'claim=department,operator=EQUALS,value=orders-engineering'
```

```text
Dynamic rule 'sso-orders-team' created for access group 'orders-team-engineers'.
```

A user federating in via SAML with `department=orders-engineering` picks
up the group's policies automatically for a 24-hour session and loses
them the moment that claim changes or the session expires — access
follows the HR/identity system of record instead of a manually
maintained list.

## Enforce MFA and session controls enterprise-wide

```bash
ibmcloud iam account-settings-update \
  --mfa TOTP4ALL \
  --session-invalidation-in-seconds 28800 \
  --restrict-create-service-id RESTRICTED \
  --restrict-create-platform-apikey RESTRICTED
```

```text
OK
Account settings updated.
```

`RESTRICTED` on service-ID and API-key creation means only users with an
explicit IAM policy grant (not just any authenticated user) can create
new service IDs or platform API keys — closing the common "any developer
can mint a long-lived credential" gap.

## Periodic access review

```bash
ibmcloud iam access-group-users orders-team-engineers --output json | \
  jq '.[] | {name: .iam_id, added: .created_at}'
```

```bash
ibmcloud iam service-ids --output json | \
  jq '.[] | select((.created_at | fromdateiso8601) < (now - 7776000)) | .name'
```

The second command lists service IDs older than 90 days — a stale service
ID nobody remembers creating is one of the most common findings in a real
enterprise access review, and this is exactly the kind of check SCC
(Level 3, Module 03) can also be configured to flag continuously.

## Terraform for the account group and trusted profile

```hcl
resource "ibm_iam_account_settings" "org_wide" {
  mfa                              = "TOTP4ALL"
  session_invalidation_in_seconds  = 28800
  restrict_create_service_id       = "RESTRICTED"
}

resource "ibm_iam_trusted_profile" "cross_account_ci" {
  name        = "cross-account-ci"
  description = "CI pipeline deploying to orders-prod"
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Enterprise account creation is not instant** — a new account needs
  the invited owner to accept before it's usable; automation that assumes
  synchronous creation will poll forever against a pending account.
- **Trusted profiles scoped too broadly** (`Administrator` instead of a
  narrow service role) reintroduce the exact static-credential risk they
  were meant to remove — scope every profile policy to the minimum
  service and role actually needed.
- **Dynamic rules depend on the identity provider sending the expected
  claim** — a SAML assertion missing the configured attribute silently
  fails to match, and the user gets no group membership with no obvious
  error.
- **`restrict-create-platform-apikey` applies account-wide** — enabling it
  without first granting the relevant IAM policy to the people who
  legitimately need to create keys locks out normal operations, not just
  the risky case.

## Cheat sheet

| Task | Command |
|---|---|
| Create account group | `ibmcloud enterprise account-group-create --name <n> --parent <crn>` |
| Create account under a group | `ibmcloud enterprise account-create --name <n> --parent <crn>` |
| Create trusted profile | `ibmcloud iam trusted-profile-create <n>` |
| Add identity to profile | `ibmcloud iam trusted-profile-identity-add <profile> --identity-type crn --identifier <crn>` |
| Create dynamic access-group rule | `ibmcloud iam access-group-dynamic-rule-create <group> --identity-provider <url> --conditions '<expr>'` |
| Update account-wide IAM settings | `ibmcloud iam account-settings-update --mfa TOTP4ALL` |

## Exercise

1. Design (as an enterprise account-group tree, in a diagram or Terraform)
   at least two account groups and three accounts for a fictional
   two-team org.
2. Create a trusted profile that lets a service ID in one account assume
   a scoped role in another, and explain why that's preferable to sharing
   an API key.
3. Write a dynamic access-group rule keyed on a SAML claim, and describe
   what happens to a user's access when that claim changes.
4. Write a one-liner that lists service IDs older than 90 days, and
   explain what you'd do with that list in a real quarterly access
   review.
