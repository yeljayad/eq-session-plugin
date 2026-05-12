---
name: Never DROP TABLE — soft deprecation only
description: No DROP TABLE / DROP SCHEMA in any migration. Decommissioned tables stay in PG marked `@@ignore` in Prisma; soft delete for rows is via `status` + `archived` columns. Project-wide policy.
type: feedback
---

Never write `DROP TABLE`, `DROP SCHEMA`, or any destructive schema-object removal in a migration. Decommissioned tables stay in the database; rows are soft-deleted via `status` + `archived` columns.

**Why:** User explicitly stated "no drop on our system we have only soft delete is disable + archive" on 2026-05-11 during the audit-service spec review. This applies platform-wide. Dropping tables loses data irretrievably and conflicts with the platform's audit/compliance posture (every entity row is recoverable via `archived = false` + `status = 'active'`; tables themselves should follow the same "preserve" principle).

**How to apply:**

- **Decommissioning a Prisma model:** add `@@ignore` to the model declaration in `schema.prisma`. The table stays in PG; Prisma excludes it from generated client + migrations.
- **Renaming/moving a model to another package:** add the new model in the target package (creates a new table via that package's migration); add `@@ignore` to the original model in the source package. Old table sits dormant; new code uses the new table. No DROP.
- **`prisma migrate dev` after `@@ignore`:** Prisma may auto-generate a DROP migration when it sees a model removed from its managed set. Run `prisma migrate diff` first to inspect. If a DROP appears, hand-edit the migration SQL file to remove the DROP statement before applying.
- **Backfilling data:** when migrating data from a decommissioned table to a new schema, use `INSERT INTO new_schema.new_table SELECT ... FROM public.old_table` — never `TRUNCATE` or `DROP` the source.
- **Rows:** mark `archived = true` or `status = 'inactive' | 'cancelled'` per the `entity_status` enum. Never `DELETE` rows.
- **Storage concern about dormant tables:** acceptable; they grow no further. Only archive to a cold/off-instance store if oncall explicitly flags PG storage cost — and even then, prefer archival over DROP.

**Verification:** in any PR review, grep the migration SQL files for `DROP TABLE`, `DROP SCHEMA`, `TRUNCATE`, `DELETE FROM` (without a WHERE clause). Any match without explicit user authorization for that specific occurrence is a blocker.
