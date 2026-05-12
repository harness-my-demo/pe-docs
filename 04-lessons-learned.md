# 04 — Lessons Learned and Platform Improvements

> What we got right, what we'd change, and the concrete next-quarter roadmap. The two large items are (1) template testing — because every template is consumed by N domain pipelines and a silent break propagates to all of them — and (2) Terraform-managed template promotion, so changes are applied through a controlled, reproducible flow rather than the Harness UI's reconcile path.

**Last updated:** 2026-05-12

---

## 1. The core operational problem

Templates are a **product with consumers**. A change to a single step template affects every domain pipeline that references it. Two failure modes follow:

1. **Silent breakage.** A leaf change (renamed env var, removed default, changed allowed values) doesn't fail the template's own validation but breaks the next pipeline run on every consumer. The blast radius is fan-out × deploy frequency.
2. **Reconcile drift.** Harness's UI/Git Experience reconcile applies template-input shape changes to dependent pipelines at the time of *next read*, not the time of authoring. So a save in the platform repo doesn't surface as a domain-pipeline failure until somebody clicks Run weeks later — by which time the cause and the symptom are far apart in time.

Both failures share a root cause: **template changes are not under test, and are not gated by anything before they reach consumers.**

---

## 2. Lessons learned (this build)

These are catalogued in the platform team's internal operations register; the load-bearing ones for future improvement work:

### 2.1 Reconcile is fragile in the UI/Git path

Three distinct reconcile gotchas hit during the build, all of which required experimentation to identify:

1. **`<+input>` at leaf level bubbles to runtime** even when an upstream stage's `templateInputs` ostensibly provides the value. Adding `.default(...)` to the leaf silently disables the upstream override entirely. The only reliable pattern: leaf step templates read `<+stage.variables.X>` directly; no `<+input>` at the bottom of a chain.

2. **`<+input>.default("X")` preserves literal quotes** when resolved into shell env vars. Every Run step touching such a value must POSIX-strip them at the top of the command (`${VAR#\"}; ${VAR%\"}`). The bash form `${VAR//\"/}` is silently rejected by Alpine `ash`.

3. **Variables with `required: true` reject empty-string values** even when `.default("")` is set. Mark optional-with-default variables `required: false`.

The fix patterns are simple once known. The cost was discovery time. **A test pipeline that exercises every published template against a known-good fixture set would have surfaced all three within minutes of authoring.**

### 2.2 Free-tier API surface has gaps that force UI workflows

- Harness Policy Engine: policies and policy sets are POST/PATCH-able via `/pm/api/v1/...`, but the `enabled` flag cannot be toggled by API (all proposed payloads return 204 without effect; sub-routes return 405). UI flip is required.
- Argo Project mapping: `/gitops/api/v1/.../appprojectmappings` and variants return 501 Not Implemented. UI "Import Projects" wizard is required.
- Repository creation on the GitOps API has a non-obvious URL shape (`/gitops/api/v1/agents/<agent>/repositories?accountIdentifier=...&identifier=...`) — body-only requests are routed to the *list* handler instead.

None of these block automation in production tiers (Terraform provider handles them), but on free tier they pin the human in the loop.

### 2.3 ArgoCD doesn't natively understand Argo Rollouts CRDs

The `argocd-cm` Lua resource customization for `argoproj.io/Rollout` is the canonical fix, but didn't load reliably on the bundled ArgoCD in the Harness GitOps Agent across controller restarts. Workaround: `wait_healthy` polls the Rollout CRD directly via kubectl. Acceptable for the lab; in production, install Argo Rollouts via its own controller and treat the Rollout CRD as the source of truth for canary health.

### 2.4 Cosign hostname strictness

Cosign signature paths are bound to the **exact registry hostname** used at sign time. Image-push tools default to `index.docker.io`; Helm OCI push uses `registry-1.docker.io`. Verification has to log in to the same hostname the signature was written to. Documented per-step; one line of operational doc would have saved an hour of debugging.

### 2.5 SecretFile vs SecretText is not interchangeable

Updating a `SecretFile` secret via the JSON `PUT /v2/secrets/<id>` endpoint silently fails with "Cannot change type". For file secrets, use `PUT /ng/api/v2/secrets/files/<id>` with `multipart/form-data`. The diagnostic signal — verify failing with `unknown Public key PEM file type: ENCRYPTED SIGSTORE PRIVATE KEY` — is the indicator that the wrong file body was uploaded under that identifier.

### 2.6 Initial Rollout deploy skips canary phasing

Argo Rollouts treats the first revision as a straight scale-up — no setWeight/pause cycles, no AnalysisRun. Canary semantics only engage on revisions ≥ 2. For verification: do an initial deploy, then a second deploy with a different `appImage` tag. Pin to commit-SHA tags so the manifest diffs.

---

## 3. Proposed improvement: a template SDLC

The platform engineering team needs to treat templates as software:

```
authoring repo (pe-pipeline-templates)
        │
        │ PR
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                  TEMPLATE TEST PIPELINE                          │
│                  (pe-pipeline-templates-ci)                      │
│                                                                  │
│  Stage 1: Static checks                                          │
│    - yamllint                                                    │
│    - opa fmt / opa test on policies                              │
│    - schema-validate templateInputs/variables shape              │
│    - Harness `template-validate` API endpoint                    │
│                                                                  │
│  Stage 2: Contract tests                                         │
│    - For each published template, render the resolved YAML       │
│      against a known-good test fixture                           │
│    - Diff against the recorded golden output                     │
│    - Fail if the diff is unexpected                              │
│                                                                  │
│  Stage 3: Integration tests                                      │
│    - Deploy the modified template to a STAGING Harness project   │
│      (separate from production)                                  │
│    - Run a thin-wrapper test pipeline in staging that exercises  │
│      every step                                                  │
│    - Pass = template is safe for promotion                       │
│                                                                  │
│  Stage 4: OPA evaluation                                         │
│    - Run the test pipeline definition through every active OPA   │
│      policy in dry-run mode                                      │
│    - Catches policy regressions before they reach domain teams   │
└─────────────────────────────────────────────────────────────────┘
        │
        │ all green
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                  TERRAFORM PROMOTION                             │
│                                                                  │
│  - PR merge to main triggers `terraform plan`                    │
│  - Plan is posted as a PR comment for review                     │
│  - On approval, `terraform apply` writes templates to            │
│    the production Harness account                                │
│  - Templates are bumped to a new versionLabel (semver)           │
│  - Old versionLabel remains for pinned consumers                 │
└─────────────────────────────────────────────────────────────────┘
        │
        │ available
        ▼
   domain pipelines pin a versionLabel:
     templateRef: account.java_maven_image_pipeline
     versionLabel: "1.4.0"     ← explicit; never "latest"
```

### 3.1 Why pinning solves silent breakage

Today, every domain pipeline references `versionLabel: "1.0"`. A breaking change to that label affects everyone at once. After this change:

- The platform publishes `versionLabel: "1.4.0"` for the new template.
- Existing domain pipelines stay on `1.3.x` (or whichever they pinned).
- The platform team announces the new version; domain teams move at their pace.
- If a critical fix is needed, it's published as `1.3.1` (patch within their pinned minor); domain teams adopt without action via patch-roll-forward policy.

Semver convention to enforce:
- **MAJOR**: backwards-incompatible (renamed/removed variable, type change, removed step). Coordinated migration.
- **MINOR**: backwards-compatible additions (new optional variable, new step appended). Domain teams can adopt freely.
- **PATCH**: bug fixes, documentation. Domain teams roll-forward without thinking.

A new OPA policy enforces the pinning syntax: `require-pinned-template-version` (denies `"latest"` or missing versionLabel on platform templateRefs). Same pattern as `require-pinned-chart-version` already deployed.

### 3.2 Where Terraform fits

Harness's Terraform provider has resources for `harness_platform_template`, `harness_platform_pipeline`, `harness_platform_policy`, `harness_platform_policy_set`, `harness_platform_gitops_*`. Switching to Terraform-managed templates achieves:

- **Atomic apply.** All templates in a release land together; no intermediate state where pipeline-template references a stage-template version that doesn't exist yet.
- **Plan visibility.** PR reviewer sees the exact set of resource changes before merge.
- **Bypasses the UI reconcile.** No `templateInputs` strip surprises.
- **Toggle automation.** `enabled` on policy sets is settable via Terraform (the resource accepts the field directly).
- **Rollback story.** `git revert <commit> && terraform apply` rolls back the template release.

The migration cost is one-time: read each existing template's YAML, wrap in `harness_platform_template` resources, parameterize. Subsequent template work is regular pull-request workflow.

### 3.3 Test fixtures

The test pipeline (Stage 2 above) needs fixtures. Pattern:

```
pe-pipeline-templates/
├── templates/
│   ├── steps/        (production templates)
│   ├── stages/
│   └── pipelines/
└── tests/
    └── fixtures/
        ├── domain-good/
        │   ├── application.yaml          (valid domain config)
        │   ├── Dockerfile                 (multi-stage, uses ${BASE_IMAGE})
        │   └── expected/
        │       ├── ci-resolved.yaml      (golden output of CI pipeline render)
        │       └── cd-resolved.yaml      (golden output of CD pipeline render)
        ├── domain-bad-no-base-image/
        │   └── ...                       (should fail OPA require-platform-base-image)
        └── domain-bad-floating-chart/
            └── ...                       (should fail OPA require-pinned-chart-version)
```

The test pipeline renders the templates against each fixture and compares to the `expected/` golden file. Golden-file diff is the contract. To update a golden, you edit it in the PR — the human review of that diff IS the test-result review.

### 3.4 Backwards compatibility check

A non-trivial migration. Recommended order:

1. **Adopt Terraform** for new templates only. Keep existing UI/Git Experience templates in place.
2. **Build the test pipeline** as a standalone CI job in the `pe-pipeline-templates` repo. Initially: lint + golden-file diff. Stage 3 (integration test against staging Harness) requires a separate staging Harness account, which is the largest blocker.
3. **Migrate existing templates** to Terraform one at a time. Each migration is a no-op refactor — Terraform imports the existing template's state, then subsequent edits go through the new flow.
4. **Cut over the policy set** to Terraform-managed, which auto-resolves the `enabled` toggle issue.
5. **Introduce semver versioning** for new template releases. Bump existing templates to `"1.0.0"` (from the current `"1.0"`) at first Terraform-managed release.

Estimated effort: 2 sprints for the test pipeline + Terraform scaffolding; ongoing migration as templates are touched.

---

## 4. Other improvements (smaller items)

### 4.1 Domain GitOps repo: subtree-per-app convention

Today's gitops repo (`team-bwr-gitops`) holds one app. The dev-flow doc describes a per-team multi-app convention. Codify this with:

- Naming: `team-<team>-gitops`, not `team-<app>-gitops`.
- Layout: `<app>/manifests/<env>/<region>/` for each app under the team.
- Scaffolder behavior: detect if the team's gitops repo exists; create-on-first-app, subtree-append-on-subsequent.

The IDP scaffolder (in `pe-idp-software-templates`) currently creates a per-service pipeline-config repo (`team-${serviceName}-pipeline-config`). It should detect and append to an existing `team-${owner}-pipeline-config` instead.

### 4.2 Harness Pipeline Trigger: CI success → CD

Today `bwr-web-cd` is triggered manually so each step can be observed. In production, wire:

- **GitHub webhook → bwr-web-ci** (push to main on team-bwr-web). Standard Harness webhook trigger.
- **bwr-web-ci success → bwr-web-cd** (with `appImage` filled from the upstream codebase commit SHA). Harness Pipeline-event trigger.

For environments other than dev, gate behind manual approval — auto-deploy to dev, manual promote to staging/prod. Approval step lives in the CD pipeline template, gated by env value.

### 4.3 AnalysisTemplate: Prometheus-backed

The lab's `success-rate` AnalysisTemplate is a synthetic Job that always succeeds. Production version queries Prometheus:

```yaml
metrics:
  - name: success-rate
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[2m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

Requires Prometheus + the app exposing `/metrics`. Out of scope for the lab; documented as a swap-in.

### 4.4 SAST and SCS

SAST is not currently wired (the SCS license isn't on free tier). Pipeline template structure already has the slot — a `sast_scan` step would sit between `java_maven_test` and `build_image` in `java_maven_image_pipeline`. Adopt when the license tier supports it. Hadolint fills the gap at the Dockerfile layer for now.

### 4.5 Multi-region promotion

Today the lab runs `region=us-east-1` everywhere. Multi-region promotion would:

- Add per-region GitOps Applications: `bwr-web-prod-us-east-1`, `bwr-web-prod-eu-west-1`, etc.
- A "promote to all regions" pipeline that fans out CD runs to each region with the same `appImage`.
- Hierarchical config layers a per-region overlay (`regions/eu-west-1.yaml`) on top of `environments/prod.yaml`.

The schema already supports this; the config-resolver already merges in region overlay order. What's missing is the fan-out pipeline and the per-region ArgoCD Applications.

### 4.6 Cost allocation (CCM)

Tag every workload at deploy time with `cost-center: <team>`. Harness CCM aggregates by tag. Already supported by the chart (`labels.app.cost-center` is in `exposed-values.yaml`); not yet enforced. Add `require-cost-center-label` to the OPA policy set.

### 4.7 Zero-trust delegate

Per-environment delegates (`delegate-dev`, `delegate-staging`, `delegate-prod`) instead of one `lab-delegate` serving everything. Prevents a compromised dev delegate from writing to prod. Stage templates reference delegate by environment-derived selector.

---

## 5. Roadmap (rough)

| Quarter | Item |
|---|---|
| Q+1 | Template test pipeline (lint + golden-file). Set up staging Harness account. |
| Q+1 | Migrate `pe-pipeline-templates` to Terraform-managed. Cut over policy set. |
| Q+2 | Introduce semver versioning + `require-pinned-template-version` OPA policy. |
| Q+2 | CI→CD auto-trigger; manual approval gate for staging/prod env. |
| Q+3 | Prometheus-backed AnalysisTemplate. Multi-region fan-out pipeline. |
| Q+3 | IDP scaffolder updates for team-scoped pipeline-config and gitops repos. |
| Q+4 | CCM cost-center tagging and policy. Zero-trust delegate model. |

---

## 6. What this catalog is for

Two audiences:

- **Future platform engineers joining this team:** the lessons in §2 save you from re-discovering the same reconcile / cosign / RBAC gotchas. Read §2 before authoring templates.
- **Anyone reviewing this lab:** §3 is the answer to "how would you operationalize this?". The lab is a working implementation; this doc is the gap analysis between "working implementation" and "platform you'd safely run for 10+ domain teams."
