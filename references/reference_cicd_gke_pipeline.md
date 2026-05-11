---
name: CI/CD Pipeline (GKE era)
description: Tag-based deploy pipeline — release-please tags `v*` → Cloud Build trigger fires `cloudbuild-images.yaml` on the private worker pool (BuildKit/docker buildx) → Artifact Registry (immutable) → BinAuthz sign → Cloud Deploy render+rollout to GKE+ASM. GitHub Actions runs lint/test only.
type: reference
---

## 1. GitHub Actions remit

`.github/workflows/ci.yml` runs ONLY: `lint`, `check-types`, `format:check` (matrix `validate` job), `test`, `test-coverage` (on PRs targeting main/staging), `build`, `security` (Semgrep on PRs to main/staging), `route-parity` (OpenAPI vs ASM VirtualService on every PR), and `hooks-lint-test` (shellcheck + bash test suite for Claude Code hooks).

Every Bun-based job MUST run `bun run --cwd packages/database db:generate` after `bun install --frozen-lockfile` — `--ignore-scripts` skips Prisma's postinstall. See `reference_cloud_sql_prisma.md` §9.

`.github/workflows/release.yml` triggers release-please. **`.github/workflows/deploy.yml` was DELETED in Plan 5** — the legacy Cloud Run deploy path is gone. All deploys now live in Google Cloud (Cloud Build + Cloud Deploy).

## 2. release-please

Owns the `v*` tag namespace. **Never push tags manually.** Three configs:

- `release-please-config-dev.json` → `v*-beta.*` (pre-release tags on push to `dev`)
- `release-please-config-staging.json` → `v*-rc.*` (release-candidate tags on PR merge to `staging`)
- `release-please-config.json` → clean semver `v*` on merge to `main`

PAT: `RELEASE_PLEASE_TOKEN` with `repo` scope, EQ-Pay-Solutions org authorized. Enterprise SSO blocks the default `GITHUB_TOKEN` — must use the PAT.

## 3. Tag → environment

| Tag pattern | Environment | Project (current → planned) |
|---|---|---|
| `v*-beta.*` | dev | cips-test → eq-envs |
| `v*-rc.*` | staging | eq-envs-stg |
| `v*` | prod | eq-envs-prod |

Pattern resolution lives in `cloudbuild-images.yaml` step `resolve-env` via grep -qP regex on `$TAG_NAME`. Unknown patterns fail the build at step 1.

## 4. Cloud Build trigger (`infra/terraform/modules/cicd-pipeline/main.tf`)

`<name_prefix>-tag-push` — TF-managed `google_cloudbuild_trigger`. Reacts to `repository_event_config.push.tag = "^v.*$"`. Runs `infra/cloudbuild-images.yaml` on the private worker pool. Substitutions injected from TF: `_ARTIFACT_REGISTRY`, `_ARTIFACT_REGISTRY_CACHE`, `_DEPLOY_PIPELINE`, `_DEPLOY_REGION`, `_WORKER_POOL`, `_NAME_PREFIX`.

Trigger SA = `eqp-<env>-cloudbuild@<project>.iam.gserviceaccount.com`. Roles bound by the TF module: `artifactregistry.writer`, `clouddeploy.releaser`, `clouddeploy.jobRunner`, `cloudbuild.builds.builder`, `logging.logWriter`, `storage.objectAdmin`, `secretmanager.secretAccessor`, **`container.developer`** (Cloud Deploy's deploy-job needs `container.clusters.get` to fetch GKE creds — discovered in build `d579fec4` on cips-test 2026-04-29).

## 5. Image build — BuildKit / docker buildx

Migrated from Kaniko in PR #171. Each of the 12 build steps (`build-iam-service` ... `build-business-service` + `build-eqp-portal`):

```
docker buildx create --use --driver docker-container --bootstrap
docker buildx build \
  --platform=linux/amd64 \
  --build-arg=SERVICE_NAME=<svc> --build-arg=HTTP_PORT=... --build-arg=GRPC_PORT=... \
  --cache-from=type=registry,ref=${_ARTIFACT_REGISTRY_CACHE}/<svc>-cache:buildkit \
  --cache-to=type=registry,ref=${_ARTIFACT_REGISTRY_CACHE}/<svc>-cache:buildkit,mode=max \
  --tag=${_ARTIFACT_REGISTRY}/<svc>:${TAG_NAME} \
  --file=docker/Dockerfile.nestjs \
  --push .
```

- Ephemeral `docker-container` builder per step — required for registry-backed `cache-to` AND Dockerfile cache mounts (the default `docker` driver supports neither).
- Cache key = `:buildkit` tag in `*-cache` AR repo, shared across services and across runs.
- `mode=max` writes ALL intermediate layers to cache (not just the final image), so warm builds skip most of `bun install`, `turbo build`, and final stage repacks.
- Dockerfiles use `RUN --mount=type=cache,...` for bun's install cache and turbo's task cache; both require `# syntax=docker/dockerfile:1.6` header.
- Worker pool is `e2-highcpu-32` since PR #167 (cold-build wall time expectation: <=20min cold, <=10min warm — Kaniko baseline was 49min cold on e2-standard-4).
- Build timeout `3600s` (PR #74).
- 11 service builds use `docker/Dockerfile.nestjs` with build-args. `eqp-portal` uses `apps/eqp-portal/Dockerfile` with NEXT_PUBLIC_FIREBASE_* args injected at build time from Secret Manager (Next.js inlines them into the JS bundle).

## 6. Worker pool egress

`infra/terraform/modules/cloudbuild-worker-pool/main.tf` — has TWO subtleties:

1. **`lifecycle.ignore_changes = [worker_config[0].no_external_ip]`** + a `null_resource.set_public_egress` that runs `gcloud builds worker-pools update --public-egress`. Why: the hashicorp/google v7.x provider doesn't surface `egress_option` in the schema. Default is `NO_PUBLIC_EGRESS`, which forces egress through the peered VPC — but Cloud NAT doesn't cover Service-Networking-peered ranges, so github.com is unreachable and FETCHSOURCE hangs on port 443 timeout. The `--public-egress` gcloud convenience flag flips both `egress_option=PUBLIC_EGRESS` and `no_external_ip=false` in one call. Codified in PR #80.
2. **`null_resource.triggers` includes `machine_type` and `disk_size_gb`** (PR #169). The GCP Cloud Build API "does not use field masks" on PUT — any in-place `worker_config` update silently resets `egress_option` to `NO_PUBLIC_EGRESS`. So machine_type bumps must re-fire the gcloud flip.

PCI scope: build infrastructure is out of PCI scope (cardholder data only touched at runtime). Egress IP being a random Google IP rather than the VPC NAT is fine — nothing in `cloudbuild-images.yaml` calls a partner API.

## 7. Artifact Registry

Two repos:

- **`eqp-platform`** — immutable tags (`immutable_tags = true`), holds the 12 service images. Immutability is the BinAuthz signing guarantee.
- **`eqp-platform-cache`** — MUTABLE, holds the per-service `:buildkit` cache tags. BuildKit must overwrite `:buildkit` every run. Created as separate repo in PR #181.

Optional CMEK via `var.enable_ar_cmek` on the platform module. `_ARTIFACT_REGISTRY_CACHE` substitution defaults to the cips-test cache repo for dev; staging/prod triggers MUST override.

## 8. BinAuthz signing

Step `sign-images` in `cloudbuild-images.yaml`:

```
DIGEST=$(gcloud artifacts docker images describe \
  "${_ARTIFACT_REGISTRY}/$SVC:${TAG_NAME}" \
  --format='value(image_summary.digest)')
gcloud beta container binauthz attestations sign-and-create \
  --artifact-url="${_ARTIFACT_REGISTRY}/$SVC@$DIGEST" \
  --attestor="${_NAME_PREFIX}-attestor" \
  --keyversion-keyring="${_NAME_PREFIX}-pci" \
  --keyversion-key="binauthz-signer" \
  --keyversion=1 \
  --validate
```

Cloud Build SA needs `roles/binaryauthorization.attestorsVerifier` on the attestor (PR #78 fix — `attestorsViewer` was insufficient; that role grants read-only, signing needs verifier). KMS key version pinned at 1; rotation is a separate runbook per `kms-pci/main.tf` comment.

Plus `roles/iam.serviceAccountUser` on the SA itself (`google_service_account_iam_member.cloudbuild_self_act_as`) — Cloud Deploy requires the caller of `releases create` to hold serviceAccountUser on the SA referenced in the target's `executionConfigs.serviceAccount` (PR #82). Plus the Cloud Deploy service agent needs to act-as the cloudbuild SA (`google_service_account_iam_member.clouddeploy_act_as`).

## 9. Cloud Deploy (`infra/k8s/skaffold.yaml`)

- Skaffold v4beta11 config — entrypoint for `gcloud deploy releases create`. Forward-declares 12 image artifacts so the `--images` flag from Cloud Build can rewrite Deployment manifests with digest-pinned refs.
- Profiles: `dev`/`staging`/`prod`, each pointing at `overlays/<env>` via kustomize.
- `requires` block pulls in per-env `verify-deploy.yaml` keyed by `activeProfiles.activatedBy`. Without this, Cloud Deploy renders fail with `VERIFICATION_CONFIG_NOT_FOUND` when `strategy.standard.verify=true` (PR #92 — verify configs MUST be scoped inside profiles).
- Container names must be unique inside `verify-deploy.yaml` (PR #90).
- TF `google_clouddeploy_delivery_pipeline.main` sequences: dev (auto-promote, `strategy.standard.verify=true`) → staging (auto-promote, same) → prod (`require_approval=true`, canary 10%→50%→100% with verify between phases — hardcoded to `iam-service` for Plan 3b; Plan 3c generalizes).

## 10. Private-cluster connectivity

Cloud Deploy must reach the cluster's PRIVATE control plane. The canonical fix (PR #106) is **GKE DNS-based endpoint**:

```hcl
google_clouddeploy_target.dev.gke {
  cluster      = var.gke_dev_cluster_id
  dns_endpoint = true
}
```

The cluster's `dns_endpoint_config.allow_external_traffic` (set in `module.gke_autopilot`) lets the worker pool resolve and reach the FQDN directly.

What failed before:

- **PR #96** — `internal_ip=true` via VPC peering — fails, peering not transitive.
- **PRs #98 + #100 + #102** — public endpoint with `gcp_public_cidrs_access_enabled` — fails, Cloud Build PRIVATE pool egress is NOT classified as GCE Public IPs. **Do NOT add the worker-pool peered range to `authorized_networks`** (PR #102 regression — it doesn't work and looks like progress).

Cluster must have `gcp_public_cidrs_access_enabled = true` and `private_endpoint_enforcement = false` for the DNS endpoint path.

## 11. Digest pinning

Cloud Deploy releases use digest-pinned image refs from the build step (PR #112). The `cloud-deploy-release` step resolves each `<svc>:${TAG_NAME}` to its digest via `gcloud artifacts docker images describe`, then passes `<svc>=<registry>/<svc>@sha256:...` entries to `--images=`. BinAuthz attestations are tied to digests (not tags), so passing tag-based refs causes admission failure:

```
denied by attestor: Expected digest with sha256 scheme,
but got tag or malformed digest
```

Skaffold rewrites Deployment.spec.containers[].image fields that match the declared artifact names to these digest refs.

## 12. First-build defects fixed

35+ infrastructure defects between PR #69 and #115 (cumulative count reached 37+ across PRs #69 → #130). Catalogued in `packages/docs/Operations/gke-cutover-runbook.md` Appendix A. Examples surfaced above: #74 timeout, #78 attestor role, #80 worker-pool egress, #82 self-actAs, #90 verify container uniqueness, #92 verify profile scoping, #96/98/100/102/106 control-plane reachability, #112 digest pinning, #119 Cloud SQL, #123 baseline migration, #150/158/160 Prisma generate, #167 machine type, #169 worker_config triggers, #171 BuildKit migration, #181 cache repo split. Treat the runbook as the bring-up checklist for any new env.
