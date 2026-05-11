---
name: release-please owns v* tagging — never push tags manually
description: Versioning on equitipay-apps is automated by release-please. Manual `git tag v*` + push is wrong — merge the bot's open PR instead. Release-please owns the entire `v*` namespace.
type: reference
---

**Never run `git tag v*-beta.x dev && git push origin v*-beta.x` (or any other `v*` tag) manually on this repo.** Release-please owns the entire `v*` namespace and the rulesets enforce it.

## How tagging actually flows

| Branch event | What release-please does | Result |
|---|---|---|
| Push to `dev` | Maintains an open PR titled `chore(dev): release 0.1.X-beta.0` with bumped version + CHANGELOG | Merge the PR → bot auto-creates the matching `v0.1.X-beta.0` tag on origin → Cloud Build trigger `eqp-dev-tag-push` fires on `^v.*$` → image build → Cloud Deploy rollout |
| PR merge to `staging` | Same flow, suffix `-rc.X` | Same chain, `eqp-staging-tag-push` fires → eq-envs-stg deploy |
| PR merge to `main` | Same flow, clean semver | Same chain, `eqp-prod-tag-push` fires → eq-envs-prod deploy (canary, requires approval) |

Three release-please configs gate the three branches:
- `release-please-config-dev.json` → beta pre-release
- `release-please-config-staging.json` → rc pre-release
- `release-please-config.json` → clean semver

Each commits to its own `.release-please-manifest.json`. All three configs use `component: ""`, `include-component-in-tag: false` (no `eqp-platform-` prefix).

## Why manual tagging is broken

1. **Picks the wrong version** — release-please's open PR is the source of truth for the next version. Computing it yourself ("v0.1.16 looks right") doesn't match the bot's bookkeeping and the bot will overwrite the manifest on its next run.
2. **Bypasses CHANGELOG + version bump** that release-please does as part of its PR.
3. **GitHub ruleset rejects the push** with `Cannot create ref due to creations being restricted` — the `tag-protection` ruleset (`.github/rulesets/tag-protection.json`) restricts `v*` creation to the release-please PAT and Admin.

## What to do instead

When you need to fire a Cloud Build / Cloud Deploy run, merge the open release-please PR for that branch:

```bash
gh pr list --search "release-please" --state open
gh pr merge <PR#> --squash --auto    # or via the UI
```

The bot auto-creates the tag on origin within ~30s of the PR merge. The Cloud Build trigger reacts on `^v.*$`.

If the release-please PR is **empty or stale** (no commits since the last release), there's nothing to deploy — adding a manual tag would just re-deploy the same image digest. Ship more commits to `dev` first, then let the bot reopen its PR.

## PAT

`RELEASE_PLEASE_TOKEN` — classic PAT with `repo` scope, authorized for the EQ-Pay-Solutions org. Enterprise SSO blocks the default `GITHUB_TOKEN` from creating PRs from a workflow.

## When tag rulesets can be temporarily relaxed

`tag-protection` ruleset can be flipped to `enforcement: disabled` for a brief window via:
```bash
gh api -X PUT repos/EQ-Pay-Solutions/equitipay-apps/rulesets/<ID> --input <(echo '{"enforcement":"disabled"}')
```
ONLY for documented emergency / force-push recovery — and re-enabled immediately after. NEVER push manual tags during the window.
