# 07 · Automation Pipelines at Scale (GitOps + Schematics)

Level 2 built a basic CI/CD toolchain. Level 3's Schematics module ran
Terraform through a managed service instead of a laptop. This module
connects the two into a **GitOps** pipeline: every infrastructure and
application change is proposed as a pull request, validated
automatically, and applied only after merge — nobody runs `apply` by
hand against production.

## The pipeline shape

```text
git push (feature branch)
   │
   ▼
CI: lint + terraform validate + policy check   (Module 08's precursor)
   │  PR opened
   ▼
CI: terraform plan (Schematics) posted as PR comment
   │  human review + approval
   ▼
merge to main
   │
   ▼
CD: terraform apply (Schematics)  ──▶  OpenShift GitOps reconciles app manifests
```

Two separate concerns run through this one pipeline shape: infrastructure
changes (Terraform via Schematics) and application deployment changes
(Kubernetes manifests via GitOps) — keep them in separate repos or at
least separate directories so an app deploy never accidentally re-applies
unrelated infrastructure.

## CI pipeline: validate before merge

```yaml
# .github/workflows/infra-ci.yml (works equally as a Tekton pipeline on IBM Cloud)
name: infra-ci
on: pull_request

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform -chdir=infra/environments/prod init -backend=false
      - run: terraform -chdir=infra/environments/prod validate
      - name: Schematics plan
        run: |
          ibmcloud login --apikey "$IBMCLOUD_API_KEY" -r us-south
          WORKSPACE_ID=$(ibmcloud schematics workspace list --output json | \
            jq -r '.workspaces[] | select(.name=="orders-platform-prod") | .id')
          ACT_ID=$(ibmcloud schematics plan --id "$WORKSPACE_ID" --output json | jq -r .activityid)
          sleep 30
          ibmcloud schematics logs --id "$WORKSPACE_ID" --act-id "$ACT_ID" > plan_output.txt
          cat plan_output.txt
```

```text
Plan: 2 to add, 1 to change, 0 to destroy.
```

Posting the plan output as a PR comment (via a follow-up step calling the
GitHub API) is what makes review meaningful — a reviewer approving a
Terraform PR without seeing the plan is approving blind.

## CD pipeline: apply only after merge, with a gate

```yaml
# .github/workflows/infra-cd.yml
name: infra-cd
on:
  push:
    branches: [main]
    paths: ['infra/environments/prod/**']

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production   # requires manual approval in GitHub environment protection
    steps:
      - uses: actions/checkout@v4
      - run: |
          ibmcloud login --apikey "$IBMCLOUD_API_KEY" -r us-south
          WORKSPACE_ID=$(ibmcloud schematics workspace list --output json | \
            jq -r '.workspaces[] | select(.name=="orders-platform-prod") | .id')
          ibmcloud schematics apply --id "$WORKSPACE_ID" --force
```

`environment: production` with GitHub's environment protection rules adds
a required-reviewer gate on top of the merge itself — a second human
checkpoint immediately before `apply`, separate from the PR review.

## OpenShift GitOps (Argo CD) for application manifests

```bash
oc apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-frontend
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/example-org/orders-platform-manifests
    targetRevision: main
    path: apps/orders-frontend/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: orders-frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

```text
application.argoproj.io/orders-frontend created
```

`selfHeal: true` means a manual `oc edit` against a live resource gets
reverted automatically back to what's in Git — the enforcement mechanism
that makes "Git is the source of truth" actually true, not just aspirational.

## Promote across environments with overlays, not copy-paste

```text
apps/orders-frontend/
  base/
    deployment.yaml
    service.yaml
  overlays/
    dev/
      kustomization.yaml     # replicas: 1, small resource requests
    staging/
      kustomization.yaml     # replicas: 2
    prod/
      kustomization.yaml     # replicas: 5, HPA enabled
```

```bash
kubectl kustomize apps/orders-frontend/overlays/prod
```

Promotion between environments becomes "change which overlay a
GitOps `Application` points at" (or bump an image tag in the overlay) —
never re-authoring manifests per environment, which is how environments
drift.

## Rollback: revert the commit, not the cluster

```bash
git revert <bad-commit-sha>
git push origin main
```

Argo CD's automated sync picks up the revert and reconciles the cluster
back to the prior state within its sync interval — a cluster-side
`kubectl rollout undo` bypasses Git and immediately creates drift between
what's deployed and what Git says should be deployed, defeating the whole
GitOps guarantee.

## Terraform for the Schematics workspace itself, as code

```hcl
resource "ibm_schematics_workspace" "orders_prod" {
  name            = "orders-platform-prod"
  location        = "us-south"
  resource_group  = data.ibm_resource_group.mastery_path.id
  template_repo {
    url    = "https://github.com/example-org/orders-platform-infra"
    branch = "main"
  }
  template_type = ["terraform_v1.7"]
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **`selfHeal` fighting a legitimate emergency fix**: during an active
  incident, a hotfix applied directly to the cluster gets reverted by
  Argo CD's self-heal unless the `Application` is paused first — know the
  pause command (`argocd app set orders-frontend --sync-policy none`)
  before you need it under pressure.
- **CI plan and CD apply can drift** if enough time passes between plan
  approval and merge (someone else's PR merges first) — re-plan
  immediately before apply in the CD job, don't trust a plan from an hour
  ago against a workspace that may have moved.
- **Secrets in pipeline YAML**: never put `IBMCLOUD_API_KEY` in plaintext
  in a workflow file — use the CI platform's secret store (GitHub Actions
  secrets, Tekton `Secret` resources) exclusively.
- **Kustomize overlay drift between environments** is still possible even
  with the base/overlay pattern if `staging` and `prod` overlays diverge
  in ways nobody notices (e.g., different resource limits creeping apart
  over many small PRs) — periodically diff overlays against each other,
  not just against the base.

## Cheat sheet

| Task | Command |
|---|---|
| Trigger a Schematics plan from CI | `ibmcloud schematics plan --id <workspace-id>` |
| Apply from CD (with force for automation) | `ibmcloud schematics apply --id <workspace-id> --force` |
| Create an Argo CD Application | `oc apply -f application.yaml` (kind: Application) |
| Render a kustomize overlay locally | `kubectl kustomize <overlay-path>` |
| Pause Argo CD self-heal for a hotfix | `argocd app set <app> --sync-policy none` |
| Revert a bad deploy | `git revert <sha> && git push` |

## Exercise

1. Write a CI workflow (YAML) that runs `terraform validate` and a
   Schematics plan on every pull request touching an `infra/` path.
2. Write a separate CD workflow gated by a required-reviewer environment
   that applies only on merge to `main`.
3. Define an Argo CD `Application` resource with `selfHeal: true`
   pointing at a kustomize overlay, and set up `dev`/`staging`/`prod`
   overlays for one deployment.
4. Explain, in prose, why `git revert` is the correct rollback mechanism
   in this pipeline and what breaks if you instead run `kubectl rollout
   undo` directly against the cluster.
