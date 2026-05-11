# Platform Engineering Lab — Documentation

A platform-engineering reference implementation on Harness. Two teams modelled: a Platform Engineering team owning templates, base images, governance policies, and self-service onboarding; a Domain team (Baby & Wedding Registry — "BWR") consuming the platform with thin, declarative pipeline wrappers.

The lab runs end-to-end on a single-node kind cluster against a real Harness account, with signed images, hierarchical config, GitOps-native CD, Argo Rollouts canary in prod, and OPA Rego policies attached to every pipeline event.

**Status:** live as of 2026-05-11.

> **Placeholder names used throughout these docs:** `acme-platform` stands in for the GitHub org and Docker Hub namespace; `<HARNESS_ACCOUNT_ID>` stands in for the Harness account ID. Substitute with your own identifiers when reproducing.

---

## Documents

| Doc | Read for |
|---|---|
| [`01-developer-flow.md`](01-developer-flow.md) | Six end-user scenarios from a domain developer's perspective. Useful for onboarding new domain engineers and for walking a reviewer through the platform. |
| [`02-architecture.md`](02-architecture.md) | Comprehensive system architecture: tenant topology, repo map, cluster layout, supply-chain flow, CD flow, canary mechanism, config resolution, policy enforcement, decisions log, failure-mode catalogue, operational walkthrough. |
| [`03-runbook.md`](03-runbook.md) | Step-by-step reproduction: from empty Harness account to live BWR pipelines. Each step has commands, expected output, and a checkbox. |

**Suggested reading order:** `01` → `02` → `03`. The developer-flow doc primes the platform narrative; the architecture doc backs every claim with a diagram; the runbook reproduces the setup.

---

## Tenant references

| External system | Identifier |
|---|---|
| Harness account | `<HARNESS_ACCOUNT_ID>` |
| GitHub org | [`acme-platform`](https://github.com/acme-platform) |
| Docker Hub namespace | [`acme-platform`](https://hub.docker.com/u/acme-platform) |

---

## Repository inventory

Platform repos (platform engineering team owns):

| Repo | Contents |
|---|---|
| `pe-pipeline-config` | Hierarchical config layers + `schema/exposed-values.yaml` contract + `lib/config_parser.sh` resolver |
| `pe-pipeline-templates` | 3 pipeline templates, 3 stage templates, ~12 step templates |
| `pe-helm-charts` | Helm `microservice` chart published as a Cosign-signed OCI artifact at `acme-platform/microservice:<semver>` |
| `pe-base-images` | Hardened Dockerfiles built nightly, Cosign-signed, pushed as `acme-platform/base-<lang>-<version>` |
| `pe-policies` | 5 Rego policies + 27/27 unit tests + policy-set definition |
| `pe-idp-software-templates` | Backstage software templates for IDP self-service onboarding |

Domain repos (BWR team owns):

| Repo | Contents |
|---|---|
| `team-bwr-web` | Spring Boot app code; multi-stage Dockerfile; `.harness/{pipeline,cd-pipeline}.yaml` thin wrappers (~45 lines each) |
| `team-bwr-pipeline-config` | `applications/bwr-web/{application,environments/*}.yaml` — domain config layer |
| `team-bwr-gitops` | Auto-written manifest tree under `bwr-web/manifests/<env>/<region>/`. Read-only to humans (ArgoCD source of truth). |

---

## What's enforced

- **Base image must come from `docker.io/acme-platform/base-*`** — OPA `require-platform-base-image` (onrun, hard-deny)
- **Domain pipelines must reference a platform pipeline template** — OPA `require-platform-pipeline-template` (onsave, hard-deny)
- **Image must be signed and SBOM-attested before any deploy** — `verify_image_signature_inline` step in `gitops_deploy_stage` (hard-fail) + OPA `require-image-signature-verification` (onrun, hard-deny)
- **Helm chart version pinned to semver** — OPA `require-pinned-chart-version` (onsave, hard-deny). No `latest`, no branch names.
- **Disabling a security gate requires a ticket reference** — OPA `disable-scan-needs-justification` (onrun, hard-deny)
- **Domain config schema-validated against `pe-pipeline-config/schema/exposed-values.yaml`** — config parser hard-fails on unknown or non-overridable keys
- **Deploy strategy is platform-controlled per env** — `app.deployStrategy` is not in the schema's `overridable:` list; prod gets canary via Argo Rollouts regardless of what domain wants
