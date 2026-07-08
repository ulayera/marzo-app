## Context

Marzo is a brand-new project — the repository currently contains only the `openspec/` directory and `AGENTS.md`. No application code exists. This spec defines the architectural foundation that eight downstream specs (auth, households, bank sync, transactions, dashboard, settlement, bills, notifications) will build upon without redefining shared infrastructure.

The app is web-first (PWA-ready) with a future React Native client, so the API layer MUST remain REST-like and rendering-agnostic. Users are migrating from spreadsheets, so the UI must be dense and information-rich with generous infinite scroll. The billing cycle is fixed (25th→24th), currency is multi-currency with daily FX snapshots, and bank credentials require AES-256 encryption at rest.

## Goals / Non-Goals

**Goals:**
- Establish a runnable Next.js (App Router) project with Tailwind, Prisma, PostgreSQL, Storybook, i18n, and theming wired up.
- Define the complete master database schema (13 models) so downstream specs never drift.
- Provide a reusable, stateful component library with empty/loading/error states for every interactive component.
- Establish API conventions, a consistent response envelope, and auth/household-scoping middleware patterns.
- Deliver core utilities (encryption, email, FX, billing-cycle) as documented contracts that downstream specs consume.
- Make every requirement testable via WHEN/THEN scenarios.

**Non-Goals:**
- Implement any domain business logic or screen beyond the foundation (auth flows, sync execution, settlement math, notification triggers, etc.).
- Build a real bank aggregator integration — only the provider-ready interface is defined.
- Support OAuth/social login, complex i18n pluralization, or formal WCAG certification.
- Produce React Native code (only keep the API REST-like so it remains possible).

## Decisions

### D1 — Next.js App Router with route groups

**Decision:** Use the App Router with route groups `(auth)`, `(app)`, and `(settings)` to separate layouts without affecting URL paths.

**Rationale:** App Router is the project mandate and supports nested layouts, server components, and colocation. Route groups let auth and app areas have distinct layout shells (e.g. centered card vs. sidebar nav) while keeping clean URLs (`/login`, `/dashboard`, `/settings/profile`).

**Alternatives considered:**
- Pages Router — rejected; project mandate is App Router.
- Single flat layout with conditional rendering — rejected; harder to reason about auth boundaries and leads to layout thrash.

### D2 — Single Prisma schema file as the master schema

**Decision:** All 13 models live in one `prisma/schema.prisma` file defined in this spec. Downstream specs reference model names and fields but MUST NOT redefine or alter shared fields without a superseding spec.

**Rationale:** A single schema file is the single source of truth, preventing schema drift across 8 specs. Prisma natively supports one schema file and generates one client. Splitting into multiple schema files (via `prisma-schema-multi-file` preview feature) adds tooling complexity for no benefit at this scale.

**Alternatives considered:**
- Per-domain schema files — rejected; drift risk and Prisma preview-feature dependency.
- Define schema incrementally per spec — rejected; downstream specs would block on schema changes and foreign-key relationships span domains (e.g. `Transaction.bankAccountId` → `BankAccount`, `BankAccount.householdMemberId` → `HouseholdMember`).

### D3 — PostgreSQL `numeric`/`decimal` for all money columns

**Decision:** Every monetary column (`amount`, `amountInMainCurrency`, `ExchangeRate.rate`, `SettlementTransfer.amount`, `Bill.amount`) uses PostgreSQL `decimal` (Prisma `Decimal` / `@db.Decimal(precision, scale)`), never floating point.

**Rationale:** Floating-point arithmetic causes rounding errors on money. `numeric`/`decimal` preserves exact precision. Prisma maps `Decimal` to a JS `Decimal.js` instance, avoiding float coercion.

**Alternatives considered:**
- Store integer minor units (cents) — rejected; multi-currency with varying decimal precision (e.g. CLP has 0 decimals, JPY has 0, USD has 2) makes a single integer strategy error-prone; `decimal` is clearer and the mandate explicitly says so.
- `float8` / `double precision` — rejected; rounding errors.

### D4 — Theming via Tailwind `dark:` class strategy + CSS variables for semantic tokens

**Decision:** Use Tailwind's `darkMode: 'class'` strategy. Define semantic CSS variables (e.g. `--color-bg`, `--color-surface`, `--color-primary`, `--color-danger`, `--color-text`, `--color-text-muted`) scoped under `:root` and `.dark`. Tailwind config maps these to utility classes (e.g. `bg-background`, `text-primary`). A `ThemeToggle` component toggles the `dark` class on `<html>` and persists the choice to `localStorage`, defaulting to `prefers-color-scheme` when no stored preference exists.

**Rationale:** Class strategy gives explicit control (needed for a manual toggle and SSR consistency). CSS variables allow theme values to change without recompiling Tailwind and keep the semantic token layer stable while raw palette values can evolve. Defaulting to `prefers-color-scheme` satisfies "default to device appearance."

**Alternatives considered:**
- `darkMode: 'media'` — rejected; no manual toggle possible.
- A runtime theme object (e.g. styled-components ThemeProvider) — rejected; adds runtime cost and conflicts with Tailwind utility-first approach; project mandate is Tailwind.

### D5 — i18n with a lightweight library supporting namespaced JSON + simple interpolation

**Decision:** Use a lightweight i18n library (e.g. `next-intl` or equivalent) that supports namespaced JSON message files, locale routing/provider, and simple `{{name}}` interpolation only. Spanish (`es`) is the default and primary locale. Translation files live under `i18n/locales/es/*.json` organized by namespace (`common.json`, `auth.json`, `dashboard.json`, etc.). An i18n provider wraps the app and a `useTranslation` hook exposes `t('namespace.key', { name })`. ALL user-facing strings use keys; no hard-coded user-facing text.

**Rationale:** `next-intl` integrates well with App Router (server + client components), supports message namespaces, and handles simple interpolation without complex pluralization/gender engines (which the project explicitly excludes). Namespacing keeps translation files maintainable as the app grows across 8 specs.

**Alternatives considered:**
- `react-i18next` — viable but heavier; App Router integration requires extra config.
- Custom hand-rolled interpolator — rejected; reinvents locale resolution and provider ergonomics.
- `next-intl` with ICU MessageFormat — rejected; the project mandates no complex pluralization/gender, so ICU is overkill.

### D6 — AES-256-GCM encryption for bank credentials with app-level key from env

**Decision:** Implement an encryption utility using AES-256-GCM (authenticated encryption) with a 256-bit key loaded from the `ENCRYPTION_KEY` environment variable. The utility exposes `encrypt(plaintext) → string` and `decrypt(ciphertext) → string` where the stored string encodes the IV, auth tag, and ciphertext (e.g. base64-encoded composite). The key is NEVER logged, NEVER exposed to the client, and the utility runs server-side only.

**Rationale:** GCM provides both confidentiality and integrity (auth tag), defending against tampering. A single app-level key from env is simple and matches the "app-level key" mandate. Composite encoding keeps the stored value self-contained (no separate IV column needed).

**Alternatives considered:**
- AES-256-CBC without auth tag — rejected; no integrity guarantee.
- Per-record keys with a KMS — rejected; overkill for MVP and no KMS in stack.
- Client-side encryption — rejected; credentials must be usable server-side for sync and the key must never reach the client.

### D7 — Email provider (Resend) with simple i18n HTML/text templates

**Decision:** Use Resend (or equivalent transactional email provider) as the email provider, configured via `EMAIL_PROVIDER_API_KEY`. A `sendEmail({ to, templateName, locale, variables })` function loads an i18n template (HTML + text) by `templateName` and `locale`, interpolates `{{var}}` values, and sends via the provider SDK. Templates live under `i18n/locales/es/emails/*.html` (+ `.txt` companions). Only the infrastructure is built here; notification triggers are wired in Spec 09.

**Rationale:** Resend has a simple SDK, good deliverability, and a free tier suitable for MVP. Keeping templates in the i18n locale folder reuses the same translation discipline and ensures all email text is i18n-keyed. Building only infrastructure here matches the mandate ("triggers wired in later notifications spec").

**Alternatives considered:**
- Nodemailer + SMTP — viable but requires SMTP infra management; Resend is lower operational burden.
- A full template engine (Handlebars/MJML) — rejected for now; simple `{{var}}` interpolation matches the i18n constraint and avoids a second template language. MJML can be introduced later if rich styling is needed.
- Deferring email entirely to Spec 09 — rejected; the mandate explicitly says "Email infra established in THIS foundation spec."

### D8 — FX rates from exchangerate.host with daily snapshots stored in `ExchangeRate`

**Decision:** The FX utility fetches rates from an external API (exchangerate.host or equivalent) using an HTTP client configured via `FX_API_BASE_URL` (and `FX_API_KEY` if required by the chosen provider). The utility exposes `fetchAndStoreRate(base, quote, date)` which calls the API for the given date and upserts a row in `ExchangeRate` (unique on `baseCurrency, quoteCurrency, date`), and `convertAmount(amount, fromCurrency, toCurrency, date) → Decimal` which looks up the stored rate for the date and converts. If no rate is stored for a date, the utility attempts to fetch it; if the fetch fails, conversion returns an error (no silent fallback).

**Rationale:** Storing daily snapshots means historical records keep their original conversion (no retroactive rate drift) and lookups are O(1) on an indexed date. exchangerate.host offers historical rates and a free tier. Returning an explicit error on missing rate prevents silent wrong-currency display.

**Alternatives considered:**
- Live lookup on every render — rejected; slow, rate-limited, and non-deterministic for historical records.
- Store only the latest rate — rejected; historical conversions would shift when rates update.
- Hardcode rates — rejected; multi-currency mandate requires real rates.

### D9 — Infinite scroll via IntersectionObserver hook + cursor-based API pages

**Decision:** Standard list/data-table loading uses an `useInfiniteScroll` hook built on `IntersectionObserver` that triggers loading the next page when a sentinel approaches the viewport. API list endpoints accept a cursor parameter (e.g. `?cursor=<id>&limit=<n>`) and return `{ data: [...], nextCursor: <id|null> }` inside the standard `{ data, error }` envelope. No traditional numbered pagination UI.

**Rationale:** IntersectionObserver is native, performant, and avoids scroll-event throttling. Cursor pagination is stable under inserts (unlike offset pagination) and matches the "generous infinite scroll" mandate. Users migrating from spreadsheets expect continuous lists.

**Alternatives considered:**
- Offset pagination with page numbers — rejected; sparse UI, jumps under new inserts, and contradicts the infinite-scroll mandate.
- `react-virtual`/`react-window` windowing — can be layered on top later for very large lists, but the base pattern is cursor + sentinel; windowing is an optimization, not a foundation requirement.

### D10 — API response envelope `{ data, error }` with consistent HTTP status codes

**Decision:** Every `/api/*` route returns a JSON body `{ data: <T|null>, error: { code: string, message: string, details?: object } | null }`. Successful responses have `error: null` and the appropriate 2xx status. Errors carry a 4xx/5xx status and a populated `error`. Auth failures use 401; forbidden (wrong household) use 403; not found use 404; validation errors use 400; server errors use 500. An auth middleware verifies the session; household-scoped routes run a membership check (householdId from path/query must match a `HouseholdMember` row with `status ACTIVE` for the current user).

**Rationale:** A single envelope makes client-side error handling uniform and the TypeScript response type predictable. Explicit status codes keep the API REST-like for the future React Native client. Membership checks enforce the Admin/Member permission model at the API boundary.

**Alternatives considered:**
- Bare data on success, bare error on failure — rejected; inconsistent shape forces union-type handling on every client call.
- GraphQL — rejected; project mandate is REST-like API routes.
- Per-route auth/membership checks inline — rejected; middleware centralizes the security boundary and avoids forgotten checks.

### D11 — Billing cycle: 25th→24th maps to the calendar month of the 25th

**Decision:** `getBillingMonth(date) → 'YYYY-MM'` returns the calendar month containing the **opening** 25th of the cycle. A transaction on the 25th–31st of month M, or the 1st–24th of month M+1, belongs to billing month M. Specifically:
- Day-of-month >= 25 → billing month = the transaction's own calendar month.
- Day-of-month <= 24 → billing month = the transaction's calendar month minus one month.

Examples (per the task):
- July 26 → day >= 25 → billing month "07" (the July cycle, which runs Jul 25 – Aug 24).
- July 24 → day <= 24 → billing month "06" (the June cycle, which ran Jun 25 – Jul 24).

**Rationale:** This matches the project rule ("25th of a calendar month to the 24th of the next month") and the worked examples exactly. Labeling by the opening month keeps each billing month's label tied to when the cycle starts, which is intuitive for "the July bill."

**Alternatives considered:**
- Label by the closing month (Jul 25–Aug 24 → "08") — rejected; contradicts the worked examples in the task (July 26 → "07").
- Store an explicit window start/end date pair — viable as a denormalized field, but the derived `YYYY-MM` string is the canonical key; the window can always be recomputed from it. Keeping the string as the index keeps `Transaction.billingMonth` simple and sortable.

### D12 — Component library co-located with Storybook stories; state props standardized

**Decision:** Reusable components live under `components/ui/` with co-located `*.stories.tsx` files. Each interactive component accepts standardized state-related concerns: a `loading` boolean, and renders an `EmptyState`/`ErrorState` sub-component (or accepts `emptyState`/`errorState` slots) where relevant. `DataTable` specifically renders skeleton rows while loading, an `EmptyState` when `data.length === 0 && !loading`, and an `ErrorState` when an error is passed.

**Rationale:** Co-location keeps components and stories in sync. Standardizing empty/loading/error rendering at the component level enforces the global rule ("every feature must define empty/loading/error states") at the lowest layer, so downstream specs get it for free.

**Alternatives considered:**
- A separate design-system package — rejected; premature for a single app, adds publish/version overhead.
- Headless UI primitives only (no styled wrappers) — rejected; the mandate is a crisp, consistent component library, not raw primitives.

### D13 — Storybook configuration: Next.js + Tailwind + a11y + dark-mode addons

**Decision:** Configure Storybook with the Next.js framework preset, import the global Tailwind stylesheet in `.storybook/preview`, enable the `@storybook/addon-a11y` and a storybook dark-mode addon for theme toggling in the canvas.

**Rationale:** Next.js preset handles App Router/edge quirks. a11y addon surfaces basic accessibility violations in dev (matches "basic a11y" mandate). Dark-mode addon lets stories be validated in both themes.

**Alternatives considered:**
- Storybook without addons — rejected; loses a11y and theme verification.
- Ladle / Histoire — rejected; Storybook is the project mandate.

## Risks / Trade-offs

- **Risk: Master schema errors propagate to all 8 downstream specs.**
  → Mitigation: The spec defines every model, field, index, and constraint with testable scenarios; validation runs via `openspec validate`; downstream specs are constrained to reference (not redefine) shared models.
- **Risk: Single Prisma file becomes large as specs add fields.**
  → Mitigation: This spec owns the file; downstream specs add only spec-scoped fields via superseding changes and must update this spec's delta if shared fields change. At MVP scale (13 models) a single file is still readable.
- **Risk: AES-256 app-level key in env is less secure than a KMS.**
  → Mitigation: Acceptable for MVP per mandate; key rotation and KMS migration are deferred. Key is server-side only, never logged, never sent to client. Documented as a future hardening item.
- **Risk: FX provider rate limits or downtime could block transaction conversion.**
  → Mitigation: Daily snapshots are cached in `ExchangeRate` (indexed by date), so repeated conversions hit the DB not the API. Conversion surfaces an explicit error rather than silently using a stale/wrong rate. Provider can be swapped via `FX_API_BASE_URL`.
- **Risk: Infinite scroll without windowing can degrade with very large transaction lists.**
  → Mitigation: Cursor pagination bounds DOM growth per page; `react-window` windowing can be layered on later without changing the API contract. Documented as an optimization, not a foundation blocker.
- **Risk: i18n library lock-in.**
  → Mitigation: All access goes through a `useTranslation` hook, so the underlying library can be swapped with one file changed. Namespaced JSON files are portable.
- **Trade-off: Single `ENCRYPTION_KEY` means key rotation requires re-encrypting all credentials.**
  → Accepted for MVP; a versioned-key scheme is a future hardening item, not a foundation requirement.
- **Trade-off: `Decimal` in JS (via Decimal.js) is slower than native numbers.**
  → Accepted; correctness of money arithmetic outweighs sub-millisecond perf differences at MVP scale.
