# 04 · Cloud Object Storage

**Cloud Object Storage (COS)** is IBM Cloud's S3-compatible object store —
buckets full of objects (files, blobs, whatever), accessed over HTTP(S)
instead of a filesystem. It has an always-free Lite tier, which makes it the
natural place to host the capstone project's static frontend in Module 10.

## Create a COS instance

A COS **instance** is the billable resource; buckets live inside it.

```bash
ibmcloud resource service-instance-create mastery-cos \
  cloud-object-storage lite global \
  --resource-group-name mastery-path
```

`lite` here is the pricing plan, and `global` is the "region" — COS
instances themselves aren't tied to one region; individual buckets are.

## Resiliency options

Choose resiliency **per bucket**, not per instance, based on how much
geographic redundancy you need versus latency:

| Resiliency | What it means | Good for |
|---|---|---|
| **Cross Region** | Data replicated across multiple geographic regions (e.g. all of `us` or `eu`) | Maximum durability/availability, DR-sensitive data |
| **Regional** | Data replicated across availability zones within one region | Lower latency to compute in that region, still zone-fault-tolerant |
| **Single Data Center** | Data stored in one data center, no automatic replication | Lowest cost, non-critical/transient data |

## Create a bucket

```bash
# Get credentials the CLI/SDKs use to talk to the S3-compatible API
ibmcloud resource service-key-create mastery-cos-key Writer \
  --instance-name mastery-cos

ibmcloud resource service-key mastery-cos-key --output json

# Create a regional bucket via the cos-cli plugin
ibmcloud cos bucket-create \
  --bucket mastery-path-site \
  --ibm-service-instance-id <instance-id-from-above> \
  --region us-south \
  --class standard
```

Storage **classes** (Standard, Vault, Cold Vault, Flex) trade retrieval speed
for storage cost — Standard for frequently accessed data, Vault/Cold Vault
for archival, Flex to auto-tier based on access patterns.

## Upload and download objects

```bash
ibmcloud cos object-put \
  --bucket mastery-path-site \
  --key index.html \
  --body ./index.html

ibmcloud cos objects --bucket mastery-path-site

ibmcloud cos object-get \
  --bucket mastery-path-site \
  --key index.html \
  ./downloaded-index.html
```

## Static website hosting

A bucket can serve its contents directly as a static website — exactly what
the capstone project's frontend uses.

```bash
ibmcloud cos bucket-website-put \
  --bucket mastery-path-site \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

# Public read access is required for a public static site -- scope it to
# GetObject only, never broader
ibmcloud cos bucket-policy-put \
  --bucket mastery-path-site \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::mastery-path-site/*"]
    }]
  }'
```

The public website endpoint follows the pattern
`https://<bucket>.s3-web.<region>.cloud-object-storage.appdomain.cloud`.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> cloud-object-storage <plan> global` | Create a COS instance |
| `ibmcloud resource service-key-create <name> Writer --instance-name <cos>` | Issue API/HMAC credentials |
| `ibmcloud cos bucket-create --bucket <name> --region <r> --class <c>` | Create a bucket |
| `ibmcloud cos object-put --bucket <b> --key <k> --body <file>` | Upload an object |
| `ibmcloud cos object-get --bucket <b> --key <k> <dest>` | Download an object |
| `ibmcloud cos objects --bucket <b>` | List objects in a bucket |
| `ibmcloud cos bucket-website-put --bucket <b> --website-configuration <json>` | Enable static website hosting |
| `ibmcloud cos bucket-delete --bucket <b> --force` | Delete a bucket and its contents |

## Exercise

Create a COS instance and a **regional**, Standard-class bucket named after
your own project. Upload a small `index.html` you write yourself, enable
static website hosting on the bucket, apply the public-read bucket policy
above, and load the resulting website endpoint URL in a browser to confirm
it serves your page.
