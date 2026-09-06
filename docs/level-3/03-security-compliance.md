# 03 · Security & Compliance (Security Advisor, Key Protect)

Levels 1–2 covered IAM roles and security groups — controlling *who* and
*what network path* can reach a resource. This module covers *proving* the
account stays that way over time: centralized findings via **Security and
Compliance Center (SCC)**, and centralized key management via **Key
Protect**.

## Key Protect: bring your own encryption keys

Most IBM Cloud storage services (Cloud Object Storage, block storage,
databases) encrypt at rest by default with IBM-managed keys. For anything
regulated, root keys should be customer-managed instead, via Key Protect.

```bash
ibmcloud resource service-instance-create keyprotect-mastery \
  kms tiered-pricing us-south --resource-group-name mastery-path
```

```text
Service instance keyprotect-mastery is being created.
OK
```

```bash
ibmcloud kp key create root-key-orders \
  --instance-id $(ibmcloud resource service-instance keyprotect-mastery --output json | jq -r .[0].guid) \
  --key-ring-name orders-app \
  --standard-key false
```

```text
OK
Key root-key-orders was created.
ID: 6f2c1a9e-....
```

`--standard-key false` creates a **root key** (used to wrap other keys,
never leaves Key Protect in plaintext), as opposed to a standard key stored
directly. Root keys are what services like Cloud Object Storage reference
for "bring your own key" (BYOK) or, with a dedicated HSM-backed instance,
"keep your own key" (KYOK, covered in Level 4's Hyper Protect module).

## Attach a Key Protect root key to Cloud Object Storage

```bash
ibmcloud cos bucket-create --bucket orders-archive-mastery \
  --ibm-service-instance-id $(ibmcloud resource service-instance cos-mastery --output json | jq -r .[0].guid) \
  --region us-south \
  --class standard \
  --kms-key-crn $(ibmcloud kp key root-key-orders --instance-id <kp-guid> --output json | jq -r .crn)
```

```text
Bucket orders-archive-mastery created
```

**Gotcha**: once a bucket is created with a specific root key, you cannot
swap the key later without recreating the bucket and re-copying objects —
decide on key ownership (which team's Key Protect instance) before
creating production buckets, not after.

## Rotation and dual authorization

```bash
ibmcloud kp key rotate root-key-orders \
  --instance-id <kp-guid>
```

```text
OK
Key root-key-orders was rotated.
```

Rotating creates a new key version; services referencing the key by CRN
pick it up automatically re-wrapping data keys, with no data
re-encryption needed (envelope encryption). For keys protecting regulated
data, enable **dual authorization** so no single administrator can delete
a key unilaterally:

```bash
ibmcloud kp instance dual-auth-enable --instance-id <kp-guid>
```

## Security and Compliance Center: continuous posture checks

SCC evaluates the account against a chosen profile (CIS IBM Cloud
Foundations Benchmark, or a custom profile) continuously, rather than at a
point-in-time audit.

```bash
ibmcloud resource service-instance-create scc-mastery \
  compliance free-plan us-south --resource-group-name mastery-path
```

```bash
ibmcloud scc profile-attach \
  --profile "CIS IBM Cloud Foundations Benchmark" \
  --scope-id $(ibmcloud account show --output json | jq -r .account_id) \
  --instance-id $(ibmcloud resource service-instance scc-mastery --output json | jq -r .[0].guid)
```

```text
Attachment created.
Name: cis-foundations-mastery
Status: pending
```

Findings appear after the first scan cycle (typically within an hour):

```bash
ibmcloud scc results --attachment-id <attachment-id>
```

```text
Control                                   Status    Resources
1.2 Ensure API keys are rotated           FAILED    3 non-compliant
2.1 Ensure COS buckets are not public     PASSED    12 compliant
4.3 Ensure MFA is enabled for all users   FAILED    2 non-compliant
```

## Event Notifications: don't let findings sit unread

Wire SCC findings to Event Notifications so a failed control pages
someone instead of waiting to be noticed in the console:

```bash
ibmcloud resource service-instance-create notify-mastery \
  event-notifications lite us-south --resource-group-name mastery-path

ibmcloud en source-create --instance-id <en-guid> \
  --name scc-findings --description "SCC compliance findings" \
  --enabled true
```

Connect the SCC instance as an Event Notifications source in the console
(no CLI subcommand for this wiring as of the current CLI plugin version —
a known gap worth noting rather than working around with a script).

## Activity Tracker: the audit trail underneath SCC

SCC tells you current posture; **Activity Tracker** answers "who did
this and when":

```bash
ibmcloud atracker target create \
  --name cos-audit-target \
  --target-type cloud_object_storage \
  --cos-endpoint s3.us-south.cloud-object-storage.appdomain.cloud \
  --cos-bucket audit-logs-mastery \
  --cos-instance-id $(ibmcloud resource service-instance cos-mastery --output json | jq -r .[0].guid)

ibmcloud atracker route create \
  --name all-events-route \
  --rules '[{"target_ids":["<target-id>"],"locations":["us-south","global"]}]'
```

```text
OK
Route all-events-route created.
```

## Gotchas

- **Key Protect instance deletion is a two-step, delayed operation** —
  keys go through a "scheduled for deletion" window (default 30 days)
  before permanent removal, specifically to prevent accidental data loss;
  plan around that window rather than assuming instant deletion.
- **SCC profile attachments scope to an account or a resource group** —
  attaching at account scope and expecting per-resource-group exceptions
  needs separate attachments with narrower scopes, not one attachment with
  exclusions.
- **Activity Tracker routes are additive** — multiple routes can all match
  the same event and each writes a copy to its target, which silently
  multiplies storage costs if routes overlap.
- **Free-plan SCC and Activity Tracker have retention/volume limits** —
  fine for this module's exercise, not for a production compliance
  program.

## How It Actually Works

- **Key rotation needs no data re-encryption because of envelope
  encryption, the same mechanism Level 2's COS module covered.** Each
  object or volume has its own data-encryption key (DEK) wrapped by the
  Key Protect root key, and rotating the root key only changes what future
  wrap operations use — existing DEKs already wrapped under the prior
  version stay valid because Key Protect retains old key versions
  internally to keep unwrapping them, it just stops issuing new wraps with
  them. That's the whole reason rotation is near-instant regardless of how
  much data the key protects.
- **SCC's continuous checks work by periodically querying each resource's
  actual configuration through the same APIs you'd call directly** — a
  control like "COS buckets are not public" isn't a static rule matched
  against a snapshot; SCC's scanner calls the COS API to read each
  bucket's public-access setting on every scan cycle and compares it
  against the profile's expected value, which is why a manual console
  change that violates a control shows up as a new failure on the very
  next scan rather than staying silently unnoticed.
- **The 30-day deletion window on a Key Protect instance is a scheduled
  hold, not a soft-delete flag your code can query around** — the key
  material stays fully wrappable/unwrappable during that window (a
  cancellation restores full use), and only after it elapses does Key
  Protect irreversibly destroy the key. Since deleting the wrapping root
  key is what makes envelope-encrypted data unrecoverable, that window
  exists specifically as the last chance to notice a wrong deletion before
  every object it protects becomes permanently unreadable.
- **Activity Tracker routes are additive because each one independently
  subscribes to the same underlying platform event stream** — every
  IAM-authenticated API call across the account emits an event onto that
  stream regardless of whether any route exists, and each configured route
  is simply a filter-plus-forward rule matching those events into its own
  target. Two overlapping routes both fire because they're two separate
  subscriptions reading the same stream, not two paths through one
  pipeline — hence the doubled storage cost.

## Cheat sheet

| Task | Command |
|---|---|
| Create Key Protect instance | `ibmcloud resource service-instance-create <n> kms tiered-pricing <region>` |
| Create root key | `ibmcloud kp key create <n> --instance-id <id> --standard-key false` |
| Rotate a key | `ibmcloud kp key rotate <key> --instance-id <id>` |
| Enable dual auth | `ibmcloud kp instance dual-auth-enable --instance-id <id>` |
| Attach SCC profile | `ibmcloud scc profile-attach --profile <name> --scope-id <account>` |
| View SCC results | `ibmcloud scc results --attachment-id <id>` |
| Create Activity Tracker route | `ibmcloud atracker route create --name <n> --rules '[...]'` |

## Exercise

1. Create a Key Protect instance and a root key, then create a Cloud
   Object Storage bucket referencing that key's CRN.
2. Rotate the root key and confirm (via `ibmcloud kp key versions`) a new
   version was created without touching the bucket's contents.
3. Attach the CIS IBM Cloud Foundations Benchmark profile via SCC and
   review at least three findings once the first scan completes.
4. Create an Activity Tracker route targeting a Cloud Object Storage
   bucket and describe, in prose, what event categories (`global`,
   region-specific) it will capture.
