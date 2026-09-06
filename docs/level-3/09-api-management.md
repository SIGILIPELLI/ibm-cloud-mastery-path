# 09 · API Management (API Connect)

The microservices built across this level so far are reachable directly —
a Route on ROKS, a raw VPC load balancer. Exposing them to external
partners or third-party developers directly is a mistake: no consistent
auth, no rate limiting, no versioning story. **API Connect** puts a
managed gateway in front.

## Provision API Connect

```bash
ibmcloud resource service-instance-create apic-mastery \
  api-connect professional us-south --resource-group-name mastery-path
```

```text
Service instance apic-mastery is being created.
OK
```

`professional` plan includes the full developer portal; `lite` (free) is
enough for this module's exercises but caps call volume hard.

## Define an API from an existing backend

```yaml
# orders-api.yaml (OpenAPI 3.0, imported into API Connect)
openapi: 3.0.0
info:
  title: Orders API
  version: 1.0.0
servers:
  - url: https://frontend.roks-mastery.us-south.containers.appdomain.cloud
paths:
  /orders:
    get:
      operationId: listOrders
      security: [{ apiKeyAuth: [] }]
      responses:
        '200': { description: OK }
    post:
      operationId: createOrder
      security: [{ apiKeyAuth: [] }]
      responses:
        '201': { description: Created }
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
```

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('orders-api.yaml'))" && echo "valid YAML"
```

```text
valid YAML
```

```bash
ibmcloud apic draft-apis:create --server apic-mastery orders-api.yaml
```

```text
API 'orders-api:1.0.0' created in draft catalog.
```

## Publish to a catalog with a rate-limit policy

```bash
ibmcloud apic products:create --server apic-mastery \
  --title "Orders API — Public" \
  --apis orders-api:1.0.0 \
  --plan default \
  --rate-limit "1000/1hour"
```

```bash
ibmcloud apic products:publish --server apic-mastery orders-api-product:1.0.0 \
  --catalog production-catalog \
  --space default
```

```text
Product 'orders-api-product:1.0.0' published to catalog 'production-catalog'.
```

The rate limit lives on the *product/plan*, not the backend service — the
backend has no idea a caller was throttled; API Connect returns `429`
before the request ever reaches the frontend Route.

## Developer portal and API keys

Once published, API Connect stands up a developer portal where partners
self-register and generate API keys scoped to a specific plan:

```bash
ibmcloud apic developer-orgs:create --server apic-mastery \
  --title "Acme Logistics" --org-owner-email partner@acmelogistics.example
```

Callers then authenticate with the issued key:

```bash
curl -H "X-API-Key: 3f9c...redacted" \
  https://apic-mastery.us-south.apiconnect.appdomain.cloud/orders-api/orders
```

```text
HTTP/1.1 200 OK
[{"id":"ord_1029","status":"shipped"}]
```

## Versioning without breaking existing callers

```bash
ibmcloud apic draft-apis:create --server apic-mastery orders-api-v2.yaml
ibmcloud apic products:publish --server apic-mastery orders-api-product:2.0.0 \
  --catalog production-catalog --space default
```

Both `1.0.0` and `2.0.0` products can be live in the same catalog
simultaneously, each with its own base path (`/orders-api/v1`,
`/orders-api/v2`) — existing partner integrations keep working against
`v1` while new integrations target `v2`, and `v1` gets deprecated on its
own timeline rather than a hard cutover.

## Policies: transform, validate, protect

API Connect's assembly lets you attach policies per operation without
touching backend code:

```yaml
assembly:
  execute:
    - invoke:
        target-url: "https://frontend.roks-mastery.../orders"
    - json-to-xml:
        title: legacy-partner-format
    - gatewayscript:
        title: strip-internal-fields
        source: |
          var body = JSON.parse(context.message.body.toString());
          delete body.internalCostBasis;
          context.message.body = JSON.stringify(body);
```

Stripping internal-only fields (`internalCostBasis`) at the gateway means
the backend team can add internal fields freely without a second review
of "is this safe to expose externally" on every backend change — the
gateway is the enforcement point.

## Terraform for the API Connect instance

```hcl
resource "ibm_resource_instance" "api_connect" {
  name              = "apic-mastery"
  service           = "api-connect"
  plan              = "professional"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **API Connect gateway and the developer portal are separate moving
  parts** — publishing a product doesn't automatically mean the portal UI
  shows it; portal content sometimes needs an explicit portal sync.
- **Rate limits are per-plan, and a caller with multiple API keys across
  plans can exceed what looks like "the" limit** — design plans assuming
  a determined caller could hold more than one key.
- **OpenAPI validation is stricter at import time than most hand-written
  specs expect** — missing `operationId` or ambiguous `security`
  definitions are common import failures; validate the YAML/JSON
  syntactically first, then expect API Connect's own schema validation to
  flag semantic issues.
- **Catalog vs. space vs. organization** is a three-level hierarchy that's
  easy to get backwards — a product published to the wrong catalog is
  invisible to the portal users expecting it in another one.

## How It Actually Works

- **The API Connect gateway sits as a full reverse proxy in the request
  path, executing the assembly pipeline before anything reaches the
  backend — rate limiting is enforced there by counting requests against a
  shared counter keyed to the caller's API key and plan.** Each incoming
  request is authenticated against that key first, then the gateway checks
  and increments the plan's rolling counter (`1000/1hour` here); a caller
  over the limit gets `429` straight from the gateway's own logic and the
  `invoke` policy that would call the backend never executes. That's
  exactly why the backend "has no idea" — it's not in the call chain at
  all for a throttled request.
- **The `gatewayscript` and `json-to-xml` policies run as ordered steps in
  the same assembly pipeline as `invoke`, operating on the in-flight
  message object rather than on a copy of the backend's response sent
  separately.** `invoke` populates `context.message.body` with whatever
  the backend returned; each subsequent policy reads and mutates that same
  object in place, which is why ordering matters (stripping a field before
  a format-conversion step processes different bytes than stripping it
  after) and why the backend's actual response, internal fields included,
  genuinely leaves the backend network — it's removed downstream at the
  gateway, not withheld by the backend itself.
- **Two product versions can be live at once because each published
  product gets its own base path bound to a specific OpenAPI document and
  assembly, not because API Connect merges document versions.** Publishing
  `2.0.0` doesn't touch `1.0.0`'s gateway configuration at all — they're
  independent artifacts that happen to route to the same or different
  backend URLs; a partner's existing key, scoped to the `1.0.0` product's
  plan, simply never resolves against `/orders-api/v2` unless a separate
  subscription is created for it.
- **The developer portal is a separate content-management layer reading
  catalog/product metadata via API Connect's management API, not a live
  view directly into the gateway's runtime config.** Publishing a product
  writes its definition into the catalog; the portal periodically (or on
  an explicit "publish portal") pulls that catalog metadata to regenerate
  its own pages — which is the actual reason a freshly published product
  can be callable at the gateway before it's visible for self-service
  discovery in the portal UI.

## Cheat sheet

| Task | Command |
|---|---|
| Create API Connect instance | `ibmcloud resource service-instance-create <n> api-connect professional <region>` |
| Import an OpenAPI spec | `ibmcloud apic draft-apis:create --server <inst> <file.yaml>` |
| Create a product | `ibmcloud apic products:create --server <inst> --apis <api:ver> --plan <n>` |
| Publish to a catalog | `ibmcloud apic products:publish --server <inst> <product:ver> --catalog <cat>` |
| Create a developer org | `ibmcloud apic developer-orgs:create --server <inst> --title <name>` |
| List published products | `ibmcloud apic products:list --server <inst> --catalog <cat>` |

## Exercise

1. Write an OpenAPI 3.0 spec for two endpoints of a service from an
   earlier module, validate it with `python3 -c "import yaml..."`, and
   import it as a draft API.
2. Create a product with a rate limit and publish it to a catalog.
3. Add a `gatewayscript` assembly policy that strips one field from the
   response before it reaches the caller.
4. Publish a `v2` of the API alongside `v1` and explain the base-path
   strategy that lets both stay live at once.
