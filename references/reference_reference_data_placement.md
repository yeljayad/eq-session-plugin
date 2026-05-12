---
name: Reference-data placement decision (D18, 2026-05-11)
description: ISO + MOP + service_definition reference data lives as REAL PG tables in @repo/database shared schema, but per-service packages reference them as plain String columns with NO @relation. App-layer Zod validation against seeded values.
type: reference
---

**Decision (2026-05-11):** Pick the **schema-spec version** (real PG tables) but apply the **service-ownership spec rule** (no cross-package FK constraints). Hybrid that satisfies both.

## What lives as a real PG table (in `@repo/database` shared schema)

| Table                | PK                                                                | Owner                             | Seed source                                 |
| -------------------- | ----------------------------------------------------------------- | --------------------------------- | ------------------------------------------- |
| `country`            | `alpha2_code CHAR(2)` (ISO 3166)                                  | `@repo/database`                  | seed-only — never written at runtime        |
| `currency`           | `alphabetic_code CHAR(3)` (ISO 4217) + `minor_unit`               | `@repo/database`                  | seed-only                                   |
| `language`           | `id CHAR(2)` (ISO 639-1)                                          | `@repo/database`                  | seed-only                                   |
| `method_of_payment`  | `name VARCHAR(50)` + `domestic_solution BOOLEAN`                  | `@repo/database`                  | seed-only (~130 rows; grows via migrations) |
| `service_definition` | `code VARCHAR(10)` + `name`, `category`, `requires_configuration` | `@repo/database`                  | seed-only (50 service codes)                |
| `permission`         | `id VARCHAR(100)` `entity:action` format + `name`, `category`     | `@repo/database-iam` (iam schema) | seed via iam migration                      |

`permission` lives in `iam` schema because RBAC is iam's domain. Everything else lives in `@repo/database` (likely the `public` schema or a new `reference` schema — decide during impl).

## How per-service packages reference them

Per D14 + D15 (no cross-package `@relation`), all references are plain `String` columns:

```prisma
// Inside @repo/database-organization (organization schema)
model Retailer {
  // ...
  country_code  String @db.Char(2)       // logically references country.alpha2_code
  currency_code String @db.Char(3)       // logically references currency.alphabetic_code
  // NO @relation. NO FK. NO @@index on country_code unless query-driven.
  // ...
}
```

App-layer validation enforces correctness:

```ts
// @repo/common Zod schemas reference the seeded enum-like constants
import { z } from 'zod';
import { COUNTRY_CODES } from '@repo/common'; // generated from seed at build time

export const RetailerCreateSchema = z.object({
    country_code: z.enum(COUNTRY_CODES),
    currency_code: z.enum(CURRENCY_CODES),
    // ...
});
```

## Why this resolves the contradiction

| Concern                                                                           | Tables-in-PG resolution                                           | Service-ownership rule resolution                          |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------- |
| Queryability / reporting (`SELECT ... FROM retailer JOIN currency`)               | ✓ tables exist                                                    | —                                                          |
| Future analytics jobs need the metadata (e.g. `minor_unit` for amount formatting) | ✓ tables exist                                                    | —                                                          |
| `audit_log.service_id` is data not a constant — must be queryable                 | ✓ `service_definition` table exists                               | —                                                          |
| Per-service Prisma packages stay decoupled                                        | —                                                                 | ✓ NO `@relation` across schemas; columns are plain strings |
| App layer can validate against the seeded set                                     | ✓ same seeded data exposed as TS constants in `@repo/common`      | —                                                          |
| No DROP TABLE / TRUNCATE ever (D17)                                               | ✓ seed inserts are additive (`INSERT ... ON CONFLICT DO NOTHING`) | —                                                          |

## Seeding strategy

- `@repo/database` ships a seed module (`prisma/seed.ts`) that reads canonical lists from `@repo/common` constants and INSERTs them with `ON CONFLICT DO NOTHING`.
- Same constants exported from `@repo/common` for app-layer Zod validation. Single source of truth: the `@repo/common` constants ARE the canon; PG tables are projections.
- Changes (new currency, new MOP) ship as: (1) edit constants in `@repo/common`, (2) run seed in CI/prod. No migration needed for content changes; only for new columns on the reference tables themselves.

## What lands in PR-1 (next)

A reference-data foundation PR ahead of `@repo/database-organization` (which depends on these tables existing for clean reads):

1. Add the 5 reference tables to `@repo/database` (no @@ignore on them — they're live)
2. Add canonical constants to `@repo/common` (`COUNTRY_CODES`, `CURRENCY_CODES`, `LANGUAGE_CODES`, `METHOD_OF_PAYMENT_NAMES`, `SERVICE_DEFINITION_CODES`)
3. Add `prisma/seed.ts` to populate from the constants
4. Foundational balance-management functions (`fn_update_ledger_balance`, `fn_update_available_balance`) for later trigger attachment by `@repo/database-payment-processing`

`permission` rows + `permission` table itself land with PR #197's `@repo/database-iam` migration (already merged on the branch) OR in a follow-up commit — verify before duplicating.

## Cross-reference

- `project_postgres_schema_spec.md` §1 ("Currency & Country References" — original FK spec)
- `project_service_ownership_spec.md` §4.2 (TS-constants alternative)
- `feedback_no_drop_table.md` (D17)
- `project_per_service_schema_prisma.md` (D14/D15 — no cross-package `@relation`)
