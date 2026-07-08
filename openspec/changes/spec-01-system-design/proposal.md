## Why

Marzo is an EOM bill-management app for households, but no architectural foundation exists yet. Every subsequent spec (auth, households, bank sync, transactions, dashboard, settlement, bills, notifications) depends on a shared database schema, routing shell, theming system, i18n setup, reusable component library, and core utilities. Without a single foundation spec defining these, later specs will diverge and produce schema drift, inconsistent API contracts, and duplicated infrastructure.

## What Changes

- Establish the Next.js (App Router) + React + Tailwind CSS + Prisma + PostgreSQL project scaffold, dependency set, and environment-variable management.
- Define the **complete master database schema** for all domains (User, Household, HouseholdMember, BankAccount, Transaction, Category, ExchangeRate, Settlement, SettlementTransfer, Bill, Notification, SyncJob, AuditLog) including relationships, indexes, and constraints — so later specs reference it without schema drift.
- Define the App Router routing structure: route groups for `(auth)`, `(app)`, `(settings)`, a navigation shell (sidebar/header), and protected-route redirects.
- Establish the theming system: light/dark mode, default to `prefers-color-scheme`, manual toggle persisted in localStorage, Tailwind `dark:` class strategy with CSS variables for semantic colors.
- Establish the i18n setup: Spanish default locale, namespaced JSON key structure, fixed labels + simple `{{name}}` interpolation only, i18n provider/hook, and the rule that ALL user-facing text uses keys.
- Configure Storybook for Next.js + Tailwind with a11y and dark-mode addons.
- Define a reusable UI component library (Button, Input, Select, Card, DataTable, Modal, Badge, Tooltip, Spinner, EmptyState, ErrorState, Amount, ThemeToggle, Navigation/Layout shell) — each with empty, loading, and error states.
- Define state & data-fetching patterns: standard loading/error/empty handling, an infinite-scroll hook, consistent API response envelope `{ data, error }`, and optimistic-update guidance.
- Provide an AES-256 encryption utility for bank credentials with app-level key management (never log plaintext).
- Provide an email-utility foundation (provider setup + simple i18n HTML/text templates + `sendEmail` function) — infrastructure only; triggers wired in the notifications spec.
- Provide an FX-rate utility: interface for fetching daily exchange rates from an external API, storing snapshots in the `ExchangeRate` table, and converting an amount to the household main currency for a given date.
- Define API route conventions: REST-like routes under `/api/`, the `{ data, error }` envelope, auth middleware, household-scoped membership checks, and proper HTTP status codes.
- Provide a billing-cycle utility that computes a billing month (`YYYY-MM`) from a date using the 25th–24th rule.

## Capabilities

### New Capabilities

- `system-design`: The architectural foundation for the Marzo app — project scaffold, master database schema, routing structure, theming system, i18n setup, Storybook configuration, reusable UI component library, state & data-fetching patterns, encryption/email/FX/billing-cycle utilities, and API route conventions. All subsequent specs build upon this capability without redefining shared infrastructure.

### Modified Capabilities

<!-- This is a new project — no existing capabilities are modified. -->

## Non-goals

- Authentication flows, registration screens, and session management UI — covered by Spec 02 (auth-users). This spec only reserves the `(auth)` route group and the `User` schema.
- Household creation, member invitation flows, and onboarding UX — covered by Spec 03 (households-onboarding). This spec only defines the `Household`/`HouseholdMember` schema and role/status enums.
- Bank sync implementation, provider integration, and the mock fixture loader — covered by Spec 04 (bank-accounts-sync). This spec only defines the `BankAccount`/`SyncJob` schema and the encryption utility the sync will use.
- Transaction import logic, categorization UI, and monthly-cycle views — covered by Spec 05 (transactions-monthly-cycle). This spec only defines the `Transaction`/`Category` schema, billing-month derivation utility, and FX conversion utility.
- Dashboard charts, summaries, and analytics — covered by Spec 06 (summary-dashboard). This spec only provides the `DataTable` and `Amount` components the dashboard will consume.
- Settlement calculation and confirmation flows — covered by Spec 07 (balance-settlement). This spec only defines the `Settlement`/`SettlementTransfer` schema.
- Bill checklist UI and payment tracking — covered by Spec 08 (pay-bills-checklist). This spec only defines the `Bill` schema.
- Notification triggers and delivery scheduling — covered by Spec 09 (notifications). This spec only defines the `Notification` schema and the email-sending infrastructure (provider + templates + `sendEmail`).
- React Native mobile app — architecture must allow it, but no mobile code is produced.
- OAuth/social login — schema allows nullable `passwordHash`, but no OAuth provider integration in this spec.
- Complex pluralization/gender i18n rules — explicitly out of scope; fixed labels + simple interpolation only.
- Formal WCAG conformance certification — basic a11y only.
- Real bank aggregator integration (Belvo/Plaid) — only the provider-ready interface and mock fixtures are referenced; real integration is a later concern.

## Impact

- **New code areas**: project root scaffold (`package.json`, `next.config`, `tailwind.config`, `prisma/`, `app/`, `lib/`, `components/`, `i18n/`, `stories/`, `.env.example`).
- **Database**: a single Prisma schema defining all 13 models is created here; later specs must NOT alter shared model fields without a superseding spec.
- **APIs**: `/api/*` route conventions and the `{ data, error }` response envelope are established; all later API routes follow them.
- **Dependencies**: Next.js, React, Tailwind, Prisma, PostgreSQL driver, i18n library, Storybook + addons, encryption library, email provider SDK, FX HTTP client.
- **Environment variables**: `DATABASE_URL`, `ENCRYPTION_KEY`, `FX_API_BASE_URL`, `FX_API_KEY` (if required), `EMAIL_PROVIDER_API_KEY`, `NEXT_PUBLIC_APP_URL`, theme/i18n config.
- **Risk**: the master schema is the single source of truth; any mistake here propagates to all 8 downstream specs. Mitigated by comprehensive requirements + scenarios and explicit non-goals bounding each downstream spec.
