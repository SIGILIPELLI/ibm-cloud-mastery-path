# 04 · Data & AI Services (watsonx, Db2 Warehouse)

Every database module so far used Databases for PostgreSQL —
transactional, row-oriented. This module adds two different workload
shapes: **Db2 Warehouse** for analytical/columnar queries at scale, and
**watsonx.ai** for foundation-model inference wired into an application.

## Provision Db2 Warehouse

```bash
ibmcloud resource service-instance-create orders-analytics \
  dashdb-for-transactions performance us-south \
  --parameters '{"num_cpu": 4, "storage_gb": 500}'
```

```text
Service instance orders-analytics is being created.
OK
```

`performance` plan is compute-elastic; use it for analytical workloads
whose query load spikes rather than staying flat, unlike the steady
transactional load Databases for PostgreSQL was sized for in Level 1.

## Load data from the transactional database

Analytical warehouses work best fed by an ETL pipeline, not queried
directly against a live OLTP database (Level 3's Databases for PostgreSQL
instances weren't sized for large scans):

```bash
ibmcloud cdb deployment-connections orders-db --output json | jq -r .connection.postgres.hosts[0]
```

```sql
-- Db2 Warehouse: external table over a Cloud Object Storage export
CREATE EXTERNAL TABLE orders_export (
  order_id VARCHAR(32),
  customer_id VARCHAR(32),
  total_cents INT,
  created_at TIMESTAMP
)
USING (
  DATAOBJECT 's3://orders-archive-mastery/exports/orders/'
  FORMAT 'parquet'
);

CREATE TABLE orders_fact AS
  SELECT * FROM orders_export;
```

```text
DB20000I  The SQL command completed successfully.
```

Exporting periodically to the `orders-archive-mastery` Cloud Object
Storage bucket (already built in Level 3's DR module) as Parquet, then
loading via external table, keeps the transactional database's I/O
budget for transactions, not analytics.

## Provision watsonx.ai

```bash
ibmcloud resource service-instance-create watsonx-mastery \
  pm-20 lite us-south --resource-group-name mastery-path
```

```text
Service instance watsonx-mastery is being created.
OK
```

## Call a foundation model from application code

```python
from ibm_watsonx_ai import APIClient, Credentials
from ibm_watsonx_ai.foundation_models import ModelInference

creds = Credentials(
    url="https://us-south.ml.cloud.ibm.com",
    api_key=os.environ["WATSONX_API_KEY"],
)
client = APIClient(creds)
client.set.default_project(os.environ["WATSONX_PROJECT_ID"])

model = ModelInference(
    model_id="ibm/granite-13b-instruct-v2",
    api_client=client,
    params={"decoding_method": "greedy", "max_new_tokens": 200},
)

response = model.generate_text(
    prompt=f"Summarize this order's shipping status in one sentence: {order_json}"
)
print(response)
```

```text
"Order ord_1029 shipped from the us-south fulfillment center on 2026-08-24 and is expected to arrive within 3 business days."
```

## Ground responses with your own data (RAG pattern)

```python
from ibm_watsonx_ai.foundation_models import ModelInference

# Embed and store order-policy documents once (offline step)
# ... vector store setup omitted — any COS-backed vector DB works ...

def answer_policy_question(question: str, retrieved_chunks: list[str]) -> str:
    context = "\n".join(retrieved_chunks)
    prompt = f"""Using only the context below, answer the question.
Context:
{context}

Question: {question}
Answer:"""
    return model.generate_text(prompt=prompt)
```

Retrieval-augmented generation matters here for the same reason API
Connect's gateway policies mattered in Level 3: it constrains what the
model is allowed to say to information your own system provided,
substantially reducing (never eliminating) fabricated answers about
things like return policy specifics.

## Governance: watsonx.governance for tracking model behavior

```bash
ibmcloud resource service-instance-create watsonx-governance \
  aiopenscale lite us-south
```

Wire a deployed model's inference calls to log through watsonx.governance
to track drift (are responses trending away from historical baseline) and
fairness metrics over time — the AI-specific analogue of Activity Tracker
and SCC: it isn't enough to deploy a model once, its behavior needs
ongoing monitoring the same way infrastructure posture does.

## Terraform for the service instances

```hcl
resource "ibm_resource_instance" "db2_warehouse" {
  name              = "orders-analytics"
  service           = "dashdb-for-transactions"
  plan              = "performance"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}

resource "ibm_resource_instance" "watsonx" {
  name              = "watsonx-mastery"
  service           = "pm-20"
  plan              = "lite"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Don't point BI dashboards at the OLTP database** — the temptation to
  skip the warehouse and query `orders-db` directly for reports
  reintroduces the exact contention problem Db2 Warehouse exists to
  avoid; keep the ETL boundary even when it's tempting to skip for a
  "quick" report.
- **Foundation model token limits and cost scale with prompt length,
  including retrieved RAG context** — an unbounded number of retrieved
  chunks stuffed into a prompt is a silent cost and latency problem, not
  just an accuracy one; cap retrieved context deliberately.
- **`lite` plans on watsonx have low request-per-minute caps** — fine for
  this module's exercises, will visibly throttle under any real
  application load; check plan limits before assuming a production
  workload can run on the free tier.
- **Model outputs are not deterministic** even with `greedy` decoding
  across model version updates — pin a specific model version
  (`ibm/granite-13b-instruct-v2`, not an unpinned "latest" alias) for
  anything where consistent output matters, such as automated report
  generation.

## Cheat sheet

| Task | Command |
|---|---|
| Create Db2 Warehouse instance | `ibmcloud resource service-instance-create <n> dashdb-for-transactions performance <region>` |
| Create watsonx.ai instance | `ibmcloud resource service-instance-create <n> pm-20 lite <region>` |
| Create watsonx.governance instance | `ibmcloud resource service-instance-create <n> aiopenscale lite <region>` |
| List service instance plans | `ibmcloud catalog service-marketplace <service> --output json` |

## Exercise

1. Design an ETL flow (prose plus a SQL `CREATE EXTERNAL TABLE`
   statement) moving order data from Cloud Object Storage Parquet exports
   into Db2 Warehouse.
2. Write a Python snippet calling a watsonx.ai foundation model with a
   prompt built from a variable (not a hardcoded string).
3. Extend that snippet into a minimal RAG pattern: retrieve a short list
   of text chunks and constrain the prompt to answer only from them.
4. Explain, in a few sentences, what watsonx.governance would add on top
   of a working RAG integration that inference alone doesn't provide.
