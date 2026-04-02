---
name: Frontend Development Blueprint
description: Complete Next.js portal architecture blueprint based on eqp-portal — use when building any new page, component, feature, or auth flow in the frontend
type: reference
---

## Template App: `apps/eqp-portal`

---

## 1. App Router Structure

```
app/
  layout.tsx            — Server Component: HTML shell, font, <Providers>
  page.tsx              — Server Component: redirect('/login')
  globals.css           — TailwindCSS v4 tokens + @theme inline
  providers.tsx         — Client: QueryClient > ThemeProvider > I18nProvider > HtmlAttributes > TooltipProvider
  (auth)/               — Route group (no URL prefix)
    layout.tsx          — Client: auth page wrapper (header, footer, centered main)
    login/page.tsx
    forgot-password/page.tsx
    reset-password/page.tsx
  (dashboard)/          — Route group (no URL prefix)
    layout.tsx          — Client: PortalShell (sidebar + header)
    dashboard/page.tsx  — Server Component
    components/         — Dashboard-specific components
```

**Route groups** `(auth)` and `(dashboard)` apply different layouts without affecting URLs.

## 2. Provider Stack (outside-in)

1. `QueryClientProvider` — TanStack Query v5, staleTime 60s, refetchOnWindowFocus false
2. `ThemeProvider` — next-themes wrapper, attribute="class", defaultTheme="light", enableSystem
3. `I18nProvider` — React Context `{ locale, setLocale, t, dir }`, starts at `'en'`
4. `HtmlAttributes` — useEffect sets `document.documentElement.dir` and `lang`
5. `TooltipProvider` — ShadCN tooltip root

`QueryClient` created inside `useState` (not module scope) to avoid SSR state sharing.

## 3. Component Architecture

### Directory Taxonomy in `@repo/ui`

- `static-comp/` — ShadCN base primitives (button, input, dialog, popover) — checked-in, fully owned
- `components/auth/` — auth layout, card composition
- `components/portal/` — sidebar, shell, header, menu items, user profile
- `components/spaces/` — space/retailer selection dialog, breadcrumb
- `components/notifications/` — panel, card, toast, bell button
- `components/i18n/` — language selector
- `components/common/` — FAB, language popover, mode toggle, dropdown

### App-specific features in `apps/eqp-portal/features/`

- `features/auth/components/` — orchestrator pages
- `features/auth/components/steps/` — one file per login step
- `features/auth/hooks/` — state machine hooks
- `features/auth/types/` — discriminated union types
- `features/auth/utils/` — helpers (step meta, cookie, phone mask)

## 4. Component Patterns

### Props Typing

```typescript
function MyComponent({
    title, description, className, children, ...props
}: React.ComponentProps<'div'> & {
    title: string;
    description?: string;
}): React.ReactElement { ... }
```

- Base: `React.ComponentProps<'element'>` extended with custom props
- Return type: always explicit `React.ReactElement`
- All component roots get `data-slot="component-name"` attribute

### CVA Variants (class-variance-authority)

```typescript
const myVariants = cva('base-classes', {
    variants: {
        variant: { default: '...', outline: '...', ghost: '...' },
        size: { default: '...', sm: '...', lg: '...' },
    },
    defaultVariants: { variant: 'default', size: 'default' },
});
```

### cn Utility

```typescript
import { cn } from '@repo/ui/lib/utils';
// Combines clsx (conditional classes) + twMerge (conflict resolution)
<div className={cn('base-class', isActive && 'active-class', className)} />
```

### Client vs Server Components

- **Server** (default, no directive): `app/layout.tsx`, `app/page.tsx`, static pages
- **Client** (`'use client'`): anything with hooks, event handlers, browser APIs
- All `@repo/ui` components are client components
- All `features/auth/` components are client components

## 5. State Management

**No external state library.** Only React built-ins + TanStack Query.

### Custom Hook Extraction Pattern

Complex pages extract state logic into co-located hooks:

```typescript
function useFeatureForm({ onError, onSuccess }) {
    const [field, setField] = useState('');
    const [loading, setLoading] = useState(false);
    const handleSubmit = async (e: React.FormEvent): Promise<void> => { ... };
    return { field, setField, loading, handleSubmit };
}

function FeatureComponent() {
    const { field, setField, loading, handleSubmit } = useFeatureForm({ onError, onSuccess });
    return <form onSubmit={handleSubmit}>...</form>;
}
```

### Hook Naming: `use<Domain><Action>` (e.g. `useLoginStep`, `useCredentialsForm`)

### Communication: Parent passes callback props `onError`, `onSuccess`, `onNavigate`. Each step component owns its own state.

## 6. Auth Flow (Discriminated Union State Machine)

**Login steps as a typed union:**

```typescript
type LoginStep =
    | { kind: 'credentials' }
    | { kind: 'email-verification'; email: string }
    | { kind: 'force-mfa-select' }
    | { kind: 'mfa-select'; maskedPhone: string; resolver: MultiFactorResolver; ... }
    | { kind: 'sms-otp'; verificationId: string; resolver: MultiFactorResolver; ... }
    | { kind: 'sms-enroll-phone' }
    | { kind: 'sms-enroll'; verificationId: string; phone: string }
    | { kind: 'totp'; resolver: MultiFactorResolver; ... }
    | { kind: 'totp-enroll' }
```

**Step rendering:** `LoginStepBody` switches on `step.kind`, renders the matching step component.

**Navigation:** `useLoginNavigation(setStep)` provides transition functions. `useLoginMfa(setStep)` provides MFA side-effects.

**Firebase services** (module-level singletons, NOT DI):
- `authService` — email/password sign-in, email verification
- `mfaSmsService` — reCAPTCHA management, SMS OTP send/verify, enrollment
- `mfaTotpService` — TOTP secret generation, QR URL, enrollment, sign-in
- `phoneAuthService` — standalone phone number sign-in

**Auth cookie:** Managed via `features/auth/utils/auth-cookie.ts`:
- `setAuthCookie(token)` — sets `auth-token` cookie with `path=/; SameSite=Strict; Secure`
- `clearAuthCookie()` — clears cookie with matching flags + epoch expiry
- All auth flows use these utilities (never set `document.cookie` directly)
- Dashboard layout logout uses `clearAuthCookie()` + `router.push('/login')`

## 7. Form Patterns

**No form library.** Individual `useState` for each field:

```typescript
const [email, setEmail] = useState('');
const [password, setPassword] = useState('');
const [loading, setLoading] = useState(false);
const isFormFilled = email.trim() !== '' && password.trim() !== '';

async function handleSubmit(e: React.FormEvent): Promise<void> {
    e.preventDefault();
    setLoading(true);
    try { ... }
    catch (error) { onError(error instanceof Error ? error.message : 'Unknown error'); }
    finally { setLoading(false); }
}
```

Error propagation via `onError: (msg: string) => void` callback. Loading state disables inputs and changes button text.

## 8. Styling

### TailwindCSS v4 (all-CSS config in `globals.css`)

```css
@import 'tailwindcss';
@import 'tw-animate-css';
@import 'shadcn/tailwind.css';
@source "../../../packages/ui/src";  /* scan UI package */
@variant rtl ([dir="rtl"] &);       /* custom RTL variant */
@theme inline { --color-primary: var(--primary); ... }
```

### Design Token System

Colors as CSS custom properties in `:root` / `.dark`, mapped to Tailwind via `@theme inline`. Key custom tokens beyond ShadCN: `--brand`, `--brand-foreground`, `--brand-dark`, `--brand-tertiary`, `--input-bg`, `--placeholder`, `--indicator-*`, `--sidebar-*`.

### Dark/Light Theme

`ThemeProvider` with `attribute="class"`. `.dark` class on `<html>`. Switch via `setTheme('light' | 'dark')`. `suppressHydrationWarning` on `<html>` prevents flicker.

### RTL Support

1. `HtmlAttributes` sets `dir="rtl"` when locale is `'ar'`
2. `@variant rtl ([dir="rtl"] &)` enables `rtl:` prefix utilities (e.g. `rtl:rotate-180`)
3. Use logical CSS properties everywhere: `ps-` not `pl-`, `start-` not `left-`, `border-s` not `border-l`, `ms-auto` not `ml-auto`
4. Force `dir="ltr"` on phone inputs (numbers are always LTR)

## 9. i18n

- `@repo/i18n` statically imports all locale JSONs at build time
- 4 locales: `en`, `ar`, `fr`, `pt`
- Each locale has `auth.json` and `common.json`
- Access: `const { t } = useI18n(); t.auth.login.title`
- Typed access (no string keys) — TypeScript infers shape from JSON
- Interpolation: `t.auth.text.replace('{{email}}', email)`
- Locale switch: `setLocale('ar')` → `HtmlAttributes` updates `dir` + `lang`

## 10. Navigation & Layout

### Sidebar (`PortalSidebar`)

- Header: logo + collapse button
- Spaces: optional space items with skeleton
- Nav: grouped menu items (labels shown only when expanded)
- Footer: logout + user profile with settings dropdown
- Width: `16rem` expanded / `3.5rem` collapsed, `transition-[width] duration-200`
- Active state: `data-[active=true]:bg-bg-active data-[active=true]:text-brand-tertiary`
- `collapsed` state managed in dashboard layout with `useState(false)`

### Space Selection (`SelectSpaceDialog`)

Two-level drill-down: spaces list → retailers list. Internal state: `view: 'spaces' | 'retailers'`, `activeSpaceId`, `search`. Opened from `WorkspaceBreadcrumb`.

### Notifications

`Popover` containing `NotificationsPanel`. `NotificationsBellButton` with `hasNew` red dot indicator. `NotificationCard` items with type-based styling.

## 11. API Communication (Infrastructure Ready)

- TanStack Query v5 configured in providers
- `@tanstack/react-table` installed for tables
- No live API calls yet — all dashboard data is static demos
- Expected pattern: `useQuery({ queryKey: [...], queryFn: () => fetch(...) })`

## 12. Testing

- Cypress 14.x for E2E (baseUrl `http://localhost:3001`)
- No Jest unit tests in the portal yet
- E2E tests: health check, smoke tests (URL, h1, title)
