---
name: Coding Standards Locations
description: Where to find coding rules, patterns, and conventions in the EQP Platform monorepo
type: reference
---

Coding standards and patterns are spread across these locations:

- **`packages/docs/`** — Comprehensive architecture and coding standards documentation (main source of truth)
- **`packages/eslint-config/`** — ESLint 9 flat configs: `base.ts`, `nest-js.ts`, `next-js.ts`, `react-internal.ts` — enforces naming, function design, TypeScript rules, import ordering, security
- **`packages/typescript-config/`** — Shared tsconfig files (`base.json`, `nestjs.json`, `nextjs.json`)
- **`packages/vitest-config/`** — Vitest presets for testing (nestConfig, nestE2eConfig, nextConfig). Uses unplugin-swc for NestJS decorator metadata + ssr.noExternal for Bun workspace compat
- **Root configs** — `.prettierrc.mjs` (singleQuote), `commitlint.config.ts` (conventional commits), `.husky/` (pre-commit hooks)
- **`CLAUDE.md`** — High-level architecture, commands, tech stack, conventions summary

**Key enforcement**: Three-layer pipeline — ESLint (zero-warnings policy) → Prettier → Husky pre-commit hooks. No `--no-verify` bypasses permitted.
