---
name: GitHub Repository Rulesets
description: Branch/tag/PR protection rulesets, OIDC auth, release-please setup, repo URLs
type: reference
---

## Repo Settings

- **Squash merge only** — merge commits and rebase disabled
- **Auto-delete branches** after merge
- **PR title as commit message**
- **Actions workflow permissions**: read-only (enterprise restriction — can't write PRs with GITHUB_TOKEN)

## Active Rulesets (6, managed via API)

| Ruleset | Target | Key rules |
|---|---|---|
| main-protection | main | 2 approvals, CODEOWNERS, lint+test+check-types+format:check+build, linear history, signed commits |
| staging-protection | staging | 1 approval, lint+test+check-types+format:check+build, linear history |
| dev-protection | dev | 0 approvals, lint+check-types+format:check |
| tag-protection | All tags | Admin-only create/update/delete |
| branch-naming | All (excl main/staging/dev/release-please--*) | `^(feat\|fix\|chore\|docs\|refactor\|test\|hotfix\|release)/.+$` |
| commit-message | All branches | Conventional commits |

All bypass: `RepositoryRole: 5` (Admin).

## Manage rulesets

```bash
# List
gh api repos/EQ-Pay-Solutions/equitipay-apps/rulesets --jq '.[] | "\(.id) \(.name)"'

# Delete + re-import
gh api -X DELETE repos/EQ-Pay-Solutions/equitipay-apps/rulesets/{ID}
gh api -X POST repos/EQ-Pay-Solutions/equitipay-apps/rulesets --input .github/rulesets/{file}.json

# Temporarily disable for force-push
gh api -X PUT repos/EQ-Pay-Solutions/equitipay-apps/rulesets/{ID} --input <(echo '{"enforcement":"disabled"}')
```

## Not importable via API

`file-path-restrictions.json` — Enterprise-only (`file_path_restriction`, `max_file_size` rules). Import manually via Settings > Rules > Rulesets.

## OIDC Auth (GCP Workload Identity Federation)

GitHub vars (repo-level, all envs share same GCP project):
- `GCP_WORKLOAD_IDENTITY_PROVIDER`: `projects/857138853031/locations/global/workloadIdentityPools/eqpv2-github/providers/eqpv2-github`
- `GCP_SERVICE_ACCOUNT`: `deploy@cips-test.iam.gserviceaccount.com`

Setup: `scripts-local/setup-gcp-oidc.sh`

## Release-Please

Uses `RELEASE_PLEASE_TOKEN` (classic PAT, `repo` scope, authorized for EQ-Pay-Solutions org).
Enterprise blocks GITHUB_TOKEN from creating PRs → PAT required.

## Repo URLs

- **Org repo:** `EQ-Pay-Solutions/equitipay-apps` (origin)
