# 06 · Cloud Databases

**Databases for PostgreSQL** is one of several fully-managed engines under
IBM Cloud's "Databases for..." family (also MySQL, MongoDB, Redis,
Elasticsearch, and more) — IBM handles patching, backups, and failover, you
handle schema and queries. This module provisions an instance and connects
an app to it, the same database the capstone in Module 10 will use.

## Provision a PostgreSQL instance

```bash
ibmcloud resource service-instance-create mastery-postgres \
  databases-for-postgresql standard us-south \
  --resource-group-name mastery-path \
  -p '{"members_memory_allocation_mb": 1024, "members_disk_allocation_mb": 5120}'

# Provisioning takes several minutes -- poll status
ibmcloud resource service-instance mastery-postgres
```

## Get connection credentials

```bash
ibmcloud resource service-key-create mastery-postgres-cred Administrator \
  --instance-name mastery-postgres

ibmcloud resource service-key mastery-postgres-cred --output json
```

The key contains a `connection.postgres` block with `hosts`, `authentication`
(username/password), the default `database` name, and a `certificate` (base64
CA cert) — PostgreSQL deployments require TLS by default.

```bash
# Save the CA cert locally so psql/your driver can verify the connection
echo "<certificate_base64>" | base64 -d > mastery-postgres-ca.crt
```

## Connect with `psql`

```bash
psql "host=<hostname> port=<port> dbname=ibmclouddb \
  user=<username> password=<password> \
  sslmode=verify-full sslrootcert=mastery-postgres-ca.crt"
```

```sql
CREATE TABLE visits (
    id SERIAL PRIMARY KEY,
    path TEXT NOT NULL,
    visited_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO visits (path) VALUES ('/');
SELECT * FROM visits;
```

## Connecting from application code (Node.js example)

```javascript
// db.js
const { Pool } = require("pg");
const fs = require("fs");

const pool = new Pool({
  host: process.env.PGHOST,
  port: process.env.PGPORT,
  database: "ibmclouddb",
  user: process.env.PGUSER,
  password: process.env.PGPASSWORD,
  ssl: {
    ca: fs.readFileSync("./mastery-postgres-ca.crt").toString(),
    rejectUnauthorized: true,
  },
});

async function recordVisit(path) {
  const result = await pool.query(
    "INSERT INTO visits (path) VALUES ($1) RETURNING id, visited_at",
    [path]
  );
  return result.rows[0];
}

module.exports = { recordVisit };
```

Credentials come from environment variables sourced from the service key —
never hardcode them, and never commit the CA cert alongside secrets in a
public repo.

## Backups and scaling

```bash
# On-demand backup (automatic daily backups are already included)
ibmcloud cdb deployment-backup mastery-postgres

# List available backups / point-in-time recovery windows
ibmcloud cdb deployment-backups mastery-postgres

# Scale memory/disk (triggers a rolling, minimal-downtime resize)
ibmcloud cdb deployment-scale mastery-postgres \
  --memory 2048 --disk 10240
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud resource service-instance-create <name> databases-for-postgresql <plan> <region>` | Provision a PostgreSQL deployment |
| `ibmcloud resource service-key-create <key> Administrator --instance-name <db>` | Get connection credentials |
| `ibmcloud cdb deployment-backup <name>` | Trigger an on-demand backup |
| `ibmcloud cdb deployment-backups <name>` | List backups |
| `ibmcloud cdb deployment-scale <name> --memory <mb> --disk <mb>` | Resize a deployment |
| `ibmcloud resource service-instance-delete <name>` | Deprovision the deployment |

## Exercise

Provision a `databases-for-postgresql` instance on the `standard` plan,
create a service key, extract the CA certificate, and connect with `psql`
using `sslmode=verify-full`. Create the `visits` table above, insert three
rows with different `path` values, then write a `SELECT ... GROUP BY path`
query that counts visits per path.
