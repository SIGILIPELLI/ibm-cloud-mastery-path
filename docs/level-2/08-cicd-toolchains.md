# 08 · CI/CD with Toolchains

Every module so far has been "run a command, watch it happen." A real
team doesn't deploy that way — a `git push` should be what triggers a
build, a test run, and a deployment, with no human running `ibmcloud ce
app update` by hand each time. **IBM Cloud Continuous Delivery** provides
that: **toolchains** wire a Git repo, an issue tracker, and a **Tekton
pipeline** together, and the pipeline does the build/test/deploy work.

## Gotcha: no dedicated CLI plugin for this one

Unlike VPC (`ibmcloud is`), Kubernetes Service (`ibmcloud ks`), or Code
Engine (`ibmcloud ce`), Continuous Delivery has no equivalent
`ibmcloud toolchain` plugin for day-to-day use. Toolchains and Tekton
pipelines are provisioned through the console, the REST API, or — the
approach this module uses, consistent with Module 9's Schematics-first
style — **Terraform**, via the `ibm` provider's `ibm_cd_toolchain*` and
`ibm_cd_tekton_pipeline*` resources.

## Define the toolchain and a Git tool

```hcl
# toolchain.tf
resource "ibm_cd_toolchain" "mastery_toolchain" {
  name              = "mastery-path-toolchain"
  resource_group_id = data.ibm_resource_group.group.id
  description       = "CI/CD for the hello-web app"
}

resource "ibm_cd_toolchain_tool_hostedgit" "repo" {
  toolchain_id = ibm_cd_toolchain.mastery_toolchain.id
  parameters {
    repo_url   = "https://github.com/<you>/hello-web"
    type       = "link"
    has_issues = false
  }
}
```

`ibm_cd_toolchain_tool_hostedgit` links an already-existing repo into the
toolchain as a source; it doesn't create the GitHub repo itself — push
your app code (from Module 4's `hello-web`) there first.

## Attach a Tekton pipeline

```hcl
resource "ibm_cd_toolchain_tool_pipeline" "delivery_pipeline" {
  toolchain_id = ibm_cd_toolchain.mastery_toolchain.id
  parameters {
    name = "hello-web-pipeline"
    type = "tekton"
  }
}

resource "ibm_cd_tekton_pipeline" "hello_web" {
  pipeline_id = ibm_cd_toolchain_tool_pipeline.delivery_pipeline.tool_id
  worker {
    type = "public"
  }
}

# Point the pipeline at the .tekton/ folder in the repo for its definitions
resource "ibm_cd_tekton_pipeline_definition" "hello_web_def" {
  pipeline_id = ibm_cd_tekton_pipeline.hello_web.pipeline_id
  source {
    type = "git"
    properties {
      url    = "https://github.com/<you>/hello-web"
      branch = "main"
      path   = ".tekton"
    }
  }
}
```

## The pipeline definition itself: standard Tekton CRDs

```yaml
# .tekton/pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-web-pipeline
spec:
  params:
    - name: repository
    - name: revision
  tasks:
    - name: build-and-push
      taskRef:
        name: build-push-image
      params:
        - name: image
          value: "icr.io/mastery-path/hello-web:$(params.revision)"
    - name: deploy
      runAfter: ["build-and-push"]
      taskRef:
        name: deploy-to-code-engine
      params:
        - name: image
          value: "icr.io/mastery-path/hello-web:$(params.revision)"
```

```yaml
# .tekton/tasks/deploy-to-code-engine.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-to-code-engine
spec:
  params:
    - name: image
  steps:
    - name: deploy
      image: icr.io/codeengine/ce-cli
      script: |
        ibmcloud ce app update --name hello-web --image $(params.image)
```

Splitting `build-and-push` and `deploy` into separate Tekton `Task`s
mirrors Module 3's Container Registry push and Module 4's Code Engine
deploy — the pipeline is automating the exact two manual steps you already
ran by hand in earlier modules.

## Triggers: what starts a pipeline run

```hcl
resource "ibm_cd_tekton_pipeline_trigger" "on_push" {
  pipeline_id = ibm_cd_tekton_pipeline.hello_web.pipeline_id
  type        = "scm"
  name        = "push-to-main"
  event_listener = "listener"
  scm_source {
    url        = "https://github.com/<you>/hello-web"
    branch     = "main"
    pattern    = "main"
  }
  events {
    push = true
  }
}
```

A `scm` trigger fires on a Git webhook (push/PR), while a `manual` trigger
type gives you a button in the console (or a REST call) for on-demand
runs — keep at least one manual trigger during setup so you can test the
pipeline without pushing a commit every time.

## Apply and run

```bash
terraform init
terraform apply

# Trigger a run manually via the Continuous Delivery API while testing
TOKEN=$(ibmcloud iam oauth-tokens --output json | jq -r .iam_token)
curl -X POST \
  -H "Authorization: $TOKEN" \
  "https://api.us-south.devops.cloud.ibm.com/pipeline/v2/tekton_pipelines/<pipeline-id>/pipeline_runs" \
  -d '{"trigger_name": "push-to-main"}'
```

**Gotcha:** `terraform apply` on toolchain resources creates the wiring
(toolchain, tool integrations, pipeline, triggers) but does **not** run a
pipeline — the first real run only happens once a trigger fires (a push,
or the manual API call above). Don't expect a deployed app immediately
after `apply` finishes.

## Watching a run

```bash
curl -H "Authorization: $TOKEN" \
  "https://api.us-south.devops.cloud.ibm.com/pipeline/v2/tekton_pipelines/<pipeline-id>/pipeline_runs?limit=1"
```

Continuous Delivery's console gives a live log view of each `Task` step;
the REST API is the scriptable equivalent for status checks from another
pipeline or a notification script, but for actually reading build logs
day-to-day, the console is the practical tool — there's no `tkn`-style CLI
against IBM's managed service.

## How It Actually Works

- **A toolchain is a metadata container that groups tool integrations — it
  runs nothing itself.** `ibm_cd_toolchain` just creates an ID and a
  resource-group scope; every actual capability (the Git link, the
  pipeline, a Slack notifier) is a separate `ibm_cd_toolchain_tool_*`
  resource that registers itself against that toolchain ID. That
  separation is why deleting a toolchain in Terraform cascades to delete
  its tool integrations, but the underlying GitHub repo or Container
  Registry namespace those tools point at is untouched — the toolchain
  only holds references, not the resources themselves.
- **An SCM trigger works through a webhook IBM registers on your Git
  provider, not through Continuous Delivery polling the repo.** Creating
  `ibm_cd_tekton_pipeline_trigger` with `type = "scm"` calls out to GitHub
  (via the token in the hostedgit tool integration) to add a webhook
  pointed at an `event_listener` endpoint unique to that pipeline; a push
  matching the branch pattern makes GitHub POST the commit payload to that
  endpoint, which is what actually starts the `PipelineRun`. If the
  webhook is deleted on the GitHub side out-of-band, Terraform's state
  still shows the trigger configured but nothing will ever fire.
- **Each Tekton `Task` step runs in its own ephemeral container on the
  pipeline's worker, not in a persistent build VM** — `runAfter` ordering
  is enforced by the pipeline controller creating each step's pod only
  once its dependency's pod has exited successfully, and any files a task
  needs to pass to the next (like the built image reference) go through a
  workspace volume or params, not shared process state, since the
  containers never coexist. That's also why the `public` worker type has
  cold-start latency comparable to a fresh Code Engine instance — it's
  provisioning fresh compute per run, the same primitive Module 4 used for
  jobs.
- **`terraform apply` only ever changes control-plane wiring — a
  `PipelineRun` is a separate, transient Kubernetes-style custom resource
  created either by the webhook listener or your manual API call**, never
  by Terraform's own reconciliation loop. This is precisely why the state
  after `apply` has an idle pipeline: Terraform's job ends at "the trigger
  exists and is correctly configured," and something outside Terraform
  (the webhook, or you) has to be the thing that actually creates a run.

## Cheat sheet

| Resource / Command | Purpose |
|---|---|
| `ibm_cd_toolchain` | The container for a toolchain's tools |
| `ibm_cd_toolchain_tool_hostedgit` | Link an existing Git repo into the toolchain |
| `ibm_cd_toolchain_tool_pipeline` + `ibm_cd_tekton_pipeline` | Attach a Tekton pipeline |
| `ibm_cd_tekton_pipeline_definition` | Point the pipeline at a repo's `.tekton/` folder |
| `ibm_cd_tekton_pipeline_trigger` (`type = "scm"`) | Fire the pipeline on a Git push |
| `ibmcloud iam oauth-tokens --output json` | Get a bearer token for the Continuous Delivery API |
| `curl .../pipeline_runs -d '{"trigger_name": "..."}'` | Trigger a run manually (useful before wiring a real webhook) |

## Exercise

Push `hello-web` (from Module 4) to a Git repo with a `.tekton/` folder
containing the `Pipeline` and `Task` definitions above. Apply the
Terraform to stand up the toolchain, a `push-to-main` SCM trigger, and a
manual trigger. Fire a run using the manual `curl` call, watch it build
and push an image, then deploy over `--image` in Code Engine, and confirm
`ibmcloud ce app get --name hello-web` shows an updated revision timestamp
matching the run.
