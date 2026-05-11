---
name: Per-Org API Keys
description: Per-organization API keys (X-API-Key header) validated via gRPC to iam-service. Argon2id + on-disk pepper, Redis hot-path cache, Pub/Sub→BigQuery audit, per-org fixed-window rate limit. Mutually exclusive with Firebase JWT auth (X-API-Key OR Authorization: Bearer, both → 400).
type: reference
---

## 1. Why

The Plan-5 GKE migration ported ESPv2's API-key validation as a hand-rolled Lua EnvoyFilter on the IngressGateway with a **single shared secret**. ESPv2's `servicecontrol.googleapis.com.Check()` did per-consumer key validation, IAM-level audit, and per-key quotas. The Lua port lost all of that:

- One secret = full edge access for every client (no per-consumer scoping).
- No GCP IAM/audit attribution per call.
- No quotas / rate limits.
- No rotation story — rotating the shared secret simultaneously breaks every client.
- Operator-managed `terraform.tfvars` for the secret value (manual, error-prone, also blocks `terraform apply` from worktrees).

The portal also shipped `API_GATEWAY_KEY` as a server-side env (sourced from GCP Secret Manager `gateway-api-key` via CSI mount). That credential being in the portal pod pulled the web tier into the **PCI-credential boundary**.

The per-org API-keys system replaces this with credentials each organization owns, rotates, and audits independently — restoring proper credential management **and** reducing PCI scope (the portal no longer holds any backend credential; Firebase JWT is per-user, not a static secret the tier "holds").

A sibling plan `docs/superpowers/plans/2026-05-05-drop-edge-api-key.md` proposed deleting the X-API-Key concept entirely (JWT-only at the edge). That plan was **shelved**: per-org keys retain the server-to-server flow that merchant integrations need, with the proper IAM/audit/rotation semantics. The per-org plan supersedes drop-edge-api-key.

## 2. Architecture (A0 = (b): gRPC validator at iam-service)

**Single source of truth:** the Prisma `ApiKey` table + argon2id hash + on-disk pepper file all live in `iam-service` only.

New gRPC contract (`packages/protos/src/api_key.proto`, package `api_key`):

```proto
service ApiKeyValidatorService {
  rpc Validate (ValidateRequest) returns (ValidateResponse);  // mTLS-required (in-mesh only)
}
```

The 10 non-iam services inject a thin `ApiKeyValidatorClient` (gRPC wrapper at `packages/api/src/auth/api-key/api-key-validator.client.ts`) as the `API_KEY_VALIDATOR` DI token (a `Symbol`). iam-service binds the same token to `ApiKeyValidatorInProcess` (`apps/iam-service/src/api-keys/api-key-validator.in-process.ts`) — same `ApiKeyValidatorPort` interface, but calls the Prisma repo and argon2 directly without a gRPC hop.

**Consequences of A0=(b):**

- `@repo/api` does **not** depend on `@node-rs/argon2` (only iam-service does).
- `@repo/api` does **not** depend on `@repo/database` (Prisma client lives only in iam-service).
- 10 of 11 services do **not** mount the pepper file or carry the pepper rotation runbook.
- Single place to audit, single place to rotate the pepper.
- Trade-off accepted: +5–10ms per cache miss vs in-process Prisma. Mitigated by the Redis cache in the guard layer.

## 3. Guard chain

`ApiKeyGuard` (`packages/api/src/auth/api-key/api-key.guard.ts`) is the **first** of seven global guards on every NestJS service:

```
ApiKeyGuard → TokenGuard → DeviceGuard → UserGuard → PermissionGuard → RoleScopeGuard → ScopeGuard
```

Wired in every service's `app.module.ts` as `APP_GUARD`, **before** the existing six. Logic:

- Header absent or wrong prefix → `return true`; the JWT path runs as today.
- Header present + valid → set `req.apiKey`; `TokenGuard` sees `req.apiKey` and short-circuits (no JWT required).
- Header present + invalid → `401 invalid_key`.
- Header present **and** `Authorization: Bearer …` present → `400` with `AUTH_ERRORS.MIXED_AUTH = 'auth_000011'` (request-shape error, not auth failure; both paths can't be merged into a single context).
- gRPC/RPC executions (`context.getType() !== 'http'`) → short-circuit `true`. Inter-service gRPC trust is enforced by ASM mTLS STRICT + controller-level guards on `@GrpcMethod` handlers.

`PermissionGuard` consults `req.apiKey.scopes` (array `.some()` lookup) instead of the user profile (object `profile[resource][permission]` lookup). Same `(resource, permission ∈ C|R|U|D)` vocabulary, different storage shape.

`ScopeGuard`: reads `ownerId` from `req.query['ownerId']` as today, falling back to `req.apiKey.firestoreOwnerId` if absent. API-key requests skip the membership check (membership was bound at issuance).

## 4. Hot-path caching

Cache key is `apikey:v1:<sha256(raw_secret)>` — sha256 of the **raw secret**, not the prefix. Prefix-only caching is a security regression: an attacker presenting a different secret with the same prefix would hit the cached record and bypass argon2. The sha256-of-raw cache hits only an exact already-validated input.

Flow:

1. Redis `get(apikey:v1:sha256(raw))`.
2. Hit (positive) → use cached `ApiKeyContext`; no DB / argon2 / network.
3. Hit (negative, `ctx: null`) → throw `UnauthorizedException('invalid_key')`.
4. Miss → 500ms-deadline gRPC `Validate(secret)` → iam-service does prefix lookup + argon2 verify + status check.
5. Positive result → cache 5 min (`API_KEY_CACHE_TTL_S = 300`).
6. Definitive failure (`invalid_key` / `revoked` / `expired`) → negative-cache 30s (`API_KEY_NEGATIVE_CACHE_TTL_S = 30`) to absorb scrapers.
7. Transient gRPC failure (`UNAVAILABLE` / `DEADLINE_EXCEEDED` / RxJS timeout) → `503 auth_dependency_unavailable`. **Cache is NOT poisoned** on transient outage; fail-CLOSED on auth, same posture as Firebase Auth being down.

Inside `ApiKeyValidatorInProcess.validate()`:

1. Prefix lookup (indexed `WHERE prefix = ? AND status IN ('ACTIVE','ROTATING')`).
2. Cheap status/expiry assert **before** argon2 (revoked → emit `revoked_validation_attempt` audit; expired → emit `expired` audit).
3. Argon2id verify against `pepper[record.pepperVersion]` (~50ms).
4. Fire-and-forget `touchLastUsed(id)` (debounced caller-side, never blocks).
5. Emit `validated_ok` audit; return slim `ApiKeyContext` (never echoes `hashedSecret` or `pepperVersion`).

## 5. `req.apiKey` shape

```ts
interface ApiKeyContext {
  keyId: string;
  orgId: string;            // Postgres Business.id
  firestoreOwnerId: string; // Firestore retailer/owner doc id (cutover bridge)
  scopes: { resource: string; permissions: ('C'|'R'|'U'|'D')[] }[];
  environment: 'live' | 'test';
  rateLimitRpm: number | null;
}
```

`PermissionGuard` / `ScopeGuard` consult `req.apiKey.scopes` and `req.apiKey.firestoreOwnerId` instead of the user profile. The `firestoreOwnerId` field bridges the Firestore→Postgres cutover: `ApiKey` rows carry **both** `business_id` (Postgres FK) and `firestore_owner_id` (string) so the existing Firestore-keyed `ScopeGuard` membership/ownership model still works.

Use `@CurrentApiKey()` (`packages/api/src/auth/decorators/current-api-key.decorator.ts`) to access the context in controllers.

## 6. Pepper (server-side hash secret)

A pepper is a per-deployment secret concatenated into the argon2id input before hashing. Even a full DB dump can't be brute-forced offline without it.

- **File path:** mounted via CSI driver at `/var/secrets/api-key-pepper` (`API_KEY_PEPPER_PATH`, default). Optional second mount at `/var/secrets/api-key-pepper-next` for rotation (`API_KEY_PEPPER_PREVIOUS_PATH`).
- **Provisioning (out of band):** GCP Secret Manager secret `api-key-pepper-v<N>` with payload `{"version":N,"value":"<base64-32-bytes>"}`. Generate with `openssl rand -base64 32`.
- **Mount (iam-service ONLY):** `infra/k8s/base/iam-service/secretproviderclass.yaml` (and per-env overlay). The other 10 services have no pepper, no argon2, no Prisma.
- **Reader:** `PepperProvider` (`apps/iam-service/src/api-keys/crypto/pepper.provider.ts`) parses the file at request time (cheap, OS-cached) and serves `current()` or `forVersion(n)`.
- **Rotation flow** (dual-pepper, see `packages/docs/Operations/api-key-rotation-runbook.md` §1):
  1. Create `api-key-pepper-v(N+1)`, add as second SPC mount, set `API_KEY_PEPPER_PATH=…-next` and `API_KEY_PEPPER_PREVIOUS_PATH=…-v<N>`, roll iam-service.
  2. New issuances hash with v(N+1); existing keys keep validating via their recorded `pepper_version`.
  3. Argon2 cannot re-hash without the raw secret (only the merchant has it). For a hard cutover on suspected pepper compromise, mass-revoke + force re-issue per the runbook.
  4. After grace period: drop old mount, unset `PREVIOUS_PATH`, roll iam-service, `gcloud secrets delete api-key-pepper-v<N>`.

Argon2 params: `memoryCost=65_536, timeCost=3, parallelism=4, Argon2id` (OWASP 2024+ recommendation). Verify cost ~50ms — kept off the hot path by the Redis cache.

## 7. Issuance API (iam-service `/api-keys`)

Controller at `apps/iam-service/src/api-keys/api-keys.controller.ts`. All endpoints require user-auth via `TokenGuard` + `@RequirePermission('api_keys', '<C|R|U|D>')` and `?ownerId=<firestore-owner-id>` query.

| Method | Path                       | Required permission | Notes                                                                                 |
| ------ | -------------------------- | ------------------- | ------------------------------------------------------------------------------------- |
| POST   | `/api-keys`                | `api_keys:C`        | Returns raw secret ONCE. **NO `@Idempotent`** (caching the body re-serves the secret) |
| GET    | `/api-keys`                | `api_keys:R`        | List for caller's owner; redacted (no `hashedSecret`, no `pepperVersion`)             |
| GET    | `/api-keys/:id`            | `api_keys:R`        | Single key metadata (redacted). 404 (not 403) on cross-owner reads (info disclosure)  |
| POST   | `/api-keys/:id/rotate`     | `api_keys:U`        | Successor + predecessor → `ROTATING` with `expires_at = now + 7d`. **NO `@Idempotent`** |
| DELETE | `/api-keys/:id`            | `api_keys:D`        | Terminal. 410 Gone if already revoked                                                  |

**Why no `@Idempotent` on POST / rotate:** `IdempotencyInterceptor` caches the full response body in Redis. Replaying with the same `Idempotency-Key` would re-serve the raw secret for the entire cache TTL. Issuance is non-idempotent by design — retries call POST again (each call creates a distinct key).

**Scope escalation guard** (`assertScopesNotEscalating`): you can only grant a scope to an API key if your own user profile already holds that `(resource, permission)`. Escalation → `403 scope_escalation:<resource>:<permission>`.

**Business stub during Firestore cutover:** issuance calls `ensureBusinessForOwner(firestoreOwnerId)` — reuses an existing `business_id` from a prior key, else upserts a stub `Business` row keyed by `firestoreOwnerId` with `owner_id = NULL` (migration `20260505160000` made `Business.owner_id` nullable for exactly this case). Real `User.id` backfilled later when the User migration completes.

**Rotation atomicity:** wrapped in `prisma.$transaction`: mark predecessor `ROTATING` with `expires_at = now + 7d`, create successor in the same tx. The 7-day expiry **is** the cutover trigger (rotated keys self-expire).

## 8. Portal admin UI

Route: `/settings/api-keys` (under `apps/eqp-portal/app/(dashboard)/settings/api-keys/`). Feature components under `apps/eqp-portal/features/api-keys/`.

- List page: table of keys (prefix, environment, scopes summary, status, created_at, last_used_at). "Issue new key" CTA.
- `/settings/api-keys/new`: issuance form → server action calls `POST /api-keys` → one-time secret reveal modal with copy button + "I've saved this — I won't see it again" confirmation. Secret stays in React state, never in URL.
- Per-row actions: Rotate (reopens the reveal modal for the successor), Revoke (confirmation dialog).

Permission grants in `packages/common/src/config/user-profiles.ts`: `api_keys: CRUD` on `SUPER_ADMIN`, `WORKSPACE_ADMIN`, `RETAILER_ADMIN`; `api_keys: NONE` on all other roles.

## 9. Audit pipeline (Pub/Sub → BigQuery)

`infra/terraform/modules/api-key-audit/`:

- Pub/Sub topic `eqp-api-key-events` (configurable via `var.topic_name`).
- BigQuery dataset `eqp_audit` + daily-partitioned table `api_key_events` (`time_partitioning.field = occurred_at`, `type = "DAY"`).
- Pub/Sub→BigQuery subscription `<topic>-to-bq` with `bigquery_config.use_table_schema = true`, `write_metadata = false`, 7-day message retention.
- IAM: Pub/Sub service agent gets `roles/bigquery.dataEditor` at the dataset level (least-privilege, not project-level); iam-service GSA gets `roles/pubsub.publisher` on the topic.

`ApiKeyAuditService` in iam-service publishes CloudEvents — fire-and-forget, logs failures but **never throws** (audit must not break the request path).

Event types: `issued`, `rotated`, `revoked`, `validated_ok`, `validated_fail`, `expired`, `revoked_validation_attempt`, `rate_limited`. Each row carries `event_type`, `key_id`, `org_id?`, `environment?`, `request_id?`, `source_ip?`, `occurred_at`, `detail?` (JSON-serialised opaque per-event-type fields, e.g. `predecessor_key_id` on `rotated`).

**The Postgres `ApiKeyAuditEvent` model was dropped** (migration `20260505140000_drop_api_key_audit_events`). The early PR-D draft had a relational audit table; the final design routes all audit through Pub/Sub → BigQuery.

Sample queries (full set in the runbook):

```sql
-- Revoked-key replay attempts
SELECT key_id, COUNT(*) FROM `<project>.eqp_audit.api_key_events`
WHERE event_type = 'revoked_validation_attempt' AND occurred_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
GROUP BY key_id ORDER BY 2 DESC;

-- Brute-force from a source IP
SELECT source_ip, COUNT(*) FROM `<project>.eqp_audit.api_key_events`
WHERE event_type = 'validated_fail' AND occurred_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY source_ip ORDER BY 2 DESC;
```

## 10. Rate limiting (per-org, sliding window, Redis)

`ApiKeyRateLimitService` (`packages/api/src/auth/api-key/rate-limit.service.ts`): two-bucket weighted sliding window, **per org** (not per key).

- Default `API_KEY_RATE_LIMIT_DEFAULT_RPM = 600` and `API_KEY_RATE_LIMIT_WINDOW_SECONDS = 60` (constants in `packages/common/src/constants/cache.constants.ts`). Per-key override via `ApiKey.rate_limit_rpm`.
- Redis key `rate:apikey:<orgId>:<floor(epochSeconds / windowSec)>`. Atomic `INCR` + idempotent `EXPIRE` (TTL = 2× window so the previous bucket stays readable).
- Effective count = `floor(previousCount × weight) + currentCount` where `weight = 1 − elapsed/windowMs`. Burst cannot exceed 2× the limit at the boundary.
- **Fail-OPEN on Redis unavailable** (rate-limit is anti-abuse, not auth; denying legit traffic on cache failure is worse than the abuse leakage; caller-side circuit breaker handles).
- Check runs AFTER auth succeeds (never charge a request against the rate limit before knowing the key is valid).
- Throttled response: HTTP `429 Too Many Requests` with body `{ error: 'rate_limited', retry_after_seconds, count }`; client should honour `Retry-After` with exponential back-off + jitter.

## 11. Operator pre-deploy actions

Authoritative form: `scripts/operator/per-org-api-keys-deploy.sh`; prose form in `packages/docs/Operations/api-key-rotation-runbook.md` §8.

a. **Validate the hand-written migration** with `prisma migrate diff` (the 3 migrations: `20260505120000_add_api_key_model`, `20260505140000_drop_api_key_audit_events`, `20260505160000_allow_business_owner_id_null`). Apply via Cloud Build private worker pool (the only path that can reach the private-IP Cloud SQL instance). Don't `set -x` (leaks `PGPASSWORD`). Run `prisma migrate resolve --applied <name>` after each so subsequent `prisma migrate deploy` doesn't re-apply.

b. **Provision `api-key-pepper-v1`** GCP Secret Manager secret (`--replication-policy=automatic`), add version 1 as JSON `{"version":1,"value":"<openssl rand -base64 32>"}` via temp file (not shell history). Grant `roles/secretmanager.secretAccessor` on the secret to the iam-service Workload Identity GSA. Add the SPC mount to `infra/k8s/base/iam-service/secretproviderclass.yaml` (path `api-key-pepper`).

c. **TF apply BEFORE rolling new portal image.** Order is critical:
   1. `module.managed_kafka` topic + ACL additions (Cohort 2, additive).
   2. `module.api_key_audit` (Pub/Sub topic + BigQuery dataset/table/sink + IAM). Must be live before migration 2 drops the Postgres audit table.
   3. Prisma migrations (Cloud Build).
   4. `module.asm_platform` destructive apply — destroys Lua EnvoyFilter + `kubernetes_secret.edge_api_key` + gateway `EDGE_API_KEY` env. Expect `Plan: 0 to add, 1 to change, 2 to destroy`. Abort if unexpected destroys (namespaces, `RequestAuthentication`, `PeerAuthentication`) appear — TF state is wrong.
   5. Tag a release (release-please owns the tag — never `git tag v*` manually) → 12 image builds → Cloud Deploy rolls all 11 services + portal with the new `ApiKeyGuard` wired in.

d. **Optional post-deploy cleanup:** `gcloud secrets delete gateway-api-key --project=<project> --quiet` (orphaned after enforcement ships).

If portal rolls **before** TF apply, the gateway still checks the Lua filter and new portal pods 401 because they no longer send the shared secret. If TF applies **before** services roll, the gateway passes API-key requests that services cannot yet validate.

## 12. Decommissioned (PR #147 / Plan-5 leftovers removed)

- Lua `EnvoyFilter/api-key-check` on the IngressGateway (`infra/terraform/modules/asm-platform/main.tf` — resource `kubernetes_manifest.envoy_filter_apikey`).
- `kubernetes_secret.edge_api_key` (same module).
- IngressGateway pod's `EDGE_API_KEY` env injection (in `kubernetes_manifest.ingressgateway_deployment`).
- TF root `var.edge_api_key` + `terraform.tfvars.example` line.
- Portal `API_GATEWAY_KEY` env block in `infra/k8s/base/eqp-portal/deployment.yaml`.
- Portal `secretproviderclass.yaml` (the whole file — `gateway-api-key` was its only secret).
- Portal `lib/api-data.ts` `'X-API-Key': ctx.apiKey` header + `apiKey` field on `FetchContext`.
- Portal `lib/safe-action.ts` `API_GATEWAY_KEY` env requirement.
- `scripts/local-proxy.ts` `headers.delete('x-api-key')` line.
- `turbo.json` `API_GATEWAY_KEY` entries in BOTH `dev.env` and `build.env` cache-key arrays.
- OpenAPI `APIKeyHeader` and `APIKeyQueryParam` security schemes (each path's `security:` block keeps `firebase: []` only).

The CIPS legacy `key=<...>` query parameter on `/v1c86ea3/*` routes is a different external system (CIPS API) and remains untouched; `cipsApiKey` continues to flow through.

## 13. PCI win

- Backend credential is **out of the portal pod's env / volume mounts** — Firebase JWT is per-user, not a credential the web tier holds, so the portal exits the PCI-credential boundary cleanly.
- Audit is **BigQuery-native** with daily partitioning, partition pruning on time queries, and SQL access for incident timelines (vs Postgres `ApiKeyAuditEvent` which would have grown unboundedly with hot-table cost).
- Rotation is **self-serve via portal** with the 7-day overlap window — no operator coordination.
- Revocation is **DB-immediate** with at most 5 minutes of cached validity until the Redis entry expires (per-key flush in the runbook for confirmed compromise).
- Pepper is mounted only on iam-service (one place to rotate, one place to compromise-audit).

## 14. Failure modes (each one has a defined error code)

| Failure                                                | HTTP | Body `error` code               | Notes                                                                                           |
| ------------------------------------------------------ | ---- | ------------------------------- | ----------------------------------------------------------------------------------------------- |
| `X-API-Key` + `Authorization: Bearer` both present     | 400  | `auth_000011` (`MIXED_AUTH`)    | Request-shape error, **not** auth failure. Don't try to merge contexts                          |
| Unknown / wrong secret                                 | 401  | `invalid_key`                   | Negative cache 30s. Not 403 — caller doesn't even hold a valid principal yet                    |
| Revoked key (`status = REVOKED`)                       | 401  | `key_revoked` (alias `revoked`) | Audit emits `revoked_validation_attempt`                                                        |
| Expired key (past `expires_at`)                        | 401  | `key_expired` (alias `expired`) | Typically a `ROTATING` predecessor whose 7-day overlap elapsed without successor adoption       |
| Scope insufficient for endpoint                        | 403  | `scope_mismatch`                | `PermissionGuard` consulted `req.apiKey.scopes` and found no match                              |
| Per-org rate limit exceeded                            | 429  | `rate_limited`                  | Body includes `retry_after_seconds`, `count`. Honour `Retry-After` with backoff + jitter        |
| iam-service unreachable (gRPC UNAVAILABLE/DEADLINE)    | 503  | `auth_dependency_unavailable`   | Cache NOT poisoned. Request is guaranteed not applied — safe to retry with same Idempotency-Key |
| Rotate when key already `ROTATING`                     | 409  | `key_already_rotating`          | Only one rotation cycle at a time per key                                                       |
| Revoke an already-revoked key                          | 410  | `key_already_revoked` (Gone)    | Revocation is terminal                                                                          |
| Cross-owner read of a key id                           | 404  | `api_key_not_found`             | Returns 404 not 403 to avoid info disclosure about whether a key id exists                      |
