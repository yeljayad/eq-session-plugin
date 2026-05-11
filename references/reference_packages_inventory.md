---
name: Shared Packages Inventory
description: One-line purpose for each @repo/* package. New packages since the last update: @repo/kafka (domain-event spine), @repo/observability (OTel), @repo/ai (engineering reference, not a code package). @repo/jest-config is gone (Vitest migration complete).
type: reference
---

20 directories under `packages/`. 19 are real packages; `ai` is a README-only reference doc; `jest-config` has been deleted.

## Inventory

| Package                  | Purpose                                                                                                              | Key exports / gotchas                                                                                                       |
|--------------------------|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **@repo/common**         | Zod schemas, DTOs, error codes, constants (CACHE_TTL, RedisKeys), currency/fee/CSV utils, `forbidPii` helper         | Subpath exports: `./constants`, `./errors`, `./schemas`, `./config`, `./client`. Built with **tsdown** (dual ESM/CJS).      |
| **@repo/api**            | NestJS auth guards (ApiKey/Token/User/Permission/Device/RoleScope/Scope), decorators (`@RequirePermission`, `@CurrentUser`, `@CurrentApiKey`, `@Public`, `@Idempotent`), `ZodValidationPipe`, `IdempotencyInterceptor`, `ApiExceptionFilter`, auth repositories, `ApiKeyValidatorClient` | DI-injected classes need VALUE imports (no `type` keyword) — SWC strips type-only imports breaking DI. String tokens: `AUTH_REPOSITORY_TOKENS.USER`, `API_KEY_VALIDATOR`. |
| **@repo/database**       | Prisma client, schema, migrations (PostgreSQL)                                                                       | `db:generate` MUST run before `check-types`. Money columns use `Decimal(18,4)`. snake_case via `@map`/`@@map`.              |
| **@repo/firebase**       | Firestore DAO, Firebase Auth guards/decorators, server-side MFA/TOTP, `FirestoreBaseRepository<T>`                   | Optional peerDep on `@repo/observability` (instruments Firestore calls when present). Firebase Admin uses `applicationDefault()` on GKE (no JSON keys). |
| **@repo/firebase-client**| Browser Firebase Auth SDK wrapper: `AuthService`, `MfaSmsService`, `MfaTotpService`, phone auth, MFA enrollment      | Entry point is `./dist/entry.js` (not `index.js`). Used by the portal `(auth)/` route group.                                |
| **@repo/gcp**            | NestJS modules for Cloud Tasks, Scheduler, Pub/Sub, Storage. `getCloudRunIdToken` (Cloud Run auth), `CloudRunAuthError` | Strips path/query from service URL to derive the audience. Used by orchestration-service smart-routing module.              |
| **@repo/redis**          | NestJS Redis caching module + `IdempotencyService` (check/lock/store/unlock) + `RedisKeys` helpers                   | `RedisKeys.smartRouting*` family powers the smart-routing cache layer. Optional `@repo/observability` peerDep.              |
| **@repo/resilience**     | Circuit breaker (state machine + fallback) + retry (exponential backoff + jitter) NestJS decorators                  | `@UseCircuitBreaker()`, `@UseRetry()`. Optional `@repo/observability` peerDep (emits span events on state transitions).      |
| **@repo/protos**         | `.proto` files only — loaded at runtime via `@grpc/proto-loader`, NOT codegen                                        | 14 protos: health, iam, session, orchestration, transaction, payment-gateway, notification, callback, config, fraud, card, business-crud, common, api-key |
| **@repo/grpc**           | `GrpcClientModule.forServices()` factory, centralized service registry, metadata interceptor for idempotency key    | Peer-deps `@nestjs/microservices`. ProtoLoader is runtime-only. Cross-service idempotency-key propagation is wired here.    |
| **@repo/kafka**          | Producer/consumer base classes, CloudEvents envelope builder, cert-watcher, dead-letter handler, DLQ replay CLI       | See deep dive below. Optional `@repo/observability` peerDep. Pinned to `kafkajs@2.2.4`.                                      |
| **@repo/observability**  | OpenTelemetry SDK initialiser + StructuredLogger + `ObservabilityModule`                                              | See deep dive below. **`import '@repo/observability/instrumentation'` MUST be the FIRST import in `main.ts`.**               |
| **@repo/i18n**           | React i18n context/provider + locale JSON (en/fr/pt/ar) with RTL support                                              | Build copies `src/locales/` into `dist/locales/`. Add new keys via `i18n-completeness-checker` agent.                       |
| **@repo/theme**          | Dark/light theme via next-themes                                                                                      | Peer-deps `next` and `next-themes`. Frontend-only.                                                                          |
| **@repo/ui**             | ShadCN/ui component library (CVA, Tailwind v4) — `static-comp/`, `components/auth/`, `components/portal/`, etc.       | Subpath-only exports (no barrel). Includes `recharts` (used by smart-routing chart).                                        |
| **@repo/eslint-config**  | Centralized ESLint 9 flat configs: `base`, `library`, `next-js`, `nest-js`, `prettier-base`, `react-internal`         | All rule relaxations MUST live here, never inline (inline lint-suppression pragmas are blocked by the pre-commit hook).      |
| **@repo/vitest-config**  | Vitest presets: `nestConfig` (unplugin-swc), `nestE2eConfig`, `nextConfig`. No build step — raw TS.                   | Used as devDependency in every service for SWC-decorator compatibility.                                                     |
| **@repo/typescript-config** | Shared tsconfigs — `base.json`, `nestjs.json`, `nextjs.json`, `react-library.json`                                  | No source code, just JSON.                                                                                                  |
| **@repo/docs**           | Architecture deep-dives, runbooks, ops procedures (no code)                                                           | Reference material. Not bundled. Includes `Operations/gke-cutover-runbook.md`, `Operations/api-key-rotation-runbook.md`.    |
| **@repo/ai**             | README-only reference doc on the team's AI tooling (reviewer agents, skills, hooks, memory, MCP)                      | Has no `package.json` — it is documentation, not a workspace package. Lives at `packages/ai/README.md`.                     |
| ~~@repo/jest-config~~    | **DELETED** — Vitest migration complete; zero consumers remain.                                                       | Removed; don't re-introduce.                                                                                                |

## @repo/observability deep dive

Three things it does:

1. **`./instrumentation` entry point** (side-effect-only module) — calls `initTracing({ serviceName, gcpProjectId, environment, samplingRate })`, registering the OTel SDK before any other module loads. Reads `SERVICE_NAME`, `GCP_PROJECT_ID`, `NODE_ENV`, `OTEL_SAMPLING_RATE` (default `0.05`) from env.
2. **`./` barrel** — `ObservabilityModule` (NestJS module), `StructuredLogger`, `RequestEnrichmentInterceptor`, `@Span` decorator + `withSpan` helper, `getActiveTraceInfo` / `getTraceContextHeaders` for cross-process propagation, `formatCloudLogging` / `formatPretty` log formatters.
3. **Auto-instruments** HTTP (Fastify via `@fastify/otel`), gRPC, NestJS core, Redis, Prisma (`@prisma/instrumentation`). Trace exporter: GCP Cloud Trace (`@google-cloud/opentelemetry-cloud-trace-exporter`) with OTLP-HTTP fallback.

**Why `import '@repo/observability/instrumentation'` MUST be the FIRST import in `main.ts`:** OTel SDK patches Node's `require` / `import` to wrap target modules. If any instrumented module (`fastify`, `@grpc/grpc-js`, `ioredis`, `@prisma/client`) loads before the SDK boots, those modules are NOT patched and emit no spans for the lifetime of the process. The `instrumentation.ts` is a pure side-effect import — no exports — so the import statement itself runs the boot.

Currently wired into `business-service`, `callback-service`, `config-service` (and growing). Other services should follow the same pattern. The optional `@repo/observability` peerDep on `@repo/firebase`, `@repo/redis`, `@repo/resilience`, `@repo/gcp`, `@repo/kafka` means those packages only emit instrumentation when the consuming service has observability installed.

## @repo/kafka deep dive

Domain-event spine, built during Kafka Phase 2. Pinned to `kafkajs@2.2.4`. See `reference_kafka_spine.md` for the full architectural deep dive.

**Exports (barrel `src/index.ts`):**
- `CertWatcherService` — reloads mTLS client certs from disk without restarting the process
- `topics` constants — single source of truth for topic names
- `ConsumerService`, `@KafkaHandler()` decorator, `MessageHandler` interface
- `DeadLetterService` — DLQ writer; CLI replay tool at `bun run replay-dlq` (defined in package.json scripts)
- `EnvelopeBuilder` + `CloudEvent` type — CloudEvents 1.0 spec envelope wrapper
- `KafkaHealth` — health-check probe for `@nestjs/terminus`
- `KafkaConfig` (Zod-validated config loader) + `KafkaModule` (`KafkaModule.forRoot()` / `forRootAsync()`)
- `KAFKA_TOKENS` — DI tokens (`KAFKA_CLIENT`, `KAFKA_PRODUCER`, `KAFKA_SOURCE`)
- `ProducerService` (idempotent producer pre-configured)
- `traceparent` helper — propagates W3C trace context into Kafka headers
- Re-exports `forbidPii` from `@repo/common` for back-compat

**To register in a service AppModule:**
```ts
KafkaModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (cfg: ConfigService) => loadKafkaConfig(cfg),
})
```

mTLS config is loaded from `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CLIENT_CERT_PATH`, `KAFKA_CLIENT_KEY_PATH`. CertWatcher fires on file change (cert-manager rotation).

## @repo/ai (NOT a code package)

`packages/ai/` contains only `README.md` (no `package.json`, no `src/`, no `dist/`). It is the **team-facing engineering reference** for the AI tooling layer: reviewer agents, orchestrator skills, code-quality hooks, memory layers, MCP servers, daily workflow scenarios, onboarding checklist, and rules-of-the-road. Read it during onboarding; it is NOT runtime code and is NOT a workspace dependency.

If you need to add AI-related code (a new agent, skill, hook), the artefacts live in `.claude/{agents,skills,hooks}/` at the repo root — not in `packages/ai/`.

## Versioning rule

- Shared deps are **pinned at root `package.json` only** — sub-packages list them as `peerDependencies` (NOT `dependencies`) to avoid bundling.
- Zod is at `^4.x` root-only; sub-packages declare `"zod": ">=4.0.0"` in peerDeps.
- `@nestjs/*`, `rxjs`, `firebase-admin` follow the same pattern.
- `@repo/observability` is an OPTIONAL peerDep on `@repo/firebase`, `@repo/redis`, `@repo/resilience`, `@repo/gcp`, `@repo/kafka` — services that don't enable tracing get no-ops without breaking.
- Build order is enforced by Turbo (`build` depends on `^build`); manual builds may fail without `bun run build:packages` first.
