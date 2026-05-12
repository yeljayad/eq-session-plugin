---
name: Enum values must match DB values exactly
description: TS/Prisma enum identifiers must equal the exact string saved in the database — no casing transforms, no mapping layers, no display-vs-stored split
type: feedback
---

Enum values in code (TypeScript unions, Prisma enums, NestJS DTOs, Zod schemas) MUST be the exact same string that gets written to PostgreSQL. No transformations, no mappers, no display vs stored split.

**Why:** User explicitly required this. Mismatches between enum identifiers and DB values create silent bugs: filter queries miss rows, snapshots compare unequal, audit logs become unreadable. Keeping them identical removes an entire class of bugs and avoids transformation code on every read/write.

**How to apply:**

- **PG `entity_status` ENUM** = `('active', 'inactive', 'cancelled')` → TS union must be `'active' | 'inactive' | 'cancelled'` (lowercase, exact). NOT `ACTIVE | INACTIVE | CANCELLED`, NOT `Active | Inactive | Cancelled`.
- **CHECK-constraint enums** (account_type, card_type, transaction.status, decision, screening_status, identity_type, gender, channel, payout_type, scope_type, entity_type, limit_type, period, card_sequence.reason, etc.) — TS values match the CHECK string verbatim.
- **Reference natural-key IDs** stored on rows:
    - `currency_code CHAR(3)` → uppercase ISO 4217 ("USD", "EUR", "JPY") — TS constants match exactly
    - `country_code CHAR(2)` → uppercase ISO 3166-1 alpha-2 ("US", "FR") — TS constants match exactly
    - `language.id CHAR(2)` → lowercase ISO 639-1 ("en", "fr") — TS constants match exactly
    - `method_of_payment.name VARCHAR(50)` → exact name string ("card", "crypto", "mpesa", "pix"…) — TS constants match exactly
    - `service_definition.code VARCHAR(10)` → padded codes ("000015", "000048", "000021") — TS constants keep leading zeros
    - `permission.id VARCHAR(100)` → `entity:action` format ("transaction:refund", "card:activate") — TS constants match exactly
- **Prisma enums** (when used instead of CHECK) — `enum EntityStatus { active inactive cancelled }` — lowercase identifiers matching DB. Use Prisma's `@map` only if the identifier is reserved in TS — but prefer making them legal so no mapping is needed.
- **Zod schemas** — `z.enum(['active', 'inactive', 'cancelled'])` — same strings as DB.
- **No display layer in enums.** UI capitalization/translation belongs in the i18n layer (`@repo/i18n`), not in the type system.
- **No "code" ↔ "label" maps in code.** If the user-facing display differs from the stored value, translate at the render boundary, not at the persistence boundary.

**Verification:**

When reviewing a PR that touches any enum / union / Zod schema / Prisma enum / DTO field whose value is persisted, check that every literal value in the TS code is byte-identical to what would land in the DB column. If they differ, flag as a bug and fix the TS side — never add a mapper.
