# 01 — Developer Flow

> What a domain developer does on this platform, scenario by scenario. Each section maps to something demonstrable on screen given the prep from [`03-runbook.md`](03-runbook.md). The `Operating principle` callouts capture the design intent behind each scenario.

**Persona:** Alice, a software engineer on the BWR (Baby & Wedding Registry) domain team. New to the company; has never touched Harness, Argo CD, OPA, or Cosign before. The point is that she doesn't have to.

---

## Scenario 1 — Onboarding a brand-new service (Day 1)

Alice has been told to ship `bwr-web`, the front-end web service for the new registry product. She has never set up a CI/CD pipeline before and doesn't need to.

### What Alice does

1. Opens **Harness IDP catalog** in her browser
2. Clicks **+ Create** → picks **New Java Microservice** template
3. Fills the form:
   - Service name: `bwr-web`
   - Owner: `domain-bwr`
   - Description: `Baby & Wedding Registry web service`
   - Language: `Java 21`
   - Port: `8080`
4. Clicks **Create**

### What happens behind the scenes (~90 seconds)

| Step | Result |
|---|---|
| `fetch:template` × 2 | Renders skeleton + skeleton-pipeline-config locally with Alice's values |
| `publish:github` × 2 | Creates `acme-platform/team-bwr-web` and `acme-platform/team-bwr-pipeline-config` (or appends to the latter if BWR already had one) |
| `catalog:register` | Adds the new service to the Backstage catalog |
| `harness:create-service` | Registers the Harness service entity, links the auto-generated pipelines |

### What Alice sees at the end

- Two new repos in `acme-platform` org
- The bwr-web service appears in the IDP catalog with: links to repos, owner team, language, an active pipeline section
- Two pipelines (`bwr-web-ci`, `bwr-web-cd`) visible in `domain_bwr` Org → `bwr_web` Project, ready to run
- A README in the new repo explaining the developer flow

### What Alice did NOT have to do

- Choose a base image
- Configure SAST / container scanning / SBOM / signing
- Write a Dockerfile FROM line
- Write a Helm chart
- Configure deployment strategy
- Set up RBAC, network policy, security context
- Pick a registry, set image-pull secrets

All of that came pre-wired by the platform.

**Status note:** Scenario 1 describes what an IDP-driven onboarding looks like. The Backstage software template is authored and committed in `pe-idp-software-templates`. Live registration in the Harness IDP module is deferred pending IDP module availability confirmation on the target Harness account; once the module is enabled, the template can be registered and used as-is.

**Operating principle:**
> "From form-submit to first runnable pipeline: 90 seconds. From form-submit to production-ready devsecops surface: 0 seconds — it was always there."

---

## Scenario 2 — First feature (Day 2: code change → CI → CD to dev)

Alice clones `team-bwr-web`, writes some code, pushes a branch.

### What Alice does

```bash
git clone https://github.com/acme-platform/team-bwr-web.git
cd team-bwr-web
# Edit src/main/java/com/harness/app/Application.java — add a /products endpoint
git checkout -b add-products-endpoint
git add . && git commit -m "Add /products endpoint"
git push -u origin add-products-endpoint
gh pr create
```

### What happens automatically

The platform CI pipeline (`bwr-web-ci`, ~45-line thin wrapper around `account.java_maven_image_pipeline`) runs against the PR commit. Alice watches it in the Harness UI:

```
Stage: Test (account.java_maven_test_stage)
└── Java Maven Test                ✓  mvn test, JUnit reports surfaced

Stage: Build Image (account.image_build_stage)
├── Hadolint                       ✓  Dockerfile lint (with platform-allowed ignore rules)
├── Build Image                    ✓  Kaniko BuildAndPushDockerRegistry
│                                       BASE_IMAGE injected from config_parser:
│                                       acme-platform/base-java-21:latest
│                                       (multi-stage Dockerfile: Maven build runs INSIDE
│                                        the image build, jar copied into runtime layer)
├── Container Scan                 ✓  Trivy via Harness STO
└── Cosign Sign Image              ✓  Inline alpine + syft + cosign:
                                       syft → CycloneDX SBOM
                                       cosign sign        → image signature
                                       cosign attest      → SBOM attestation
```

Image lands at `docker.io/acme-platform/bwr-web:<commit-sha>` (and `:latest`), with both a Cosign signature and a CycloneDX SBOM attestation as separate OCI artifacts.

**Note**: SAST is not currently wired (deferred — requires Harness SCS license on the free tier we're running). The hadolint step provides Dockerfile-level lint as a free-tier alternative.

### Reviewer merges the PR

CI runs again on `main`; same green path. Then:

### CD runs (manual today; auto-trigger configurable)

The CD pipeline (`bwr-web-cd`, ~45-line thin wrapper around `account.standard_cd_pipeline`) runs with `env=dev`, `region=us-east-1`, `appImage=docker.io/acme-platform/bwr-web:<sha>`.

In the lab today, this is triggered manually so each pipeline step can be observed in isolation. In production, configure a Harness Pipeline Trigger so `bwr-web-ci` success auto-fires `bwr-web-cd` with the new image SHA.

```
Stage: GitOps Deploy (account.gitops_deploy_stage, type=CI)
├── Parse Config                ✓  Clones pe-pipeline-config + team-bwr-pipeline-config,
│                                  runs config_parser.sh (schema-validated layering),
│                                  writes /harness/config/resolved.yaml
├── Verify Image Signature      ✓  cosign verify + verify-attestation against
│                                  account.cosign_public_key. HARD-FAIL if missing.
├── Render Manifests            ✓  cosign-verify chart at acme-platform/microservice:1.0.0,
│                                  helm pull (OCI), helm template w/ resolved values
├── Commit Manifests            ✓  Push to team-bwr-gitops/bwr-web/manifests/dev/us-east-1/
├── GitOps Sync                 ✓  POST Harness GitOps API: trigger Application sync
├── Wait for Healthy            ✓  Poll Rollout CRD via kubectl (canary path) +
│                                  Harness GitOps API (rolling path) until Healthy
└── Smoke Test                  ✓  curl http://bwr-web.bwr-web-dev.svc.cluster.local:8080/healthz
```

Dev uses `deployStrategy: rolling` (no canary), so the wait_healthy step exits in seconds. Total CD run time for dev: ~30s.

### What Alice sees

- New commit in `team-bwr-gitops` on path `bwr-web/manifests/dev/us-east-1/`
- `kubectl get pods -n bwr-web-dev` shows her new image running
- `/products` endpoint live at the dev URL

### What Alice did NOT have to do

- Trigger any pipeline manually (other than the PR)
- Touch any deploy YAML
- Know that ArgoCD exists
- Verify the image signature
- Know that there's a separate gitops repo

**Operating principle:**
> "The whole devsecops surface — scanning, signing, attestation, signature verification, ArgoCD sync, smoke testing — runs invisibly. Alice watches her PR turn green and her change appear in dev. She doesn't know what STO is. She doesn't have to."

---

## Scenario 3 — Config-only change (the 12-factor superpower)

A few days later. The DBA tells Alice: "Bump the connection pool size to 20 on dev."

### What Alice does

```bash
git clone https://github.com/acme-platform/team-bwr-pipeline-config.git
cd team-bwr-pipeline-config
# Edit applications/bwr-web/environments/dev.yaml: BWR_DB_POOL_SIZE: "10" → "20"
git add . && git commit -m "Bump dev DB pool size to 20"
git push
gh pr create
```

She opens a PR against `team-bwr-pipeline-config` (NOT the app repo).

### What happens

Reviewer merges. The CD pipeline runs on `main` of `team-bwr-pipeline-config`:

```
Stage: Deploy (dev / us-east-1)
├── Resolve Config              ✓
├── Verify Supply Chain         ✓  (image is already signed; just verifies, no rebuild)
├── Render and Publish          ✓  (helm template → ConfigMap data changes; pod template's checksum/config annotation flips)
└── Sync and Confirm            ✓  (ArgoCD detects diff, rolling-restarts pods)
```

### What's notable

- **CI did NOT run.** No code changed.
- **No new image was built.** Container image bit-identical to before.
- **Pods rolling-restarted** because the `checksum/config` annotation on the pod template changed when the ConfigMap content changed.
- **Spring Boot picked up the new value** from `/app/config/application-override.yaml` on restart.

```bash
# Proof
kubectl describe deploy bwr-web -n bwr-web-dev | grep 'Image:'
# Image is the same SHA as before the config change.

kubectl get configmap bwr-web-config -n bwr-web-dev -o yaml | grep BWR_DB_POOL_SIZE
# Now: BWR_DB_POOL_SIZE: "20"
```

**Operating principle:**
> "12-factor done right: config lives in the environment, not the image. A config change is a CD-only run that produces a new ConfigMap and an annotation flip on the pod template. ArgoCD does the rolling restart. No CI, no rebuild, no dependency cache invalidation, no version-bump-and-redeploy ceremony. The image you scanned and signed yesterday is the image still running today."

---

## Scenario 4 — Promotion to prod (canary deploy)

Alice's feature is ready for prod. Tech lead clicks **Run Pipeline** on `bwr-web-cd` with `env=prod`.

### What's different vs. dev

The resolved config layer for prod:
- `app.replicas: 4` (vs 1 in dev)
- `app.deployStrategy: canary` (vs `rolling`)
- `app.canary.steps: [10%, pause 120s, 30%, pause 180s, 60%, pause 180s, 100%]` (BWR's slower phasing)
- `app.canary.analysisTemplate: success-rate`
- `scanning.containerScan.severityThreshold: MEDIUM` (tighter than account default of HIGH)

### What's rendered

The Helm chart picks `argoproj.io/v1alpha1 Rollout` (instead of `Deployment`) because `deployStrategy=canary`.

### What the developer sees

The pipeline pauses at the GitOps sync step while Argo Rollouts orchestrates the canary in-cluster:

```
T+0s     10% of pods are new image; analysis template starts measuring success rate
T+120s   New cohort still healthy; advance to 30%
T+300s   Still healthy; advance to 60%
T+480s   Advance to 100%; canary done
T+482s   Pipeline step `wait-healthy` returns
T+484s   Smoke test passes
T+490s   Pipeline complete
```

If the analysis fails at any step, **Argo Rollouts auto-rolls back**. The pipeline reports failure; Alice's previous version is restored without intervention.

**Operating principle:**
> "Canary phasing isn't a pipeline feature — it's a manifest feature. The pipeline writes a `Rollout` resource to Git; Argo Rollouts orchestrates the phasing in-cluster, runs the analysis template against Prometheus, and either advances or rolls back. The pipeline is the bridge between supply-chain checks and declarative state — it's not the deploy orchestrator. That's the right factoring."

---

## Scenario 5 — Trying to break the rules (the policy block)

What happens when a developer ignores platform conventions and tries to use a different base image?

### What Alice (or a malicious dev) does

```bash
git clone https://github.com/acme-platform/team-bwr-web.git
cd team-bwr-web
# Edit Dockerfile:
#   ARG BASE_IMAGE
# - FROM ${BASE_IMAGE}
# + FROM openjdk:21-slim
git checkout -b try-different-base
git add . && git commit -m "Use openjdk base instead"
git push -u origin try-different-base
gh pr create
```

### What happens

The CI pipeline starts. **At policy evaluation time, it hard-fails:**

```
ERROR: Pipeline run blocked by policy `require-platform-base-image`.

Step `build_image` uses non-platform base image "openjdk:21-slim"
(must start with "docker.io/acme-platform/base-").

Domain teams pick a language; the platform picks the base image. See
pe-base-images for the catalog.

Policy: pe-policies/harness/require-platform-base-image.rego
```

The pipeline UI shows a red gate at the OPA step. No image is built.

### What's also blocked (without Alice trying)

- Disabling SAST or container scan from a runtime input — `disable-scan-needs-justification` denies unless a Jira ticket reference is in `pipeline.tags.justification`
- Authoring a raw pipeline YAML in the BWR Org — `require-platform-pipeline-template` denies on save (allowed templates: `standard_image_build_pipeline`, `java_maven_image_pipeline`, `standard_cd_pipeline`)
- Deploying without verifying the image signature — `require-image-signature-verification` denies on run
- Setting `chartVersion: latest` (or any non-pinned semver) on a CD pipeline — `require-pinned-chart-version` denies on save. This forces an explicit chart upgrade PR; that PR is the audit trail for "when did prod start using chart 1.1.0?"

**Status note**: the policy set is committed and registered with Harness, but the `enabled` toggle is currently UI-only on free tier (the API endpoints exist but don't toggle the flag). Flip it in the Harness UI before relying on policy enforcement.

### What this demonstrates

Self-service is policy-bounded. Alice has full freedom inside the platform's contract; she cannot ergonomic-her-way around the contract. This is a hard wall, not a guideline — the pipeline doesn't start at all when a violation is detected.

**Operating principle:**
> "The platform makes the right thing easy — and the wrong thing impossible, not just discouraged. Four OPA policies, all hard-deny, all tested. A developer cannot scaffold or refactor their way past devsecops invariants. Override requires a Jira ticket reference, which is itself audited by the policy."

---

## Scenario 6 — Adding a second service to the team

Six months later. BWR team needs `bwr-api` (a backend API to complement the web service). What does this look like?

### What Alice does

Same as Scenario 1 — opens IDP, clicks Create, picks **New Java Microservice**, fills the form:
- Service name: `bwr-api`
- Owner: `domain-bwr`
- Language: `Java 21`

### What happens differently

The scaffolder detects that `team-bwr-pipeline-config` and `team-bwr-gitops` already exist. Instead of creating new ones, it:

- Creates `team-bwr-api` (new app code repo)
- Adds `applications/bwr-api/` subtree to the existing `team-bwr-pipeline-config`
- Adds `bwr-api/` subtree to the existing `team-bwr-gitops`

Both shared repos remain team-scoped. Adding the second app does not duplicate platform infrastructure.

### What this demonstrates

The team-scoped multi-app pattern lets a 1-team-N-app domain grow organically. RBAC stays at one place, audit history stays at one place, the IDP catalog naturally groups by team.

**Operating principle:**
> "Repos are team-scoped except for app code. Adding a new app to an existing team is one repo plus two subtree additions, not three repo creations. The convention scales: 5 teams × 4 apps = 25 repos, not 5×4×3 = 60."

---

## Summary — what the platform delivers

1. **Platform is invisible to the developer most of the time.** Devs see PRs, green pipelines, deployed services. They don't see scanning configs, signing keys, OPA, ArgoCD.

2. **Platform is unbreakable on the things that matter.** Supply chain, base image, scanning, signature verification — all hard-enforced.

3. **Config is in the environment, never in the image.** A config change is a 90-second CD run, no rebuild.

4. **Deployment strategy is declarative.** Canary phasing lives in Git as an Argo Rollouts resource, not in pipeline orchestration code.

5. **GitOps closes the audit loop.** Every state of every cluster is reproducible from a public Git history.

6. **Onboarding scales.** A new app on an existing team adds one repo and two subtrees. A new team adds three repos via the IDP scaffolder.

7. **Policy is testable.** OPA Rego with `opa test` (5 policies, 27/27 assertions passing) means the governance layer itself is software-engineered, not a wiki page.

8. **Templates compose cleanly.** Every domain pipeline is a ~45-line thin wrapper. Platform team owns 3 pipeline templates, 3 stage templates, ~12 step templates. Add a new language stack (Go, Python, Node) by adding ONE pipeline template + one test stage; reuse `image_build_stage` and `gitops_deploy_stage` unchanged.
