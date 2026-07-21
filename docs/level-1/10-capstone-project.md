# 10 · Capstone Project — Visit Counter

A small end-to-end app that ties together three services from this level:
**Cloud Object Storage** serves a static frontend, a **Cloud Functions** web
action handles requests, and **Databases for PostgreSQL** persists a visit
count. No servers to patch, nothing running when nobody's using it.

## Architecture

```text
Browser
  |
  |--- GET  https://<cos-bucket>.s3-web.<region>...   (static page: index.html)
  |
  '--- POST https://<region>.functions.cloud.ibm.com/.../visits-api/record
             |
             '--- INSERT ... RETURNING count   (Databases for PostgreSQL)
```

The page is pure static HTML/JS served from COS (Module 4); the only
"backend" is one Cloud Functions web action (Module 7) that talks to the
Postgres instance from Module 6.

## 1. Database: create the table

Reuse the `mastery-postgres` instance from Module 6 (or provision a fresh
one the same way):

```sql
CREATE TABLE visit_counter (
    id INT PRIMARY KEY DEFAULT 1,
    count INT NOT NULL DEFAULT 0,
    CONSTRAINT single_row CHECK (id = 1)
);

INSERT INTO visit_counter (id, count) VALUES (1, 0);
```

The `CHECK (id = 1)` constraint keeps this table to exactly one row — enough
for a simple counter.

## 2. Backend: the Cloud Functions action

```javascript
// record.js
const { Client } = require("pg");

async function main(params) {
    const client = new Client({
        host: params.PGHOST,
        port: params.PGPORT,
        database: "ibmclouddb",
        user: params.PGUSER,
        password: params.PGPASSWORD,
        ssl: { ca: params.PGCA, rejectUnauthorized: true },
    });

    await client.connect();
    const result = await client.query(
        `UPDATE visit_counter SET count = count + 1 WHERE id = 1 RETURNING count`
    );
    await client.end();

    return {
        statusCode: 200,
        headers: { "Access-Control-Allow-Origin": "*" },
        body: { count: result.rows[0].count },
    };
}

exports.main = main;
```

Package it with its one dependency (`pg`) since Cloud Functions Node.js
actions run in a clean runtime that doesn't have your local `node_modules`:

```bash
mkdir action && cd action
npm init -y && npm install pg
cp ../record.js .
zip -r record.zip record.js node_modules package.json
```

## 3. Deploy the action, bound to the database via parameters

Bind connection details as **action parameters** (encrypted at rest by the
platform) rather than hardcoding them — this is the same least-privilege
instinct from Module 2, applied to secrets instead of IAM roles.

```bash
ibmcloud fn package create visits-api

ibmcloud fn action create visits-api/record record.zip \
  --kind nodejs:18 \
  --web true \
  --param PGHOST "<host-from-module-6-service-key>" \
  --param PGPORT "<port>" \
  --param PGUSER "<username>" \
  --param PGPASSWORD "<password>" \
  --param PGCA "$(cat mastery-postgres-ca.crt)"

ibmcloud fn action get visits-api/record --url
# https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/visits-api/record
```

## 4. Frontend: a static page on Cloud Object Storage

```html
<!-- index.html -->
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Visit Counter</title></head>
<body>
  <h1>You are visitor #<span id="count">...</span></h1>
  <script>
    fetch("https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/visits-api/record", {
      method: "POST",
    })
      .then((r) => r.json())
      .then((data) => {
        document.getElementById("count").textContent = data.count;
      });
  </script>
</body>
</html>
```

```bash
ibmcloud cos object-put \
  --bucket mastery-path-site \
  --key index.html \
  --body index.html
```

Reuse the bucket and static-website configuration from Module 4. Load the
bucket's website endpoint in a browser, refresh a few times, and watch the
count increase — confirming the full loop (frontend → Functions → Postgres
→ back to frontend) works.

## 5. Test end-to-end from the command line too

```bash
curl -X POST "https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/visits-api/record"
# {"count":1}
curl -X POST "https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/visits-api/record"
# {"count":2}
```

## 6. Teardown — do this before you finish the project

!!! danger "Free-tier services are safe to leave; Databases and VSIs are not"
    Cloud Object Storage (Lite) and Cloud Functions have generous
    always-free allowances and are safe to leave provisioned. **Databases
    for PostgreSQL and any VPC Virtual Server Instance are not** — they run
    continuously and bill by the hour regardless of traffic. Delete them as
    soon as you're done, not "later."

Delete the capstone's own resources first:

```bash
# Cloud Functions
ibmcloud fn action delete visits-api/record
ibmcloud fn package delete visits-api

# COS object + bucket (bucket-delete --force also removes remaining objects)
ibmcloud cos object-delete --bucket mastery-path-site --key index.html
ibmcloud cos bucket-delete --bucket mastery-path-site --force

# The database instance is the important one -- this stops all billing for it
ibmcloud resource service-key-delete mastery-postgres-cred -f
ibmcloud resource service-instance-delete mastery-postgres -f --recursive
```

Then clean up anything left over from earlier modules in this level, if you
haven't already torn each one down as you went:

```bash
# Module 3 -- VSI, floating IP, security group, subnet, key, VPC
ibmcloud is instance-delete mastery-vsi -f
ibmcloud is floating-ip-release mastery-fip -f
ibmcloud is security-group-delete mastery-sg -f
ibmcloud is subnet-delete mastery-subnet -f
ibmcloud is key-delete mastery-key -f
ibmcloud is vpc-delete mastery-vpc -f

# Module 5 -- the networking-basics VPC, if you still have it
ibmcloud is public-gateway-delete app-pgw -f
ibmcloud is subnet-delete app-subnet-1 -f
ibmcloud is subnet-delete app-subnet-2 -f
ibmcloud is vpc-delete app-vpc -f

# Module 8 -- monitoring/logging instances
ibmcloud resource service-instance-delete mastery-monitoring -f --recursive
ibmcloud resource service-instance-delete mastery-logging -f --recursive

# Module 9 -- destroy Schematics-managed infra, then the workspace itself
ibmcloud schematics destroy --id <workspace-id>
ibmcloud schematics workspace delete --id <workspace-id>
```

## 7. Confirm nothing billable is left running

```bash
ibmcloud resource service-instances --long
ibmcloud is instances
ibmcloud is vpcs
```

Both commands should come back empty (or show only Lite-tier COS/Functions
resources) for the `mastery-path` resource group. If anything unexpected
shows up, delete it before you consider the project finished.

## Cheat sheet

| Step | Command |
|---|---|
| Delete a Functions action/package | `ibmcloud fn action delete <name>` / `ibmcloud fn package delete <name>` |
| Empty + delete a COS bucket | `ibmcloud cos bucket-delete --bucket <b> --force` |
| Delete a database/service instance | `ibmcloud resource service-instance-delete <name> -f --recursive` |
| Delete a VSI and its networking | `instance-delete`, `floating-ip-release`, `security-group-delete`, `subnet-delete`, `key-delete`, `vpc-delete` |
| Destroy Schematics-managed infra | `ibmcloud schematics destroy --id <id>` |
| Final audit | `ibmcloud resource service-instances --long` and `ibmcloud is instances` |

## Exercise

Build the Visit Counter exactly as described, verify it in a browser and
with `curl`, then work through the full teardown section and re-run the
audit commands in step 7. Paste (to yourself, in a notes file — not
required to submit anywhere) the empty output confirming the `mastery-path`
resource group has nothing billable left. Completing this means you're
ready for **Level 2 · Intermediate**, where these same building blocks get
composed into a highly available, auto-scaled web app.
