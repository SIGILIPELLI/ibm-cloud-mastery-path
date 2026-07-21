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
