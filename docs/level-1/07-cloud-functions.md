# 07 · Cloud Functions

**IBM Cloud Functions** is IBM's serverless platform, built on the
open-source **Apache OpenWhisk** project: you deploy small units of code
("actions"), and the platform runs them on demand, scales them
automatically, and only bills for actual execution time. This module covers
actions, triggers, rules, and sequences — the building blocks the capstone
project in Module 10 wires together into an API.

## Install the plugin and check your namespace

```bash
ibmcloud plugin install cloud-functions

ibmcloud fn namespace list
ibmcloud target --cf   # or: ibmcloud fn namespace target <namespace>
```

## Actions: the basic unit of code

```javascript
// hello.js
function main(params) {
    const name = params.name || "world";
    return { message: `Hello, ${name}!` };
}

exports.main = main;
```

```bash
ibmcloud fn action create hello hello.js

ibmcloud fn action invoke hello --result --param name "IBM Cloud"
# { "message": "Hello, IBM Cloud!" }

# Async invocation returns an activation ID immediately
ibmcloud fn action invoke hello --param name "IBM Cloud"
ibmcloud fn activation result <activation-id>
ibmcloud fn activation logs <activation-id>
```

## Web actions: exposing an action over plain HTTP

```bash
ibmcloud fn action update hello hello.js --web true

ibmcloud fn action get hello --url
# https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/default/hello
```

```bash
curl "https://<region>.functions.cloud.ibm.com/api/v1/web/<namespace>/default/hello?name=Curl"
```

A web action is the simplest way to give a frontend (like the COS static
site from Module 4) a callable backend endpoint with no server to manage.

## Triggers and rules: reacting to events

A **trigger** is a named event channel; a **rule** connects a trigger to an
action so firing the trigger invokes the action.

```bash
ibmcloud fn trigger create every-morning \
  --feed /whisk.system/alarms/alarm \
  --param cron "0 8 * * *"

ibmcloud fn rule create run-hello-daily every-morning hello

# Manually fire any trigger to test the wiring without waiting for the feed
ibmcloud fn trigger fire every-morning
```

## Sequences: chaining actions together

```bash
ibmcloud fn action create validate validate.js
ibmcloud fn action create process process.js
ibmcloud fn action create notify notify.js

# Runs validate -> process -> notify, passing each action's output as
# the next action's input
ibmcloud fn action create pipeline \
  --sequence validate,process,notify

ibmcloud fn action invoke pipeline --result --param path "/orders/42"
```

Sequences are billed as their sum of constituent action durations, but
deploy and invoke as a single logical action — useful for small pipelines
without standing up an orchestrator.

## Packages: grouping related actions

```bash
ibmcloud fn package create visits-api

ibmcloud fn action create visits-api/record record.js
ibmcloud fn action create visits-api/list list.js

ibmcloud fn action list
# /namespace/visits-api/record
# /namespace/visits-api/list
```

## How It Actually Works

- **Cloud Functions is IBM's managed layer over Apache OpenWhisk, and an
  "action" isn't a persistently running process — it's a container
  image invoked fresh (or reused from a warm pool) per request by
  OpenWhisk's invoker component**, which pulls your action's packaged
  code (a zip, or a base Docker image plus your code layered on) into a
  container, runs it against the incoming event payload, and returns
  the result — this container lifecycle is exactly the mechanism behind
  cold starts: the first invocation after idle time pays container
  startup cost, subsequent ones within the idle window reuse a warm
  container.
- **A "sequence" isn't a workflow engine — it's OpenWhisk chaining
  actions by piping each action's JSON output directly into the next
  action's input**, invoked one after another synchronously; there's no
  separate orchestration layer, which is why a sequence fails as a unit
  the moment any action in the chain throws, with no partial-completion
  state to recover.
- **Triggers and rules are OpenWhisk's publish/subscribe layer: a
  trigger is a named event channel, and a rule is a standing
  subscription binding that channel to an action** — firing a trigger
  (via an API call, a cron feed, or a Cloudant change feed) doesn't call
  your action directly; it publishes an event that OpenWhisk's rule
  engine matches against active rules and only then invokes the bound
  action, decoupling "what happened" from "what runs" so one trigger can
  fan out to multiple actions.
- **Billing is metered in GB-seconds — allocated memory multiplied by
  wall-clock execution time, rounded up to the nearest 100ms** — which
  is the actual reason over-provisioning an action's memory (even if it
  never uses it) directly increases cost per invocation, and why the
  free-tier allowance is expressed as a monthly GB-second budget rather
  than a request count.

## Cheat sheet

| Command | Purpose |
|---|---|
| `ibmcloud fn action create <name> <file.js>` | Deploy a new action |
| `ibmcloud fn action update <name> <file.js> --web true` | Update an action / expose it as a web action |
| `ibmcloud fn action invoke <name> --result --param k v` | Invoke synchronously |
| `ibmcloud fn activation logs <id>` | Fetch logs for an async invocation |
| `ibmcloud fn trigger create <name> --feed <feed> --param k v` | Create an event trigger |
| `ibmcloud fn rule create <name> <trigger> <action>` | Bind a trigger to an action |
| `ibmcloud fn action create <name> --sequence a,b,c` | Chain actions into a sequence |
| `ibmcloud fn package create <name>` | Group related actions |
| `ibmcloud fn action delete <name>` | Delete an action |

## Exercise

Deploy the `hello` action above as a web action and confirm you can `curl`
it from the command line without any IBM Cloud CLI involved. Then create an
`every-morning` alarm trigger and a rule connecting it to `hello`, fire the
trigger manually with `ibmcloud fn trigger fire`, and inspect the resulting
activation's logs to confirm the action actually ran.
