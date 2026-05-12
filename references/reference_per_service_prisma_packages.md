---
name: Per-service PG schema + dedicated Prisma package under packages/
description: Confirmed architecture — one shared eqp database, one shared Postgres user, but each stateful service owns its own PostgreSQL SCHEMA (namespace) AND its own dedicated Prisma PACKAGE under packages/ (e.g. @repo/database-iam, @repo/database-organization). Prisma files never live under apps/. The current monolithic @repo/database is decomposed.
type: reference
---

**Decision (2026-05-11, user-confirmed, corrected):** Each stateful service in the new 11-service catalog owns its own:

1. Dedicated PostgreSQL **schema** (namespace) inside the single shared `eqp` database
2. Dedicated **Prisma package** under `packages/` (NOT under `apps/<svc>/prisma/`). The Prisma schema, migrations, and generated client all live in the package; the consuming NestJS app imports the package as a workspace dep.

**Why:** Hard isolation of data ownership at the schema boundary AND at the package boundary. Each service team can evolve its tables independently. Per-service Prisma client generation. Cross-service DB access is impossible at the import level (organization-service has no way to import the payment-processing Prisma client). Matches the existing monorepo convention where shared/reusable infra lives in `packages/`, not duplicated inside apps. Keeps `apps/<svc>/` focused on NestJS code only.

**How to apply:**

### Database layout (unchanged from prior memo)

- ONE database: `eqp`
- ONE Postgres user: `eqp` superuser (32-char password, IAM-auth follow-up pending per `reference_cloud_sql_prisma.md`)
- N schemas, one per stateful service (snake_case):
    - `iam` — iam-service (5 tables: role, role_permission, user, user_access_scope, permission)
    - `organization` — organization-service (20 tables: owner/workspace/space/retailer/customer/card/bin/card*program/saved_card/virtual_account/source_bank_account/service/approval*_/apple\__/service_definition)
    - `payment_processing` — payment-processing-service (13 tables: account/transaction/transaction*fee/transaction*\*\_detail × 6/ledger_entry/limit_rule/routing_target/routing_attempt)
    - `session` — session-service (1 table: session)
    - `audit` — audit-service (1 table: audit_log) **— in flight in PR #68 (`feat/audit-log` by ttchimou). The `@repo/database-audit` package + `audit` schema are being scaffolded there. Do NOT duplicate.**
    - (orchestration / payment-gateway / risk / notification / callback / config — stateless or Firestore-only, no PG schema)
- Reference data placement (`country`, `currency`, `language`, `method_of_payment`, `service_definition`, `permission`) — pending the contradiction in spec docs. If kept as tables: shared `reference` schema in `public` exposed read-only via a shared package OR moved into the consuming service's schema with seed sync.

### Package layout (per-service Prisma)

```
packages/
├── database-iam/
│   ├── package.json                # name: "@repo/database-iam"
│   ├── prisma/
│   │   ├── schema.prisma           # schemas = ["iam"]
│   │   └── migrations/
│   ├── src/
│   │   └── index.ts                # re-exports PrismaClient + types
│   └── tsconfig.json
├── database-organization/
│   ├── package.json                # name: "@repo/database-organization"
│   ├── prisma/
│   │   ├── schema.prisma           # schemas = ["organization"]
│   │   └── migrations/
│   └── src/index.ts
├── database-payment-processing/
│   ├── package.json                # name: "@repo/database-payment-processing"
│   └── ...
├── database-session/
│   └── ...
└── database-audit/
    └── ...
```

Each NestJS service imports its dedicated package:

```ts
// apps/organization-service/src/database/database.module.ts
import { PrismaClient } from '@repo/database-organization';
```

A service should import **exactly one** `@repo/database-*` package. Importing two = data-ownership violation and a reviewer-flag offense.

### Prisma schema template per package

```prisma
// packages/database-<svc>/prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["<svc_schema>"]      // e.g. ["organization"]
}

generator client {
  provider        = "prisma-client-js"
  output          = "../node_modules/@prisma-clients/<svc>"  // or package-local
  previewFeatures = ["multiSchema"]
}

model Retailer {
  id String @id @default(uuid()) @db.Uuid
  // ...
  @@map("retailer")
  @@schema("organization")
}
```

### Migrations per package

```bash
bun run --cwd packages/database-organization db:migrate:dev    # local
bun run --cwd packages/database-organization db:migrate:deploy # CI/prod
bun run --cwd packages/database-organization db:generate       # regen client
```

CLAUDE.md "Database (Prisma)" commands section needs N variants (one per `@repo/database-*` package) instead of the current single `packages/database` block.

### Cross-schema FKs

44-table schema cross-domain FKs that span service schemas:

- `payment_processing.transaction.customer_id → organization.customer(id)`
- `payment_processing.transaction.retailer_id → organization.retailer(id)`
- `payment_processing.transaction.account_id → payment_processing.account(id)` — same schema
- `organization.card.customer_id → organization.customer(id)` — same schema
- `audit.audit_log.workspace_id → organization.workspace(id)`
- `organization.approval_request.maker_user_id → iam."user"(id)`

**Decision needed:**

1. Keep cross-schema FKs at DB level — `REFERENCES other_schema.table(col)` works in PG, gives strong integrity, but each per-service Prisma package needs to know about foreign tables (Prisma relations across packages are not supported natively — would need `@@ignore` placeholders or raw SQL migrations)
2. Drop cross-schema FKs, snapshot fields + app-layer integrity only — matches "no cross-service DB access" rule, each Prisma package is fully self-contained, no foreign-table awareness needed

**Option 2 is strongly recommended** for per-package Prisma: it keeps each package's `schema.prisma` truly standalone. Snapshot pattern (`transaction.customer_*`, `transaction.retailer_*`) already exists for read-side denormalization. The `customer_id`/`retailer_id` columns remain as plain `String @db.Uuid` with no Prisma relation, no FK constraint. Integrity enforced at the orchestration-service layer (service confirms entity exists before creating the snapshot).

### Cloud Build migrator

Current `infra/cloudbuild-migrate.yaml` runs one Prisma migrate. With per-package Prisma:

- **N independent pipelines** (`migrate-iam.yaml`, `migrate-organization.yaml`, etc.) — recommended; per-package ownership, parallel runs, isolated failures
- One pipeline iterating over packages — simpler but less honest

Each pipeline generates its package's SQL via `prisma migrate diff` and applies it (same private-worker-pool + raw psql pattern as today per `reference_cloud_sql_prisma.md` §5–6).

### Migration of current `@repo/database`

Current state: ONE package `packages/database/` with monolithic `schema.prisma` (User, Business, Transaction, Card, Session, AuditLog, ApiKey).

Decomposition steps:

1. Create the 5 new packages (`database-iam`, `database-organization`, `database-payment-processing`, `database-session`, `database-audit`).
2. Move models from current `schema.prisma` into the matching new package, expand to the 44-table target as work progresses.
3. Migrate consumers (`transaction-service` → `payment-processing-service` import update, etc.) per service rename.
4. Delete `packages/database` once empty.

`@repo/database` survives only if reference-data tables remain centralized — then it shrinks to host just those + shared Prisma helpers/types. Otherwise it goes away entirely.

### `apps/` is for service code only

Going forward:

- Never create `apps/<svc>/prisma/` — that lives in `packages/database-<svc>/`
- Never create a service-local Prisma config — always consume `@repo/database-<svc>`
- Reviewer agents (`eqp-db-reviewer`) should flag any `prisma/` directory appearing under `apps/`

### Reference doc updates required

- `reference_cloud_sql_prisma.md`: schemas listed today (`iam`, `transaction`, `payment_gateway`) predate the new service catalog AND predate this per-package decision. Rewrite to:
    - Schemas: `iam`, `organization`, `payment_processing`, `session`, `audit`
    - Migrator: N pipelines, one per `@repo/database-*` package
    - Prisma layout: package-based, not `packages/database` monolith
- CLAUDE.md "Database (Prisma)" command block: expand to per-package commands
- AGENTS.md `eqp-db-reviewer` watch paths: update from `packages/database/prisma/**` to `packages/database-*/prisma/**`

### Sign-off blockers

- Reference-data placement (tables vs TS constants)
- Cross-schema FK strategy (recommend option 2: drop DB FKs, snapshot + app-layer integrity)
- `@repo/database` deletion vs reference-data residence
- Prisma multi-schema is GA in Prisma 5+; verify the version pinned in repo before applying
