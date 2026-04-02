---
name: CI/CD Pipeline and Deploy Workflow
description: GitHub Actions ci.yml, release.yml, deploy.yml structure, OIDC auth, release-please config, rulesets
type: reference
---

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | push to main/staging/dev + all PRs | lint, check-types, format:check, test, build, security |
| `release.yml` | push to main/dev/staging | release-please creates version PRs and tags |
| `deploy.yml` | `v*` tag push | security audit + docker build x14 + Cloud Run deploy + gateway + health check |
| `claude-code-review.yml` | PR opened/synced | AI code review via Claude Code Action |
| `pr-gate.yml` | PR opened to main/staging | Auto-closes PRs from non-admins |

## CI Jobs (ci.yml)

- `validate` (matrix): lint, check-types, format:check — parallel
- `test`: bun run test
- `test-coverage`: only on PRs to main/staging + Codecov upload
- `build`: bun run build — after validate
- `api-gateway-lint`: independent
- `security`: Semgrep SAST — only on PRs to main/staging

All jobs run `bun run --cwd packages/database db:generate` before main command.

## Release-Please

| Branch | Config | Prerelease | Tag example |
|---|---|---|---|
| dev | release-please-config-dev.json | beta | v0.1.1-beta.1 |
| staging | release-please-config-staging.json | rc | v0.1.1-rc.1 |
| main | release-please-config.json | none | v0.1.1 |

All configs: `component: ""`, `include-component-in-tag: false` (no `eqp-platform-` prefix).
Manifest: `.release-please-manifest.json` (current: `0.1.0-beta.0`).
Uses `RELEASE_PLEASE_TOKEN` (classic PAT with `repo` scope, authorized for EQ-Pay-Solutions org) — enterprise blocks GITHUB_TOKEN from creating PRs.

## Deploy Pipeline (deploy.yml)

Triggers on `v*` tag push. Environment detection:
- `v*-beta.*` → eq-envs (dev)
- `v*-*` (non-beta) → eq-envs-stg (staging)
- `v*` (clean semver) → eq-envs-prod (prod)

Steps: resolve-env → security → docker build (parallel matrix 14 services) → deploy (sequential, tracks failures) → deploy-gateway (ESPv2) → health-check

## GCP Authentication (OIDC)

All deploy.yml auth uses Workload Identity Federation:
```yaml
workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}
```
Each auth job has `permissions: { contents: read, id-token: write }`.

Pool: `eqpv2-github`, Provider: `eqpv2-github`, Project: `cips-test` (all envs share one GCP project for now).
Setup script: `scripts-local/setup-gcp-oidc.sh`

## Rulesets (6 active on GitHub)

| Ruleset | Key rules |
|---|---|
| main-protection | 2 approvals, CODEOWNERS, lint+test+check-types+format:check+build |
| staging-protection | 1 approval, same CI checks |
| dev-protection | 0 approvals, lint+check-types+format:check |
| tag-protection | Admin-only create/update/delete all tags |
| branch-naming | feat/fix/chore/docs/refactor/test/hotfix/release pattern |
| commit-message | Conventional commits |

Import via: `gh api -X POST repos/EQ-Pay-Solutions/equitipay-apps/rulesets --input .github/rulesets/{file}.json`
`file-path-restrictions.json` is Enterprise-only — manual import required.

## .prettierignore

`CHANGELOG.md` and `.release-please-manifest.json` are ignored (release-please generated files).
