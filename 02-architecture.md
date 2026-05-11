# 02 — Architecture

> Comprehensive architecture reference for the Harness Platform Engineering lab. Each section is a self-contained explanation with a diagram and operational rationale.

**Last updated:** 2026-05-11
**Status:** All tiers live and verified end-to-end. Thin-wrapper template chain working; GitOps-native CD wired through Harness Service/Environment/GitOps Cluster/Application objects; Argo Rollouts canary live in prod.

---

## 0. The big picture in one diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              HARNESS PLATFORM (SaaS)                                │
│                                                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐  ┌─────────┐ │
│  │ Connectors  │  │  Templates   │  │  Pipelines  │  │ Policy       │  │ GitOps  │ │
│  │ Secrets     │  │  (account    │  │  (project   │  │ Engine       │  │ Service │ │
│  │             │  │   scope)     │  │   scope)    │  │  (OPA)       │  │  (Argo) │ │
│  └─────────────┘  └──────────────┘  └─────────────┘  └──────────────┘  └─────────┘ │
└──┬──────────────────────────────┬──────────────────────────────────┬───────────────┘
   │ Git Experience                │ delegate-task gRPC               │ agent gRPC
   │ (read templates from Git)     │                                  │
   ▼                              ▼                                  ▼
┌──────────────────────┐  ┌──────────────────────────────────────────────────────────┐
│  GITHUB ORG          │  │            kind cluster `harness-lab` (Docker Desktop)   │
│  acme-platform     │  │                                                          │
│                      │  │  ┌────────────────────┐    ┌──────────────────────────┐  │
│  Platform repos:     │  │  │ harness-delegate-ng│    │ harness-gitops-ns        │  │
│  - pe-pipeline-      │  │  │   lab-delegate     │    │   gitops-agent (Argo CD) │  │
│    config            │  │  │   (CI orchestrator)│    │   argocd-application-    │  │
│  - pe-pipeline-      │  │  └─────────┬──────────┘    │     controller           │  │
│    templates         │  │            │ schedules     │   argocd-repo-server     │  │
│  - pe-base-images    │  │            ▼ build pods    │   argocd-redis           │  │
│  - pe-policies       │  │  ┌────────────────────┐    └──────────┬───────────────┘  │
│  - pe-idp-software-  │  │  │ harness-build      │               │ reconciles       │
│    templates         │  │  │   <build pods, 1   │               ▼                  │
│                      │  │  │    per pipeline    │    ┌──────────────────────────┐  │
│  Domain repos:       │  │  │    run, ephemeral> │    │ bwr-web-dev              │  │
│  - team-bwr-web      │  │  │                    │    │   bwr-web pod (1/1)      │  │
│  - team-bwr-         │  │  │   Maven build      │    │   bwr-web ClusterIP svc  │  │
│    pipeline-config   │  │  │   docker build     │    │                          │  │
│  - team-bwr-gitops   │  │  │   trivy scan       │    │                          │  │
│                      │  │  │   cosign sign      │    │                          │  │
└──────────┬───────────┘  │  └────────────────────┘    └──────────────────────────┘  │
           │ source-of-    │  ┌────────────────────┐                                  │
           │ truth         │  │ ingress-nginx      │                                  │
           ▼               │  │   (port 8080:80)   │                                  │
                           │  └────────────────────┘                                  │
                           └──────────────────────────────────────────────────────────┘
                                            │
                                            │ docker push (signed image)
                                            ▼
                                  ┌──────────────────────────┐
                                  │  Docker Hub              │
                                  │  acme-platform/base-java-21   │ ← signed by platform team
                                  │  acme-platform/bwr-web        │ ← signed (FROM base above)
                                  └──────────────────────────┘
```

---

## 1. Tenant topology — the three identity domains

Three external identity boundaries we operate across:

| Domain | Identifier | Holds |
|---|---|---|
| **Harness account** | `<HARNESS_ACCOUNT_ID>` | Connectors, secrets, templates, pipelines, policies |
| **GitHub org** | `acme-platform` | 8 repos (5 platform `pe-*`, 3 domain `team-bwr-*`) |
| **Docker Hub namespace** | `acme-platform` | Container images: `base-java-21`, `bwr-web` |

The platform engineering team writes everything in the `pe-*` repos and Harness Account scope. Domain teams write only in `team-bwr-*` repos and Harness `bwr_web` project. RBAC in Harness enforces the boundary.

---

## 2. Repo map with ownership

### 8 repos, deliberately split by *who edits them* and *how often*

```
                    ┌──────────────────────────┐
                    │   PLATFORM ENGINEERING   │
                    │   (writes; ~1× per       │
                    │    sprint or quarter)    │
                    └────────────┬─────────────┘
                                 │ owns
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│ pe-pipeline-    │  │ pe-pipeline-        │  │ pe-policies         │
│   config        │  │   templates         │  │  (OPA Rego)         │
│                 │  │                     │  │                     │
│ Hiera-style     │  │ 18 Harness          │  │ 4 policies          │
│ hierarchy +     │  │ templates +         │  │ + 18 unit tests     │
│ schema +        │  │ Helm umbrella       │  │ Attached to all     │
│ config_parser   │  │ chart for K8s       │  │ pipeline events     │
└─────────────────┘  └─────────────────────┘  └─────────────────────┘

        ▼                        ▼
┌─────────────────────┐  ┌──────────────────────┐
│ pe-base-images      │  │ pe-idp-software-     │
│                     │  │   templates          │
│ Hardened Dockerfiles│  │                      │
│ Java/Node/Python    │  │ Backstage scaffolder │
│ Built+signed nightly│  │ "New Microservice"   │
└─────────────────────┘  └──────────────────────┘


                    ┌──────────────────────────┐
                    │   DOMAIN TEAM (BWR)      │
                    │   (writes; daily)        │
                    └────────────┬─────────────┘
                                 │ owns
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌─────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐
│ team-bwr-web    │  │ team-bwr-           │  │ team-bwr-gitops      │
│                 │  │   pipeline-config   │  │                      │
│ App source code │  │                     │  │ Auto-written by CD   │
│ + Dockerfile    │  │ applications/       │  │ pipeline. Read-only  │
│ + .harness/     │  │   bwr-web/          │  │ to humans (ArgoCD    │
│   pipeline.yaml │  │   {application,     │  │ source of truth).    │
│ + catalog-info  │  │    environments/    │  │                      │
│                 │  │    *.yaml}          │  │                      │
└─────────────────┘  └─────────────────────┘  └──────────────────────┘
```

### Why 8 repos and not 1 monorepo?

| Property | Mono-repo problem | What 8-repo split gives us |
|---|---|---|
| Ownership / access | Same access list for everything; platform team becomes bottleneck | Platform team can lock `pe-*` to their approval; domain team has free rein on `team-bwr-*` |
| Release cadence | Templates + app code share git history | Domain code changes daily; platform templates change rarely; gitops repo is auto-written |
| Trigger isolation | One commit triggers everything | A platform template change doesn't rebuild domain images; a config change doesn't trigger CI |
| Audit clarity | "Who deployed what when?" requires log archaeology | `team-bwr-gitops` commit history IS the deploy log |
| Bootstrap-ability | Recover after disaster requires the whole monorepo | Each repo is independently restorable |

---

## 3. Cluster topology

```
┌────────────────────────────────────────────────────────────────────────┐
│                  Docker Desktop (12 GB RAM allocation)                 │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │            kind cluster `harness-lab` — single node              │  │
│  │                                                                  │  │
│  │  ┌──────────────────────┐  ┌──────────────────────────────────┐  │  │
│  │  │ harness-delegate-ng  │  │ harness-gitops-ns                │  │  │
│  │  │                      │  │                                  │  │  │
│  │  │ lab-delegate (1/1)   │  │ gitops-agent (1/1)              │  │  │
│  │  │   - executes K8s-    │  │ argocd-application-controller    │  │  │
│  │  │     based pipeline   │  │   (StatefulSet, 1/1)             │  │  │
│  │  │     steps            │  │ argocd-applicationset-controller │  │  │
│  │  │   - validates K8s    │  │ argocd-repo-server (2/2)         │  │  │
│  │  │     connector via    │  │ argocd-redis                     │  │  │
│  │  │     InheritFrom-     │  │                                  │  │  │
│  │  │     Delegate         │  │ Reconciles team-bwr-gitops →     │  │  │
│  │  └──────────────────────┘  │ bwr-web-dev namespace            │  │  │
│  │                            └──────────────────────────────────┘  │  │
│  │                                                                  │  │
│  │  ┌──────────────────────┐  ┌──────────────────────────────────┐  │  │
│  │  │ harness-build        │  │ bwr-web-dev                      │  │  │
│  │  │                      │  │                                  │  │  │
│  │  │ Build pods           │  │ bwr-web pod (1/1)                │  │  │
│  │  │   one per pipeline   │  │   image: docker.io/acme-platform/     │  │  │
│  │  │   run, ephemeral.    │  │          bwr-web:latest          │  │  │
│  │  │   Each pod has       │  │   Cosign-signed; verifiable.     │  │  │
│  │  │   N+1 containers:    │  │                                  │  │  │
│  │  │   - lite-engine      │  │ bwr-web ClusterIP service        │  │  │
│  │  │     (Harness CI)     │  │                                  │  │  │
│  │  │   - one per step     │  │                                  │  │  │
│  │  │     (each in its     │  │                                  │  │  │
│  │  │     own image)       │  │                                  │  │  │
│  │  └──────────────────────┘  └──────────────────────────────────┘  │  │
│  │                                                                  │  │
│  │  ┌──────────────────────┐                                        │  │
│  │  │ ingress-nginx        │  port :80 in cluster                  │  │
│  │  │ controller (1/1)     │  → mapped to host :8080                │  │
│  │  └──────────────────────┘                                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### Why kind?

- **Throwaway**: `kind delete cluster --name harness-lab` and the lab is gone.
- **Multi-node-capable**: We use single-node for the lab; a multi-node cluster is a config-only change.
- **Native Docker Desktop integration**: No VM driver indirection.
- **Ingress port mapping**: kind config exposes :80 in cluster as :8080 on host, so we could `curl http://localhost:8080` if we wanted to.

### What happens when memory is tight

8 GB Docker Desktop allocation was insufficient (argocd-application-controller couldn't schedule). 12 GB clears it. **Architectural lesson**: the kind cluster is a shared-resource pool — when we add components (delegate + gitops agent + ingress + argo-rollouts + ephemeral build pods), we have to budget. Ratio that worked: ~6 GB for system + Harness components, ~3 GB for the largest concurrent build pod, ~3 GB headroom.

---

## 4. CI supply chain — code push to signed image

```
Developer (Alice)                   GitHub                    Harness CI                     Docker Hub
─────────────────                   ──────                    ──────────                     ──────────

git push feature/X                  team-bwr-web              webhook                        
                  ─────────────────►   commit              ─────►
                                                              ▼
                                                          ┌────────────────────────────┐
                                                          │ bwr-web-ci pipeline        │
                                                          │ Stage: Build               │
                                                          │   on KubernetesDirect      │
                                                          │   (lab-delegate, kind)     │
                                                          ├────────────────────────────┤
                                                          │ 1. Maven Build             │
                                                          │    (maven:3.9-eclipse-     │
                                                          │     temurin-21-alpine)     │
                                                          │    mvn clean package       │
                                                          ├────────────────────────────┤
                                                          │ 2. Unit Tests              │
                                                          │    mvn test → Surefire     │
                                                          │    XML → JUnit reports     │
                                                          ├────────────────────────────┤
                                                          │ 3. Build Image             │
                                                          │    BuildAndPushDocker-     │
                                                          │    Registry (kaniko)       │
                                                          │    BASE_IMAGE=             │
                                                          │      acme-platform/base-java-21 │
                                                          │    push to dockerhub       ────► acme-platform/bwr-web:<sha>
                                                          ├────────────────────────────┤                    :latest
                                                          │ 4. Trivy Scan              │
                                                          │    AquaTrivy (orch mode)   │
                                                          │    fail_on_severity=none   │
                                                          │    [lab: production=high] │
                                                          ├────────────────────────────┤
                                                          │ 5. SBOM and Sign           │
                                                          │    alpine + syft + cosign  │
                                                          │    syft → CycloneDX SBOM   │
                                                          │    cosign sign --key=...   ────► signature artifact
                                                          │    cosign attest sbom      ────► attestation artifact
                                                          └────────────────────────────┘
```

### Step-by-step what gets pushed to Docker Hub

After a successful `bwr-web-ci` run, three artifacts land at `acme-platform/bwr-web`:

1. **The image itself** — a multi-layer container, FROM the platform base image
2. **A Cosign signature** — stored alongside as a separate OCI artifact (`*.sig` tag)
3. **A CycloneDX SBOM attestation** — same pattern, includes the package inventory

`cosign verify --key cosign.pub docker.io/acme-platform/bwr-web:<sha>` validates all three. Anyone with the public key can verify; nobody can forge without the private key.

### Why both signature *and* SBOM attestation?

| Signature alone | + SBOM attestation |
|---|---|
| "This image was produced by someone with the platform key" | "This image was produced by the platform AND here is exactly what's inside it" |
| Authenticates origin | Authenticates origin AND content inventory |
| Doesn't help with CVE response | Lets you cross-reference future CVE advisories against deployed inventory in seconds |

The supply-chain trail is: every base image is signed by platform → every domain image is signed by platform → every domain image's SBOM is platform-attested. End-to-end.

---

## 5. CD supply chain — image to running pod via GitOps

```
bwr-web-ci finishes              Harness CD (account.standard_cd_pipeline)         team-bwr-gitops              Harness GitOps Agent (Argo CD)            kind cluster
(signed :<sha> image)            ──────────────────────────────────────────         ─────────────────            ──────────────────────────────            ────────────

                                 ┌──────────────────────────────────────┐
                                 │ bwr-web-cd pipeline                  │
                                 │ ~45 lines, thin wrapper              │
                                 │ template: standard_cd_pipeline       │
                                 ├──────────────────────────────────────┤
                                 │ Stage: gitops_deploy_stage (CI-type) │
                                 │ on KubernetesDirect                  │
                                 ├──────────────────────────────────────┤
                                 │ 1. parse_config                      │
                                 │    git clone pe-pipeline-config +    │
                                 │      team-bwr-pipeline-config        │
                                 │    config_parser.sh: layer +         │
                                 │      schema-validate                 │
                                 │    write /harness/config/            │
                                 │      resolved.yaml                   │
                                 ├──────────────────────────────────────┤
                                 │ 2. verify_image_signature_inline     │
                                 │    cosign verify <appImage>          │
                                 │    cosign verify-attestation         │
                                 │      cyclonedx <appImage>            │
                                 │    HARD-FAIL if either fails         │
                                 ├──────────────────────────────────────┤
                                 │ 3. render_manifests                  │
                                 │    cosign verify chart at            │
                                 │      registry-1.docker.io/acme-platform/  │
                                 │      microservice:<chartVersion>     │
                                 │    helm pull (OCI)                   │
                                 │    helm template w/ resolved values  │
                                 │    → /harness/manifests/<app>/<env>/ │
                                 │       <region>/                      │
                                 ├──────────────────────────────────────┤
                                 │ 4. commit_manifests                  │
                                 │    git clone team-bwr-gitops         │
                                 │    replace tree at                   │
                                 │      <app>/manifests/<env>/<region>/ │
                                 │    git commit + push                 ─────► [commit on main]
                                 │                                      │      Deployment.yaml OR
                                 │                                      │      Rollout.yaml (if canary)
                                 │                                      │      Service / SA / ConfigMap
                                 ├──────────────────────────────────────┤            │
                                 │ 5. gitops_sync                       │            │
                                 │    POST Harness GitOps API           │            │
                                 │    /agents/.../applications/         │            │
                                 │      <app>-<env>-<region>/sync       │            │ Argo CD reconciles
                                 ├──────────────────────────────────────┤            ▼
                                 │ 6. wait_healthy                      │      ┌────────────────────────────┐
                                 │    kubectl preflight                 │      │ Argo CD diffs git ↔ live   │
                                 │    Path 1: kubectl get rollout       │ ───► │ Applies new manifests      │
                                 │      → if phase=Healthy, pass        │      │ Hands Rollout CRD to       │
                                 │    Path 2: GitOps API                │      │  Argo Rollouts controller  │
                                 │      → if Healthy+Synced, pass       │      └──────────┬─────────────────┘
                                 │                                      │                 │
                                 │    For canary: waits out             │                 │ canary phasing:
                                 │    setWeight + pause cycles          │                 │ 10/30/60/100%
                                 │    until Rollout reports Healthy     │                 │ + AnalysisRun gates
                                 ├──────────────────────────────────────┤                 ▼
                                 │ 7. smoke_test                        │       ┌──────────────────────────┐
                                 │    curl http://<app>.<app>-<env>.    ──────► │ bwr-web-prod ns          │
                                 │      svc.cluster.local:8080/healthz  │       │   Rollout bwr-web        │
                                 │    expect 200                        │       │   stable RS + canary RS  │
                                 └──────────────────────────────────────┘       │   smoke test → 200       │
                                                                                └──────────────────────────┘
```

### What's wired into Harness GitOps (visible in Harness UI)

The CD chain uses **Harness-native GitOps objects** (not just kubectl-applied ArgoCD CRDs):

| Harness object | Identifier | Scope | Notes |
|---|---|---|---|
| GitOps Agent | `account.labgitopsagent` | account | one agent serves all projects |
| GitOps Cluster | `account.incluster` | account | wraps `https://kubernetes.default.svc` |
| GitOps Repository | `account.teambwrgitops_tbcwsjha` | account | team-bwr-gitops, used by all 3 envs |
| Service | `bwr_web` | bwr_web project | Kubernetes type, gitOpsEnabled |
| Environments | `dev`, `staging`, `prod` | bwr_web project | dev+staging=PreProduction; prod=Production |
| GitOps Applications | `bwr-web-{dev,staging,prod}-us-east-1` | bwr_web project | one per env, mapped to ArgoCD AppProject `bwr-web` |
| ArgoCD AppProject mapping | argo `bwr-web` ↔ harness `bwr_web` | account agent | required for project-scoped GitOps Applications; created via "Import Projects" UI flow (API endpoint not exposed) |

`gitops_sync` and `wait_healthy` poll the **Harness GitOps API**, not the cluster directly. This means Harness's Deployments dashboards know about every CD run — visible in the Harness UI under bwr_web → Deployments.

### Why GitOps instead of pipeline pushing directly to k8s?

| Direct push (`kubectl apply` from pipeline) | GitOps (this design) |
|---|---|
| Pipeline log is the deploy log | `team-bwr-gitops` commit history is the deploy log; never lost |
| Cluster state can drift from intent | ArgoCD continuously reconciles; drift is auto-corrected (selfHeal: true) |
| Disaster recovery: re-run pipeline | Disaster recovery: re-apply ArgoCD Application; manifests already in Git |
| Multi-cluster needs N pipelines | One manifest repo + N ArgoCD Applications |

### What domain teams DON'T see

The CD pipeline. They edit `team-bwr-pipeline-config` (config values) or `team-bwr-web` (code). They never edit `team-bwr-gitops`. The CD pipeline is platform-team-owned automation. ArgoCD is invisible to them.

---

## 5b. Canary in prod — Argo Rollouts driven by hierarchical config

The same `bwr-web-cd` pipeline runs against `env=dev` (~30s rolling deploy) AND `env=prod` (~8min canary deploy). The pipeline doesn't know which strategy will be used. The choice is data, not code.

### How the strategy flows from config to chart

```
pe-pipeline-config/environments/prod.yaml
  app:
    deployStrategy: canary           ← platform decides: prod = canary
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 120 }
        - setWeight: 30
        - pause: { duration: 180 }
        - setWeight: 60
        - pause: { duration: 180 }
        - setWeight: 100
      analysisTemplate: success-rate

         │ parse_config layers this with team-bwr-pipeline-config
         │ (domain layer) and writes /harness/config/resolved.yaml
         ▼

render_manifests reads resolved.yaml, runs:
  helm template microservice/ --values resolved.yaml

         │ Helm chart (pe-helm-charts/charts/microservice/templates/) has:
         │   {{- if eq .Values.app.deployStrategy "rolling" }}  → Deployment
         │   {{- if eq .Values.app.deployStrategy "canary" }}   → Rollout
         ▼

commit_manifests pushes to team-bwr-gitops/bwr-web/manifests/prod/us-east-1/
  rollout.yaml  (Argo Rollouts CRD with the canary spec)
  service.yaml  serviceaccount.yaml  configmap.yaml

         │ ArgoCD syncs
         ▼

Argo Rollouts controller (cluster-wide) takes over:
  setWeight 10  → 1 canary pod alongside 4 stable pods
  pause 120s
  setWeight 30  → 1-2 canary pods
  pause 180s     ← AnalysisRun fires the success-rate template
                   (job-based; in production: Prometheus query)
                   if metric fails → rollback automatically
  setWeight 60  → 2-3 canary pods
  pause 180s
  setWeight 100 → kill stable pods, canary becomes new stable

         │ Rollout reports status.phase=Healthy when complete
         ▼

wait_healthy step polls Rollout CRD via kubectl
  (NOT ArgoCD's Application health, which doesn't natively
   recognize Rollout CRD as Healthy — see known-issues doc)

         │ exits success when rollout.phase = Healthy
         ▼

smoke_test  → 200 OK   →   pipeline ends Success
```

### Why the wait_healthy step polls the Rollout directly

ArgoCD does not natively understand the `Rollout` CRD's health — it stays at `Progressing` forever even after the Rollout reports `phase: Healthy`. There's a Lua resource customization for `argoproj.io/Rollout` that should fix this, but it didn't load reliably on the Harness GitOps Agent's bundled ArgoCD. Workaround: `wait_healthy` runs `kubectl get rollout` directly with the in-pod ServiceAccount token.

### Design note

> "The pipeline doesn't say 'canary'. The platform's hierarchical config does — `prod.yaml` sets `deployStrategy: canary` and the helm chart picks Deployment vs Rollout based on that value. Same image, same chart, same pipeline; one input changes the safety story. To switch dev to canary, edit `dev.yaml`. To downgrade prod to rolling, edit `prod.yaml` (PR-reviewed, audited). Domain teams cannot opt out — the schema doesn't expose `app.deployStrategy` as overridable."

### Live signals (what to look at during a canary)

| What you'd show | Where to look |
|---|---|
| Rollout CRD phasing live | `kubectl argo rollouts get rollout bwr-web -n bwr-web-prod -w` |
| Stable + canary ReplicaSets coexisting at 60% weight | `kubectl get pods -n bwr-web-prod` — two ReplicaSets, ~5-7 pods total |
| AnalysisRun result | `kubectl get analysisruns -n bwr-web-prod` — Status: Successful |
| Manifest diff that triggered the canary | `git log -p -1 team-bwr-gitops` — `image:` field changed |

---

## 6. Configuration resolution layer (Hiera-equivalent + 12-factor)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                  pe-pipeline-config/hierarchy.yaml                           │
│  Declares the lookup chain (data, not code)                                  │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ read by
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│         pe-pipeline-config/lib/config_parser.sh                              │
│  yq-based merger; walks every layer in declared order;                       │
│  validates domain layer against schema/exposed-values.yaml                   │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ produces
                                    ▼
                  resolved.yaml (single merged config)
                                    │
                                    │ consumed by
                       ┌────────────┼────────────┐
                       ▼            ▼            ▼
                 [pipeline   [Helm chart   [pipeline]
                  envVars]    .Values]      step inputs]


─── Layer order (least-specific → most-specific, last wins) ───

┌──────────────────────────────────────────────────────────────────────────────┐
│ PLATFORM-OWNED LAYERS (in pe-pipeline-config repo)                           │
│                                                                              │
│  common/account.yaml        ← devsecops baseline (locked)                    │
│  applications/_defaults     ← any-app defaults                               │
│  environments/<env>.yaml    ← per-env defaults                               │
│  regions/<region>.yaml      ← per-region defaults                            │
│  overrides/<env>/<region>   ← env × region intersections                     │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │  ─── boundary ───
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ DOMAIN-OWNED LAYERS (in team-<team>-pipeline-config repo)                    │
│                                                                              │
│  applications/<app>/application.yaml         ← per-app defaults              │
│  applications/<app>/environments/<env>.yaml  ← per-app env override          │
│  applications/<app>/regions/<region>.yaml    ← per-app region override       │
│                                                                              │
│  ☆ ALL three layers schema-validated against:                                │
│    pe-pipeline-config/schema/exposed-values.yaml                             │
│  Setting a key NOT in `overridable:` → pipeline hard-fails                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### What "schema-validated" means in practice

The schema lists the keys (or subtrees) domain teams may override. Anything not listed is locked.

```yaml
# pe-pipeline-config/schema/exposed-values.yaml (excerpt)
overridable:
  - app.replicas               # exposed: scalar
  - app.envVars                # exposed: any nested key under app.envVars
  - app.config                 # exposed: ConfigMap-shaped K=V map
  - app.configFiles            # exposed: filename → file-body map
  - hpa.minReplicas
  - hpa.maxReplicas
  # etc.
```

Anything not listed (like `security.runAsNonRoot`, `image.registry`, `scanning.*`) is **locked**. If a domain team's `team-bwr-pipeline-config/applications/bwr-web/application.yaml` tries to set `security.runAsNonRoot: false`, `config_parser.sh` fails the run with a message pointing at the schema file.

This is enforced *before* anything else runs. It's the contract between platform and domain teams, codified.

### 12-factor: app config NOT baked into the image

```
team-bwr-pipeline-config/applications/bwr-web/environments/dev.yaml
  app:
    config:                                ← simple env vars
      BWR_DB_URL: jdbc:postgres://...
      BWR_DB_POOL_SIZE: "10"
    configFiles:                           ← file-shaped config
      application-override.yaml: |
        management:
          endpoint:
            health:
              show-details: always

  ─── flows through config_parser into Helm chart ───

pe-pipeline-templates/templates/helm-library/microservice/
  templates/configmap.yaml         ← renders ConfigMap with `data:`
  templates/_helpers.tpl           ← pod template gets:
                                      • envFrom: configMapRef    (config keys)
                                      • volume mount /app/config (configFiles)
                                      • annotation
                                        checksum/config: <sha256 of ConfigMap>
                                        ───────────────────────────────────
                                        flips when ConfigMap changes
                                        → pod template diff
                                        → ArgoCD reconciles
                                        → rolling restart
                                        → image bit-identical
```

**Verified end-to-end**: changing `BWR_DB_POOL_SIZE` from 10 to 20 in the domain config flips the `checksum/config` annotation on the pod template. The image stays the same. Pods restart. CI never runs.

---

## 7. Policy enforcement points

```
                       ┌──────────────────────────────────────────┐
                       │  pe-policies (single source of truth)    │
                       │  4 Rego files + 18 unit tests + Policy   │
                       │  Set definition                          │
                       └──────────────────────────────────────────┘
                                          │
                                          │ gets applied to
                                          ▼
                            ┌─────────────┴─────────────┐
                            │                           │
                            ▼                           ▼
              ╔══════════════════════╗     ╔══════════════════════╗
              ║   POINT 1            ║     ║   POINT 2            ║
              ║   Schema validation  ║     ║   Harness Policy     ║
              ║   (config_parser.sh) ║     ║   Engine (OPA)       ║
              ║                      ║     ║                      ║
              ║   At config-merge    ║     ║   At pipeline save   ║
              ║   time. Catches      ║     ║   AND run. Catches   ║
              ║   domain teams       ║     ║   any pipeline that  ║
              ║   trying to set      ║     ║   would violate the  ║
              ║   locked keys.       ║     ║   4 Rego policies.   ║
              ║                      ║     ║                      ║
              ║   Hard-fail.         ║     ║   Hard-deny on run.  ║
              ╚══════════════════════╝     ╚══════════════════════╝

                         ─── future enforcement points ───

              ╔══════════════════════╗     ╔══════════════════════╗
              ║   POINT 3 (TBD)      ║     ║   POINT 4 (TBD)      ║
              ║   K8s admission ctrl ║     ║   Manifest-repo PR   ║
              ║   (Kyverno or        ║     ║   gate (conftest)    ║
              ║    Gatekeeper)       ║     ║                      ║
              ║                      ║     ║   At PR merge to     ║
              ║   At apiserver       ║     ║   team-bwr-gitops.   ║
              ║   admission. Catches ║     ║   Catches manual     ║
              ║   anything that gets ║     ║   manifest edits.    ║
              ║   to the cluster,    ║     ║                      ║
              ║   GitOps OR direct.  ║     ║   For when developers║
              ║                      ║     ║   try to edit the    ║
              ║                      ║     ║   gitops repo by hand║
              ╚══════════════════════╝     ╚══════════════════════╝
```

### The 5 OPA policies (active today)

| Policy | What it checks | When |
|---|---|---|
| `require-platform-base-image` | `BASE_IMAGE` build arg starts with `docker.io/acme-platform/base-` | onrun |
| `require-image-signature-verification` | Pipelines with K8s deploy steps must have a Cosign verify step earlier | onrun |
| `require-platform-pipeline-template` | Pipelines in `bwr_*` projects must reference an allowed Pipeline Template (`standard_ci_pipeline` / `standard_cd_pipeline` / `java_maven_image_pipeline`) | onsave |
| `disable-scan-needs-justification` | If a pipeline disables scanning at runtime, it must include a Jira ticket reference in `tags.justification` matching `^[A-Z][A-Z0-9]+-[0-9]+$` | onrun |
| `require-pinned-chart-version` *(NEW)* | Domain CD pipelines must pass an explicit pinned semver chartVersion. `latest`, `stable`, branch names, empty values denied. Pinning forces an explicit upgrade PR — that's the audit trail. | onsave |

All hard-deny. **Verified behavior**: bwr_web_ci on the `try-non-platform-base` branch fired TWO violations simultaneously (`require-platform-base-image` + `require-platform-pipeline-template`), pipeline never ran.

### Known issue — Policy Set enable toggle is UI-only

The Harness Policy Engine (`/pm/api/v1/...`) supports POST/GET/PATCH on policies and policy sets, but **the `enabled` flag on a policy set cannot be toggled via API on the free tier**. PATCH returns HTTP 204 but the field stays `false`. The toggle has to be flipped manually in the Harness UI under the Policy Engine module (which itself does not appear in the module switcher on free tier — navigate via direct URL or via the `Continuous Delivery & GitOps` module's left sidebar).

**Operational note**: the policies + policy set are committed to git, registered with Harness, and visible via API. Enforcement requires the policy set's `enabled` flag to be set to true — which has to be done in the Harness UI on free tier (the API endpoints exist but don't toggle the flag).

---

## 8. Identity & secrets flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                  Harness Account (account scope)                             │
│                                                                              │
│  Connectors (3):                                                             │
│    • harnessdemo          → GitHub PAT, reads/writes acme-platform org    │
│    • dockerhub            → Docker Hub PAT for acme-platform namespace            │
│    • kind_harness_lab     → K8s connector via lab-delegate                   │
│                                                                              │
│  Secrets (5):                                                                │
│    • cosign_signing_key       (File) ← private key for image signing         │
│    • cosign_signing_key_password (Text)                                      │
│    • cosign_public_key        (File) ← for verify-on-deploy step             │
│    • dockerhub                (Text) ← Docker Hub PAT                        │
│    • github                   (Text) ← GitHub PAT (used in CD pipeline       │
│                                         for cloning private repos)           │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │ Pipelines reference these by
                                       │ identifier: account.<name>
                                       ▼
              ┌────────────────────────┴────────────────────────┐
              │                                                 │
              ▼                                                 ▼
     CI Pipeline secrets:                          CD Pipeline secrets:
       cosign_signing_key       (sign artifact)      cosign_public_key  (verify)
       cosign_signing_key_pw    (sign artifact)      github             (clone gitops repo)
       dockerhub                (push image)         dockerhub          (manifests reference image)
```

### Cosign key flow

```
~/harness-lab/secrets/cosign.key   ← private key, encrypted with password
                                     (never leaves user's local machine + Harness Secrets vault)

         │ cosign sign uses
         ▼
   image manifest digest signed
         │
         │ stored at
         ▼
   Docker Hub: acme-platform/<image>:sha256-<digest>.sig

   ◀───── verify with ─────────────────────────────────────────
   ~/harness-lab/secrets/cosign.pub  ← public key
   (also uploaded to Harness as cosign_public_key for CD pipelines
    to verify before deploy)
```

**Design note**: the private key never enters a build pod's image. It's mounted from Harness Secrets at step runtime, written to `/tmp/cosign.key` (chmod 600), used, and `rm -f /tmp/cosign.key` at step exit. Pod lives ~5 min, image is ephemeral, key is in volatile memory only.

---

## 9. The Harness templates layer (what makes this scalable)

The whole platform-vs-domain story is enforced by a 3-tier template chain. Domain pipelines are **thin wrappers** referencing platform pipeline templates; platform pipeline templates compose stage templates; stage templates compose step templates.

```
                              Harness Account → Templates (account scope)
                                          │
                ┌─────────────────────────┼─────────────────────────────┐
                ▼                         ▼                             ▼
       ╔══════════════════╗     ╔══════════════════╗        ╔══════════════════════════╗
       ║ Pipeline         ║     ║ Stage            ║        ║ Step                     ║
       ║ Templates (3)    ║     ║ Templates (3)    ║        ║ Templates (~12 active)   ║
       ╠══════════════════╣     ╠══════════════════╣        ╠══════════════════════════╣
       ║ standard_image_  ║     ║ image_build_     ║   CI:  ║ hadolint_lint            ║
       ║   build_pipeline ║     ║   stage          ║        ║ build_image              ║
       ║   (base images)  ║     ║                  ║        ║ container_scan           ║
       ║                  ║     ║ java_maven_      ║        ║ cosign_sign_image        ║
       ║ java_maven_      ║     ║   test_stage     ║        ║ java_maven_test          ║
       ║   image_pipeline ║     ║                  ║        ║                          ║
       ║   (Java apps)    ║     ║ gitops_deploy_   ║   CD:  ║ parse_config             ║
       ║                  ║     ║   stage          ║        ║ verify_image_signature_  ║
       ║ standard_cd_     ║     ║                  ║        ║   inline                 ║
       ║   pipeline       ║     ║                  ║        ║ render_manifests         ║
       ║   (deploy)       ║     ║                  ║        ║ commit_manifests         ║
       ║                  ║     ║                  ║        ║ gitops_sync              ║
       ║                  ║     ║                  ║        ║ wait_healthy             ║
       ║                  ║     ║                  ║        ║ smoke_test               ║
       ╚══════════════════╝     ╚══════════════════╝        ╚══════════════════════════╝
                ▲                         ▲                             ▲
                │                         │                             │
                │ pipelines               │ stages compose              │ steps read
                │ reference               │ step templates              │ <+stage.variables.X>
                │ pipeline templates      │                             │ directly — no <+input>
                │ via templateInputs      │                             │ at leaf level
                │                         │                             │
```

### Three thin domain pipelines, each ~45 lines:

| Domain pipeline | Wraps | LOC | What domain team writes |
|---|---|---|---|
| `pe-base-images-ci` (platform-internal but follows the same pattern) | `standard_image_build_pipeline` | ~50 | imageName, dockerfilePath, hadolintIgnore |
| `bwr-web-ci` | `java_maven_image_pipeline` | ~45 | imageName, baseImage |
| `bwr-web-cd` | `standard_cd_pipeline` | ~45 | app, env (runtime), appImage, chartVersion, gitopsRepo |

### The three reconcile gotchas we hit and the patterns that work

Harness's deep template reconcile has three subtleties that took experimentation to nail down. Documented here so the next person doesn't re-discover them:

1. **`<+input>` at leaf level bubbles to runtime if no `.default()`.** The stage template's `templateInputs` override is sometimes silently ignored. → **Fix**: leaf step templates read `<+stage.variables.X>` directly. Stage template owns the variables; pipeline template fills them. No `<+input>` at the bottom of the chain.

2. **`<+input>.default("X")` preserves literal quotes when resolved.** Run-step shell sees `IMAGE="docker.io/foo:bar"` (with quotes) and most CLIs reject it. → **Fix**: POSIX prefix/suffix-strip at the top of each shell command (`VAR=${VAR#\"}; VAR=${VAR%\"}`). The bash form `${VAR//\"/}` doesn't work in Alpine `sh`.

3. **Variables marked `required: true` reject empty-string values, even if `<+input>.default("")` is set.** → **Fix**: mark optional-with-default variables `required: false` (e.g., `baseImage`, `hadolintIgnore`).

These three rules in combination give templateInputs that survive deep reconcile chains. The thin-wrapper pattern works as a result.

### Design note

> "The platform team owns three pipeline templates: image-build, java-maven-image, and CD. Every domain CI pipeline is a thin wrapper — ~45 lines passing values into the platform template. Domain teams cannot author raw pipelines (OPA blocks `require-platform-pipeline-template`). The platform team can ship a security improvement to all three thin wrappers by editing one stage template; no domain team has to do anything."

---

## 10. System walkthrough

The system can be exercised top-to-bottom by following the steps below; each step verifies one layer of the architecture against the live system.

### 1. Context

Two teams: a platform team writing templates and policies, a domain team (Baby & Wedding Registry) writing app code. The design intent is to make the right thing easy and the wrong thing impossible. `01-developer-flow.md` gives the same story from a developer's perspective.

### 2. Repo split

GitHub `acme-platform` org. Walk through the 8 repos in two groups:
- **Platform**: `pe-pipeline-config` (open it briefly, point at `hierarchy.yaml` and `schema/exposed-values.yaml`)
- **Domain**: `team-bwr-web` (open `.harness/pipeline.yaml` and `.harness/cd-pipeline.yaml`; each is ~45 lines, both reference platform pipeline templates; emphasize what's NOT there — no scan config, no signing, no base-image FROM, no canary phasing logic)

### 3. Signed base image

```bash
cosign verify --key ~/harness-lab/secrets/cosign.pub docker.io/acme-platform/base-java-21:latest
```

Two signatures: Cosign + CycloneDX SBOM attestation. **Design note**: this is the chain-of-custody for everything domain teams ship.

### 4. Pipelines running

Harness UI → bwr_web → Pipelines:
- `bwr-web-ci` (point at the latest green run; click into it; show stages)
- `bwr-web-cd` (same)

**Design note**: same supply chain (build, scan, SBOM, sign) as the base image, but for the domain team's app. They didn't write any of the supply-chain logic.

### 5. Policy block

Click into Build #N on `bwr-web-ci` (the `try-non-platform-base` branch run). Open the **Policy Set Evaluations** dialog. **Two policies fired simultaneously**.

**Design note**: a developer changed one variable to bypass the platform; OPA evaluated all 5 policies on the pipeline definition; two of them denied with actionable messages. The pipeline never started.

### 6. Running pod

Terminal:
```bash
kubectl get pods -n bwr-web-dev
kubectl logs -n bwr-web-dev deploy/bwr-web | head
kubectl exec -n bwr-web-dev deploy/bwr-web -- curl -s http://localhost:8080/healthz
```

ArgoCD UI (or `kubectl get application -n harness-gitops-ns bwr-web-dev-us-east-1`): Synced, Healthy.

**Design note**: pod was put there by ArgoCD reconciling from a Git commit the CD pipeline made. The Git history of `team-bwr-gitops` IS the deploy log.

### 7. 12-factor config restart

Open `team-bwr-pipeline-config/applications/bwr-web/environments/dev.yaml`. Point at `app.config` block.

**Design note**: changing any of these triggers the CD pipeline (no CI), which renders a new ConfigMap, the Deployment template gets a new `checksum/config` annotation, ArgoCD detects, pods rolling-restart. The container image is bit-identical. That's 12-factor done right.

### 8. Schema-validated config

Open `pe-pipeline-config/schema/exposed-values.yaml`. Walk through `overridable:` list.

**Design note**: this is the contract. Domain teams can override any of these keys. Anything else is locked. If they try to set `security.runAsNonRoot: false` in their config, `config_parser.sh` fails the pipeline run before anything else runs. Tested with 27/27 unit tests on the parser.

### 9. Deferred items

**Design note**: items deferred for future work — Terraform-managed Harness resources to remove reliance on the UI reconcile, multi-region promotion automation, Harness CV gating on canary analysis. The platform team maintains a deferred-items register with rationale for each.

### 10. Considered alternatives

Open `pe-pipeline-config/README.md`, scroll to "Considered alternatives".

**Design note**: YAML was chosen deliberately after evaluating TOML, CUE, Jsonnet, HCL, JSON, Dhall. CUE was the strongest technical fit because it unifies schema-and-data; rejected for ecosystem cost at this scale. Worth revisiting at >50 domain teams.

---

## 11. Failure modes catalog

| What fails | What the developer sees | Resolution path |
|---|---|---|
| Domain team sets a non-platform `BASE_IMAGE` | OPA Policy Set evaluation: `require-platform-base-image` denies; pipeline never runs | Use `ARG BASE_IMAGE / FROM ${BASE_IMAGE}`; let platform inject. |
| Domain team writes raw pipeline YAML in their project | OPA: `require-platform-pipeline-template` denies on save | Reference a platform Pipeline Template. |
| Domain team disables scanning at runtime | OPA: `disable-scan-needs-justification` denies unless `pipeline.tags.justification` matches Jira pattern | Add the Jira ref. |
| Domain team sets a locked key in `team-*-pipeline-config` (e.g., `security.runAsNonRoot: false`) | `config_parser.sh` fails with "key X is not in `schema/exposed-values.yaml`" | Either remove the override OR open a PR to expose the key (platform-reviewed). |
| Image scan finds HIGH+ CVE | Trivy step fails; image never gets to SSCA Orchestration; `:latest` tag not republished | Wait for nightly rebuild trigger to pull patched upstream OR escalate. |
| Cosign signature missing/invalid at deploy time | `verify-image-signature` step fails; manifests never get committed; ArgoCD has nothing to reconcile | Investigate signing pipeline failure. |
| Build pod can't schedule (cluster OOM) | Pipeline shows resource error | Increase Docker Desktop allocation or scale down non-essential cluster components |
| Cosign rejects `oci://` prefix on chart verify | render_manifests step exits non-zero with `unsupported scheme: oci` | Strip the prefix before passing to cosign (`CHART_REF_NOPREFIX=${CHART_OCI_REF#oci://}`); keep it for `helm pull`. Already in `render_manifests` |
| Cosign UNAUTHORIZED on push | Cosign signature push fails with 401 | Cosign needs login at the SAME hostname as the artifact target. Image artifacts: `cosign login index.docker.io`. Helm OCI artifacts: `cosign login registry-1.docker.io` |
| Public key secret contains private key | `cosign verify` fails: `unknown Public key PEM file type: ENCRYPTED SIGSTORE PRIVATE KEY` | The Harness `cosign_public_key` SecretFile got created with the private key body. Re-upload via the Files multipart endpoint with the actual `cosign.pub` content |
| Argo CD reports Rollout as Progressing forever | wait_healthy never exits via the GitOps API path | Lua resource customization for `argoproj.io/Rollout` doesn't load reliably; `wait_healthy` polls Rollout CRD via kubectl directly as the primary path |
| `<+input>.default("x")` env vars wrap value in literal quotes | CLI tool rejects argument with `"docker.io/foo:latest"` | POSIX prefix/suffix-strip at top of step shell command; documented per-step in `pe-pipeline-templates/templates/steps/*.yaml` |

---

## 12. Architectural decisions log

| Decision | What we picked | Why | Where documented |
|---|---|---|---|
| Config language | YAML | Ecosystem alignment with Helm/K8s/Harness; CUE/Jsonnet considered and deferred | `pe-pipeline-config/README.md` |
| Config hierarchy | Platform-owned + domain-owned, schema-validated | Hiera's flexibility + Harness Service Override v2's UI + a code-reviewable schema contract | `pe-pipeline-config/schema/exposed-values.yaml` |
| Repo strategy | 8 repos by ownership × cadence | Platform writes `pe-*` rarely; domain writes `team-bwr-*` daily; gitops repo auto-written | §2 of this document |
| Application layer ownership | Domain owns | Platform should not bottleneck on every new app | `team-bwr-pipeline-config/applications/bwr-web/application.yaml` |
| GitOps repo per team | Yes (multi-app) | Mirrors pipeline-config; one team-scoped repo for all apps | `team-bwr-gitops` layout |
| Image base policy | OPA hard-deny on any non-`docker.io/acme-platform/base-*` | Supply-chain integrity > developer convenience | `pe-policies/harness/require-platform-base-image.rego` |
| Build infrastructure | KubernetesDirect via lab-delegate | Free tier; self-hosted; matches "no Cloud minutes consumed" production constraint | `03-runbook.md` §1 |
| Deploy strategy | GitOps (ArgoCD) | Single source of truth in Git; reconcile loop survives operator absence | `01-developer-flow.md` Scenario 4 |
| Canary phasing | Argo Rollouts (manifest-driven) | Phasing logic lives in Git as a `Rollout` CRD; the pipeline writes manifests, doesn't orchestrate canary | `pe-helm-charts/charts/microservice/templates/rollout.yaml` |
| 12-factor config | ConfigMap + envFrom + checksum annotation | Restart-on-change without rebuild; auditable; standard pattern | `pe-pipeline-templates/.../helm-library/microservice/templates/configmap.yaml` |
| Cosign vs. SSCA module | Cosign+Syft inline (free tier doesn't include SSCA module) | Same artifact outcome; doesn't depend on Harness license tier | `pe-pipeline-templates/templates/steps/cosign-sign-image-step.yaml`, `verify-image-signature-step.yaml` |
| Rolling vs canary in dev | Rolling | Canary needs Argo Rollouts CRDs + analysis; OK for prod, overkill for dev | `pe-pipeline-config/environments/{dev,prod}.yaml` |
| Reconcile-survival pattern | Leaf step templates read `<+stage.variables.X>` directly (no `<+input>` at leaf) | Harness reconcile drops templateInputs overrides through deep chains; reading stage variables sidesteps the issue without Terraform | step template comments |
| CD stage type | CI-type stage (Run steps), not Deployment-type | Deployment-type with gitOpsEnabled requires Service+Environment+Cluster+Application all wired together; CI-type with kubectl polling for Rollout health is simpler and works the same | `pe-pipeline-templates/templates/stages/cd-deploy-stage.yaml` |
| Helm chart distribution | Cosign-signed OCI artifact at `acme-platform/microservice:<semver>`, pulled at deploy time | Pinned, immutable, verifiable; domain teams cannot fork or override; `require-pinned-chart-version` policy forbids floating tags | `pe-helm-charts-ci` pipeline, `render_manifests` step |
| Canary health source | `kubectl get rollout` directly, not ArgoCD Application health | ArgoCD doesn't natively understand Argo Rollouts CRD health on the Harness GitOps Agent's bundled ArgoCD; Rollout CRD is the source of truth for canary progression anyway | `pe-pipeline-templates/templates/steps/wait-healthy-step.yaml` |
| AnalysisTemplate provider | Job-based (synthetic for the lab; query Prometheus in production) | Lab has no Prometheus; Job-based AnalysisTemplate proves the gating mechanism works without the metric backend | `kubectl get analysistemplate -n bwr-web-prod success-rate` |

---

## 13. Companion documents

| Doc | Purpose |
|---|---|
| [`01-developer-flow.md`](01-developer-flow.md) | Six end-user scenarios from a domain developer's perspective, each with an operating-principle callout |
| [`02-architecture.md`](02-architecture.md) | This file |
| [`03-runbook.md`](03-runbook.md) | Step-by-step setup from authored YAML to live system |

---

## 14. Summary

Three load-bearing claims, all verified end-to-end:

1. **The platform is invisible to the developer most of the time.** Domain teams see PRs, green pipelines, deployed services. They don't see scanning, signing, OPA, ArgoCD.
2. **The platform is unbreakable on the things that matter.** Supply chain, base image, signature verification, chart version pinning — all hard-enforced through OPA policies and template-baked checks.
3. **Config is in the environment, never in the image.** A config change is a ~90-second CD run with no rebuild. Validated with `BWR_DB_POOL_SIZE` 10 → 20.
