# 03 — Integration Runbook

> Step-by-step from "all platform repos authored" to "BWR pipeline running on a real Harness account against a real cluster." Each step has a heading, exact commands, expected output, and a checkbox.

**Status:** Executed 2026-05-09. The lab is currently live. This runbook is the **as-authored recipe** — the live system diverged from it on a small number of operational details (connector identifiers, secret names, additional setup for canary, etc.) which the platform team tracks in an internal known-issues register. Read this top-to-bottom for the setup sequence.

**Live identifier reference (as actually deployed):**

| Authored-as | Actually in Harness |
|---|---|
| `github_acme_platform` connector | `harnessdemo` |
| `docker_hub_acme_platform` connector | `dockerhub` |
| `kind_harness_lab` connector | (same) |
| `github_gitops_pat` secret | `github` |
| `github_acme_platform_pat` secret | `github` |
| `bwr-gitops-agent` | `labgitopsagent` (account-scope) |

Org/Project layout simplified to single-Org (free tier): `default` Org holds all projects (`governance`, `bwr_web`).

---

## 0. Pre-flight checklist

- [ ] Docker Desktop running
- [ ] `kubectl` installed (we have v1.30+)
- [ ] `helm` v3.18+ installed
- [ ] `kind` installed (`brew install kind` if not)
- [ ] `cosign` installed (`brew install cosign` if not)
- [ ] `gh` CLI authenticated to `acme-platform` org (already verified)
- [ ] Harness account accessible at https://app.harness.io/ng/account/<HARNESS_ACCOUNT_ID>
- [ ] Docker Hub credentials available (username `acme-platform`)

```bash
# One-shot pre-flight verification
docker version --format '{{.Server.Version}}' && \
kubectl version --client --output=yaml | head -3 && \
helm version --short && \
which kind && which cosign && \
gh auth status
```

---

## 1. Stand up the kind cluster

A throwaway cluster for the lab. Named `harness-lab` so it's distinguishable from production clusters in `kubectl config get-contexts`.

```bash
# Create the cluster — extra port mappings for the ingress controller
cat > /tmp/kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: harness-lab
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
EOF
kind create cluster --config /tmp/kind-config.yaml

# Switch kubectl context
kubectl config use-context kind-harness-lab
kubectl get nodes
```

**Expected:** one `control-plane` node in `Ready` state.

- [ ] Cluster created
- [ ] kubectl context = `kind-harness-lab`

---

## 2. Install ingress controller (nginx)

Needed for the bwr-web ingress to actually route traffic from the host.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

**Expected:** controller pod `Ready` within 2 minutes.

- [ ] Ingress controller installed and ready

---

## 3. Install Argo Rollouts CRDs

Needed so the platform Helm chart's `kind: Rollout` resources have a controller to reconcile them when the deploy strategy is `canary`.

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl wait --namespace argo-rollouts \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=rollouts-controller \
  --timeout=120s
```

**Expected:** rollouts-controller pod `Ready`.

```bash
# Verify the CRDs are installed
kubectl get crd | grep argoproj
```

**Expected:** at minimum `rollouts.argoproj.io` and `analysisruns.argoproj.io`.

- [ ] Argo Rollouts CRDs installed
- [ ] rollouts-controller running

---

## 4. Generate Cosign keys

The platform's CI signs every image with a Cosign private key. CD pipelines verify with the matching public key. Both go into Harness as account-level secrets.

```bash
mkdir -p ~/.harness-lab/secrets
cd ~/.harness-lab/secrets

# Pick a strong password; you'll paste it into Harness as `cosign_signing_key_password`.
# Saving to a file ONLY for the lab — production would use a real KMS-backed key.
read -s -p "Choose a Cosign password: " COSIGN_PASSWORD; echo
echo "$COSIGN_PASSWORD" > cosign.password

COSIGN_PASSWORD="$COSIGN_PASSWORD" cosign generate-key-pair

# Files produced: cosign.key (private, encrypted) + cosign.pub (public)
ls -la
```

**Expected:** `cosign.key`, `cosign.pub`, `cosign.password` in `~/.harness-lab/secrets/`.

**You'll need these three values for Harness Secrets later:**
- `cosign.key` contents → `account.cosign_signing_key` (File secret)
- `cosign.password` contents → `account.cosign_signing_key_password` (Text secret)
- `cosign.pub` contents → `account.cosign_public_key` (File secret)

- [ ] Cosign keys generated and saved

---

## 5. Create GitHub Personal Access Tokens

Two PATs, both scoped to the `acme-platform` org:

| PAT name | Scopes | Used by |
|---|---|---|
| `harness-cd-readonly` | `repo:read` | Harness GitHub connector for cloning + reading templates |
| `harness-cd-gitops-write` | `repo` (full) on `acme-platform/team-*-gitops` only | `commit-manifests` step pushes to GitOps repos |

```bash
# Create at: https://github.com/settings/tokens/new
# (CLI alternative: gh auth refresh -s repo)
```

Save both tokens; you'll paste them into Harness as Text secrets:
- `account.github_acme_platform_pat` (read-only, the connector uses this)
- `account.github_gitops_pat` (the commit-manifests step uses this)

- [ ] Read-only PAT created and saved
- [ ] GitOps-write PAT created and saved

---

## 6. Create the Harness Orgs and Projects

Done in the UI for the first-time setup (Git Experience can sync these too, but for the initial run the UI is faster).

In https://app.harness.io/ng/account/<HARNESS_ACCOUNT_ID>:

1. Account Settings → Organizations → **+ New Org**
   - Name: `Platform Engineering`, identifier: `platform_engineering`
2. Inside that Org → **+ New Project**:
   - `CI Templates` (`ci_templates`)
   - `Base Images` (`base_images`)
   - `Governance` (`governance`)
3. Account Settings → Organizations → **+ New Org**
   - Name: `Domain BWR`, identifier: `domain_bwr`
4. Inside that Org → **+ New Project**:
   - `BWR Web` (`bwr_web`)

- [ ] platform_engineering org created with 3 projects
- [ ] domain_bwr org created with bwr_web project

---

## 7. Install the Harness Delegate

In Harness UI: Account Settings → Delegates → **+ New Delegate** → Kubernetes → Helm Chart.

Fill in:
- Name: `lab-delegate`
- Description: `kind cluster delegate for the platform lab`
- Tags: `lab`, `kind`

Copy the generated `helm install` command and run it locally:

```bash
kubectl create namespace harness-delegate-ng
# Paste the helm install command Harness gave you here
```

**Expected:** delegate pod `Ready` within 60 seconds; appears as `Connected` in Harness UI.

```bash
kubectl get pods -n harness-delegate-ng
```

- [ ] Delegate installed
- [ ] Delegate showing `Connected` in Harness UI

---

## 8. Install the Harness GitOps Agent

The agent is the Harness-packaged Argo CD that reconciles manifests from `team-*-gitops` repos.

In Harness UI: GitOps → Settings → Agents → **+ New Agent**.

- Name: `bwr-gitops-agent`
- Namespace: `harness-gitops-ns` (will be created)

Run the generated install command:

```bash
kubectl create namespace harness-gitops-ns
# Paste the install command Harness gave you
```

**Expected:** agent pods (`gitops-agent`, `redis`, `argocd-application-controller`, etc.) `Ready`.

- [ ] GitOps Agent installed
- [ ] Agent showing `Connected` in Harness UI

---

## 9. Create Harness Connectors (account scope)

In Harness UI: Account Settings → Connectors → **+ New Connector**.

| Connector | Identifier | Type | Auth |
|---|---|---|---|
| `github_acme_platform` | (same) | GitHub | Personal Access Token from step 5 (read-only) |
| `docker_hub_acme_platform` | (same) | Docker Registry | Docker Hub creds (username `acme-platform`, password = personal access token) |
| `kind_harness_lab` | (same) | Kubernetes Cluster | Use the lab delegate; in-cluster auth |

For each: select the lab delegate as the executor; click **Test Connection**.

- [ ] github_acme_platform: test passes
- [ ] docker_hub_acme_platform: test passes
- [ ] kind_harness_lab: test passes

---

## 10. Create Harness Secrets (account scope)

In Harness UI: Account Settings → Secrets → **+ New Secret**.

| Identifier | Type | Source |
|---|---|---|
| `cosign_signing_key` | File | `~/.harness-lab/secrets/cosign.key` |
| `cosign_signing_key_password` | Text | contents of `~/.harness-lab/secrets/cosign.password` |
| `cosign_public_key` | File | `~/.harness-lab/secrets/cosign.pub` |
| `github_acme_platform_pat` | Text | the read-only PAT |
| `github_gitops_pat` | Text | the GitOps-write PAT |

- [ ] All five secrets created

---

## 11. Sync platform templates via Git Experience

Each platform repo gets connected to Harness so its YAML auto-syncs as Templates / Pipelines / Policies.

For each of the five `pe-*` repos:

1. Account Settings → Git Experience → **+ Add Source**
2. Connector: `github_acme_platform`
3. Repo: `acme-platform/<repo>`
4. Branch: `main`
5. Auto-sync: ON

| Repo | What syncs into Harness |
|---|---|
| `pe-pipeline-config` | (no Harness entities; this repo is *consumed* by the parser at pipeline runtime) |
| `pe-pipeline-templates` | All template YAMLs in `templates/` — pipelines, stages, steps |
| `pe-base-images` | The pipeline at `.harness/pipeline.yaml` |
| `pe-policies` | Rego files + the policy set in `harness/policy-sets/` |
| `pe-idp-software-templates` | The IDP scaffolder template |

- [ ] All 5 sources connected
- [ ] Templates appear under Account → Templates
- [ ] Pipeline pe_base_images appears under platform_engineering / base_images project
- [ ] Policies appear under Account → Policy Engine
- [ ] IDP template appears under Account → IDP → Software Templates

---

## 12. Pre-create namespaces and ArgoCD Applications for BWR

ArgoCD needs an Application registered for each (app, env, region) tuple. Start with `bwr-web/dev/us-east-1`.

```bash
kubectl create namespace bwr-web-dev
```

In Harness UI: GitOps → Applications → **+ New Application**:

- Name: `bwr-web-dev-us-east-1` (matches the convention in `gitops-sync-step.yaml`)
- Source repo: `acme-platform/team-bwr-gitops`
- Source path: `bwr-web/manifests/dev/us-east-1`
- Destination cluster: `kind_harness_lab`
- Destination namespace: `bwr-web-dev`
- Sync policy: Automated, Prune, SelfHeal

- [ ] Namespace `bwr-web-dev` created
- [ ] ArgoCD Application `bwr-web-dev-us-east-1` registered (will show `Missing` until first CD run)

---

## 13. First BWR CI run

Connect the BWR app repo and trigger a build.

1. Harness UI → domain_bwr Org → bwr_web Project → Pipelines
2. The pipeline `bwr-web-ci` should auto-appear from the `.harness/pipeline.yaml` we committed in `team-bwr-web`
3. **Run pipeline** with default inputs (`env: dev`, `region: us-east-1`)
4. Watch each step group complete:
   - resolve-config — clones the two config repos, runs config_parser
   - source-and-test — Maven build, JUnit
   - security-scan — STO Semgrep
   - package-and-publish — Build → Trivy → SSCA (Cosign sign + SBOM) → Push

**Expected end state:**
- Image at `docker.io/acme-platform/bwr-web:<commit-sha>`, signed
- SBOM attached as Cosign attestation
- Pipeline UI shows green for every step

- [ ] CI pipeline ran end-to-end
- [ ] Image visible on Docker Hub
- [ ] Signature verifies: `cosign verify --key ~/.harness-lab/secrets/cosign.pub docker.io/acme-platform/bwr-web:<sha>`

---

## 14. First BWR CD run

The first deploy through the full CD chain.

1. Harness UI → domain_bwr Org → bwr_web Project → Pipelines → `bwr-web-cd`
2. Run with inputs:
   - `env`: `dev`
   - `region`: `us-east-1`
   - `appImage`: `docker.io/acme-platform/bwr-web:<sha-from-step-13>`
   - `smokeTestEndpoint`: `http://localhost:8080/healthz` (kind exposes ingress on host port 8080)
3. Watch step groups:
   - resolve-config (same parse_config step)
   - verify-supply-chain (Cosign verify via SSCA Enforcement)
   - render-and-publish (helm template + commit to team-bwr-gitops)
   - sync-and-confirm (ArgoCD sync, wait for healthy, smoke test)

**Expected end state:**
- New commit in `acme-platform/team-bwr-gitops` on path `bwr-web/manifests/dev/us-east-1/`
- ArgoCD Application `bwr-web-dev-us-east-1` shows `Synced` and `Healthy`
- BWR pod running in namespace `bwr-web-dev`
- `curl http://localhost:8080/healthz` returns `ok`

- [ ] CD pipeline ran end-to-end
- [ ] Manifests committed to gitops repo
- [ ] ArgoCD synced + healthy
- [ ] Smoke test passes

---

## 15. Engineer the OPA policy block (governance verification)

A pipeline blocked by policy — the governance layer in action.

```bash
cd /Users/jwalit/Documents/harness/repos/team-bwr-web
git checkout -b break-base-image
sed -i '' 's|^FROM ${BASE_IMAGE}|FROM openjdk:21-slim|' Dockerfile
git diff Dockerfile
git add Dockerfile && git -c commit.gpgsign=false commit -m "Try non-platform base image (will fail)"
git push -u origin break-base-image
gh pr create --title "Try non-platform base image" --body "Should be blocked by OPA policy"
```

Then run the CI pipeline against this branch.

**Expected:**
- Pipeline run starts
- OPA policy evaluation fails the run with a message pointing at `pe-policies/harness/require-platform-base-image.rego`
- Build never produces an image

```bash
# After the screenshot, undo the breakage
git checkout main
git push origin --delete break-base-image
gh pr close <pr-number>
```

- [ ] Branch with hardcoded FROM created
- [ ] Pipeline run blocked by OPA policy
- [ ] Branch cleaned up

---

## 16. Config-only restart (the 12-factor proof point)

```bash
cd /Users/jwalit/Documents/harness/repos/team-bwr-pipeline-config
sed -i '' 's/BWR_DB_POOL_SIZE: "10"/BWR_DB_POOL_SIZE: "20"/' applications/bwr-web/environments/dev.yaml
git diff
git add . && git -c commit.gpgsign=false commit -m "Bump BWR_DB_POOL_SIZE 10 -> 20"
git push
```

Then run the BWR CD pipeline (CI is NOT triggered).

**Expected:**
- CD pipeline runs the full chain
- New commit in team-bwr-gitops with only the ConfigMap's `data.BWR_DB_POOL_SIZE` and the Deployment's `checksum/config` annotation changed
- ArgoCD detects the diff
- Pods rolling-restart (logs show `application-override.yaml` re-read)
- Container image: bit-identical to before
- `kubectl describe deploy bwr-web` before and after shows the `Image:` field unchanged but the pod-template `checksum/config` annotation changed and pods cycled

- [ ] Config change committed (no CI run)
- [ ] CD pipeline ran with image unchanged
- [ ] Pods rolling-restarted with new env vars

---

## 17. Tear down

When the lab is no longer needed.

```bash
kind delete cluster --name harness-lab
# Harness: Account Settings → Delegates → delete lab-delegate
# Harness: GitOps → Agents → delete bwr-gitops-agent
```

Repos and Harness account state can stay (or be cleaned up in Harness Account Settings if not on a free trial).

- [ ] kind cluster deleted
- [ ] Delegate removed
- [ ] GitOps Agent removed

---

## Troubleshooting cheat sheet

| Symptom | Likely cause |
|---|---|
| Pipeline can't clone domain repos | `github_acme_platform_pat` doesn't have read access to private repos |
| `commit-manifests` fails to push | `github_gitops_pat` doesn't have write on `team-*-gitops` |
| ArgoCD shows `Unknown` health | Argo Rollouts CRDs not installed (canary path); fall back to rolling for first run |
| `verify-image-signature` fails | `cosign_public_key` value doesn't match the key used to sign |
| Pod stuck `CreateContainerConfigError` | Domain config references a Secret (`app.secrets.X`) but the Secret doesn't exist in the namespace yet |
| Helm render fails on `app.appImage` empty | parse-config step didn't run before render-manifests; check stage execution order |

Each of these has a "what changed" story worth recording in the platform team's operations notes if it's not already there — operational documentation for the next person reproducing.
