# 04 · Code Engine (Containers & Serverless)

IKS in Module 3 is powerful but always-on: worker nodes bill by the hour
whether or not any pod is handling traffic. **Code Engine** runs the same
container images with none of that node management — you get a fully
managed, serverless platform that can scale to zero between requests, plus
a batch **jobs** primitive for run-to-completion work that doesn't fit the
"app" shape at all.

## Apps vs. jobs vs. function

Code Engine has three workload types, and picking the right one matters:

- **Application** — long-running, request-driven, HTTP(S) traffic in.
  Scales on concurrent requests, can scale to zero when idle.
- **Job** — run-to-completion, no inbound HTTP. Good for batch processing,
  migrations, scheduled cleanup. Runs to a defined number of instances and
  exits.
- **Function** — single-purpose code (not a container you build) for the
  smallest, most event-driven pieces; closer to Level 1's Cloud Functions
  module than to the app/job model.

This module focuses on apps and jobs, since most Level 2 workloads are
either "serve traffic" or "do batch work."

## Create a project

A **Code Engine project** is the resource container for apps, jobs, and
their shared configuration — roughly Code Engine's equivalent of a
Kubernetes namespace.

```bash
ibmcloud ce project create --name mastery-ce --resource-group-name mastery-path

# Select it as the target for subsequent commands
ibmcloud ce project select --name mastery-ce
```

## Deploy an app from source (no Dockerfile needed)

Code Engine can build directly from a Git repo or local source using
Cloud Native Buildpacks, so a simple app doesn't need a hand-written
Dockerfile at all:

```bash
ibmcloud ce app create \
  --name hello-web \
  --build-source . \
  --strategy buildpacks \
  --port 8080 \
  --cpu 0.5 --memory 1G \
  --min-scale 0 \
  --max-scale 5
```

```text
Creating image build 'hello-web-build' and submitting source code from '.'...
Run 'ibmcloud ce buildrun get -n hello-web-build-run-...' to check the build status.
...
App 'hello-web' is ready.
URL: https://hello-web.abcd1234.us-south.codeengine.appdomain.cloud
```

`--min-scale 0` is the point of the whole module — with no traffic, Code
Engine stops all instances and billing drops to zero, then cold-starts a
new instance on the next request.

## Deploy from a prebuilt image instead

If you already built and pushed to Container Registry in Module 3, deploy
straight from that image and skip the build step:

```bash
ibmcloud ce app create \
  --name hello-web-img \
  --image icr.io/mastery-path/hello-web:1.0 \
  --port 8080 \
  --min-scale 1 \
  --max-scale 10
```

`--min-scale 1` here keeps one instance always warm — useful if cold-start
latency matters more than idle cost for this particular app.

## Environment variables and secrets

```bash
# Plain config
ibmcloud ce app update hello-web --env LOG_LEVEL=info

# Reference a secret instead of a literal value
ibmcloud ce secret create --name db-creds \
  --from-literal PGHOST=<host> \
  --from-literal PGPASSWORD=<password>

ibmcloud ce app update hello-web --env-from-secret db-creds
```

Secrets referenced with `--env-from-secret` are injected as environment
variables at container start without ever appearing in `ibmcloud ce app
get` output or deployment logs — the same least-privilege instinct from
Level 1's IAM module applied to app configuration.

## Jobs: run-to-completion work

```bash
ibmcloud ce job create \
  --name nightly-report \
  --image icr.io/mastery-path/report-gen:1.0 \
  --cpu 1 --memory 2G \
  --array-indices 0-4 \
  --retry-limit 2
```

```bash
# Trigger one run of the job right now
ibmcloud ce jobrun submit --job nightly-report

ibmcloud ce jobrun get --name nightly-report-jobrun-...
# Status: Succeeded / Failed, plus per-array-index status
```

`--array-indices 0-4` runs five parallel instances of the same image, each
receiving a different `CE_SUBJOB_INDEX` environment variable — useful for
partitioning a batch (e.g., process shard `$CE_SUBJOB_INDEX` of 5) without
writing your own coordination logic.

## Scale-to-zero and concurrency, in practice

```bash
ibmcloud ce app update hello-web \
  --concurrency 10 \
  --min-scale 0 \
  --max-scale 5 \
  --scale-down-delay 300
```

- **`--concurrency 10`** — max simultaneous requests one instance handles
  before Code Engine starts a new instance. Too high and a slow request
  blocks others sharing that instance; too low and you scale out (and pay
  for) more instances than the traffic needs.
- **`--scale-down-delay 300`** — seconds an idle instance stays warm before
  scaling toward zero, trading a little idle cost for fewer cold starts on
  bursty-but-frequent traffic.

**Gotcha:** scale-to-zero means the *first* request after idle time pays
the cold-start cost (buildpack-built images especially can take a few
seconds to start). For latency-sensitive endpoints, `--min-scale 1` trades
that guaranteed idle cost for consistent response times — there's no
setting that gives you both zero idle cost and zero cold starts.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud ce project create --name <n>` | Create a Code Engine project |
| `ibmcloud ce app create --name <n> --build-source . --strategy buildpacks --port <p>` | Deploy an app, building from source |
| `ibmcloud ce app create --name <n> --image <img> --port <p>` | Deploy an app from a prebuilt image |
| `ibmcloud ce app update <n> --min-scale <n> --max-scale <n>` | Set scaling bounds |
| `ibmcloud ce secret create --name <n> --from-literal K=V` | Create a secret |
| `ibmcloud ce app update <n> --env-from-secret <secret>` | Inject a secret as env vars |
| `ibmcloud ce job create --name <n> --image <img> --array-indices <a-b>` | Create a parallel batch job |
| `ibmcloud ce jobrun submit --job <n>` | Trigger one run of a job |
| `ibmcloud ce app get --name <n>` | Show an app's URL and current scale |

## Exercise

Deploy `hello-web` from source with `--min-scale 0 --max-scale 3`, hit its
URL a few times with `curl`, then leave it idle for 10+ minutes and confirm
`ibmcloud ce app get --name hello-web` shows zero running instances before
the next request. Separately, create the `nightly-report` job with
`--array-indices 0-2`, submit a run, and use `ibmcloud ce jobrun get` to
confirm all three array indices reached `Succeeded` before deleting both
the app and the job.
