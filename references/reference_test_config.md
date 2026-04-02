---
name: Test Configuration Management
description: How test configs are structured across the monorepo — Vitest presets, package setup, passWithNoTests, CI pitfalls
type: reference
---

## Test Framework: Vitest 3.x (fully migrated from Jest)

All packages and apps use **Vitest**. Jest has been fully removed from all test scripts. `@repo/jest-config` still exists as dead code but has zero consumers.

## Shared Presets in `@repo/vitest-config`

Three presets exported from `packages/vitest-config/src/index.ts`:

- **`baseConfig`** — Minimal: globals + v8 coverage provider
- **`nestConfig`** — NestJS: unplugin-swc for decorator metadata, node environment, `src/**/*.spec.ts`
- **`nextConfig`** — React: jsdom environment, `**/*.spec.ts` + `**/*.test.ts`. **Requires `jsdom` installed**

### Critical: vitest-config tsconfig

`packages/vitest-config/tsconfig.json` uses `moduleResolution: "Bundler"` (NOT `NodeNext`). This is required because:
- The package is consumed as raw TS (no build step)
- Internal imports are extensionless (`./base` not `./base.ts`)
- `Bundler` resolution allows extensionless imports
- `NodeNext` would require `.ts` extensions which breaks consuming packages' type-checks (TS5097)
- `noEmit: true` is set since there's no build step

If this tsconfig is changed to extend `@repo/typescript-config/base.json` (which uses `NodeNext`), ALL packages that use vitest-config presets will fail type-checking in CI.

## Per-Package Setup

### Packages WITH tests (use shared presets)

```typescript
// vitest.config.ts
import { nestConfig } from '@repo/vitest-config';
import { mergeConfig } from 'vitest/config';
export default mergeConfig(nestConfig, {});
```

DevDependencies: `@repo/vitest-config`, `vitest`
Scripts: `"test": "vitest run"`, `"test:coverage": "vitest run --coverage"`
Examples: `@repo/api`, `@repo/redis`, `apps/iam-service`, `apps/business-crud-service`

### Packages WITHOUT tests (inline config)

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({
    test: { globals: true, passWithNoTests: true },
});
```

DevDependencies: `vitest` only (NO `@repo/vitest-config` — avoids ESM resolution issues)
Scripts: `"test": "vitest run"` (NO `test:coverage` — nothing to cover)
Examples: `@repo/i18n`, `@repo/theme`, `@repo/firebase-client`, `@repo/common`, `@repo/firebase`, `@repo/resilience`, `@repo/gcp`, `apps/eqp-portal`

### Why inline for no-test packages?

Importing `@repo/vitest-config` from packages with no tests caused `ERR_MODULE_NOT_FOUND` in CI (Node ESM can't resolve extensionless `.ts` imports from raw TS packages). Inline configs avoid this dependency entirely.

## CI Gotchas

1. **`@vitest/coverage-v8` version must match vitest** — v4.x with vitest 3.x causes `BaseCoverageProvider` import error. Pin at root: `"@vitest/coverage-v8": "^3.2.1"`
2. **Prisma generate before check-types** — `bun run --cwd packages/database db:generate` must run before `check-types` in CI, or `@repo/database` fails with TS2305 (missing Prisma model exports)
3. **`passWithNoTests: true`** — Required in vitest config for packages with zero test files, otherwise vitest exits code 1
4. **No `test:coverage` script for empty packages** — Only add `test:coverage` to packages that actually have test files, otherwise CI fails on missing `@vitest/coverage-v8`
