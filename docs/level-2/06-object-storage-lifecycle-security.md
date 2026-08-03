# 06 · Object Storage Lifecycle & Security

Level 1's COS module (Module 4) covered creating buckets and putting
objects in them by hand. Production buckets need more than that: rules
that age out or archive old objects automatically, retention that stops
even an account owner from deleting data too early, encryption keys you
control instead of IBM's default, and access scoped to one bucket instead
of the whole COS instance. This module covers all four.

## Lifecycle rules: let objects age out on their own

A **lifecycle configuration** is a set of rules, evaluated daily, that
transition or expire objects matching a prefix once they reach a certain
age — no cron job, no script polling the bucket.

```bash
ibmcloud cos bucket-lifecycle-configuration-put \
  --bucket mastery-path-site \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "expire-old-logs",
        "Filter": {"Prefix": "logs/"},
        "Status": "Enabled",
        "Expiration": {"Days": 90}
      },
      {
        "ID": "archive-uploads",
        "Filter": {"Prefix": "uploads/"},
        "Status": "Enabled",
        "Transitions": [
          {"Days": 30, "StorageClass": "GLACIER"}
        ]
      }
    ]
  }'

ibmcloud cos bucket-lifecycle-configuration-get --bucket mastery-path-site
```

**Gotcha:** `Transitions` moves an object to a colder, cheaper storage
class (mapped onto COS's Vault/Cold Vault classes) — it does not delete
it, and retrieving a transitioned object later can incur a retrieval
charge and a delay measured in hours, not milliseconds. Don't transition
data your application still reads on a normal request path.

## Retention and Object Lock: data even you can't delete early

**Retention** protects objects from deletion or overwrite until a minimum
period has passed, independent of any IAM policy — useful for compliance
data (audit logs, financial records) where "an admin account got
compromised" should never be a path to destroying evidence.

```bash
# Enable Object Lock at bucket creation -- cannot be turned on afterward
ibmcloud cos bucket-create \
  --bucket mastery-path-audit-logs \
  --region us-south \
  --class standard \
  --object-lock

# Set a default retention period for every object added to the bucket
ibmcloud cos object-lock-configuration-put \
  --bucket mastery-path-audit-logs \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {"Mode": "COMPLIANCE", "Days": 365}
    }
  }'

# Or set retention per object, for finer control
ibmcloud cos object-retention-put \
  --bucket mastery-path-audit-logs \
  --key 2026-08-audit.log \
  --retention '{"Mode": "COMPLIANCE", "RetainUntilDate": "2027-08-01T00:00:00Z"}'
```

**Gotcha:** `--object-lock` can only be set at bucket creation time — there
is no `bucket-update` path to add it to an existing bucket. If you forget
and need retention later, the fix is create a new Object-Lock-enabled
bucket and copy the data across, not a flag change on the old one.
`COMPLIANCE` mode is stricter than `GOVERNANCE` mode: under `COMPLIANCE`,
*nobody* — not even the account owner — can shorten or remove the
retention period before it expires.

## Legal hold: retention with no fixed end date

A **legal hold** blocks deletion indefinitely, independent of any
retention period, and is lifted explicitly rather than expiring:

```bash
ibmcloud cos object-legal-hold-put \
  --bucket mastery-path-audit-logs \
  --key 2026-08-audit.log \
  --legal-hold '{"Status": "ON"}'

# Lift it once the hold reason (an investigation, a litigation matter) resolves
ibmcloud cos object-legal-hold-put \
  --bucket mastery-path-audit-logs \
  --key 2026-08-audit.log \
  --legal-hold '{"Status": "OFF"}'
```

Retention and legal hold stack — an object can be past its retention date
and still undeletable because a legal hold is still `ON`.

## Encryption: default vs. customer-managed keys

Every COS bucket is encrypted at rest by default with IBM-managed keys —
no configuration required. For data where you need to control (and be
able to revoke) the encryption key yourself, bring your own via **Key
Protect** or **Hyper Protect Crypto Services**:

```bash
ibmcloud resource service-instance-create mastery-keyprotect \
  kms tiered-pricing us-south \
  --resource-group-name mastery-path

ibmcloud kp key create mastery-cos-root-key \
  --instance-id <keyprotect-instance-id> \
  --standard-key

# Create a bucket that encrypts with that root key instead of IBM's default
ibmcloud cos bucket-create \
  --bucket mastery-path-secure \
  --region us-south \
  --class standard \
  --ibm-sse-kp-customer-root-key-crn crn:v1:bluemix:public:kms:us-south:a/...:key:...
```

**Gotcha:** if you ever delete the Key Protect root key backing a bucket,
every object in that bucket becomes permanently unreadable — COS cannot
decrypt data whose key no longer exists, and there's no recovery path
short of a key backup taken beforehand. Treat root key deletion with the
same care as deleting the data itself.

## IAM access scoped to one bucket, not the whole instance

Level 1's IAM module scoped policies to a resource group. For COS you can
go one level tighter — scope a policy to a single bucket by CRN, so a
service ID that only needs to write to `mastery-path-site` can't touch
`mastery-path-audit-logs` in the same instance:

```bash
ibmcloud iam service-policy-create mastery-path-app \
  --roles Writer \
  --service-name cloud-object-storage \
  --service-instance <cos-instance-id> \
  --resource-type bucket \
  --resource mastery-path-site
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud cos bucket-lifecycle-configuration-put --bucket <b> --lifecycle-configuration <json>` | Set expiration/transition rules |
| `ibmcloud cos bucket-lifecycle-configuration-get --bucket <b>` | View current lifecycle rules |
| `ibmcloud cos bucket-create --bucket <b> --object-lock` | Create a bucket with Object Lock enabled (creation-time only) |
| `ibmcloud cos object-lock-configuration-put --bucket <b> --object-lock-configuration <json>` | Set a default retention mode/period |
| `ibmcloud cos object-retention-put --bucket <b> --key <k> --retention <json>` | Set retention on one object |
| `ibmcloud cos object-legal-hold-put --bucket <b> --key <k> --legal-hold <json>` | Toggle a legal hold on/off |
| `ibmcloud cos bucket-create --bucket <b> --ibm-sse-kp-customer-root-key-crn <crn>` | Encrypt a bucket with your own Key Protect key |
| `ibmcloud iam service-policy-create <id> --service-name cloud-object-storage --resource-type bucket --resource <bucket>` | Scope IAM access to one bucket |

## Exercise

Create a bucket with Object Lock enabled, set a `COMPLIANCE` default
retention of a few minutes (for testing — real audit data would use
months or years), upload an object, and confirm `ibmcloud cos
object-delete` fails while the retention window is still open. Separately,
set a lifecycle rule on `mastery-path-site` from Module 4 (Level 1) that
expires anything under `tmp/` after 1 day, and issue a service ID a policy
scoped to that one bucket rather than the whole COS instance.
