## 1. Project Scaffolding & Dependencies

- [ ] 1.1 Initialize a Next.js App Router project (`package.json`, `next.config`, `tsconfig.json`, `app/` directory) and verify the dev server starts
- [ ] 1.2 Install and configure Tailwind CSS + PostCSS (`tailwind.config`, `postcss.config`, global stylesheet import) and verify Tailwind classes apply
- [ ] 1.3 Install Prisma and the PostgreSQL driver, create `prisma/` directory, and add a minimal `schema.prisma` with the datasource + generator blocks
- [ ] 1.4 Create `.env.example` listing `DATABASE_URL`, `ENCRYPTION_KEY`, `FX_API_BASE_URL`, `EMAIL_PROVIDER_API_KEY`, `NEXT_PUBLIC_APP_URL` with placeholder values
- [ ] 1.5 Add a startup guard that errors clearly when a required environment variable is missing (no opaque connection errors)
- [ ] 1.6 Create the directory skeleton: `lib/`, `components/ui/`, `i18n/locales/es/`, `stories/` (or `.storybook/`)

## 2. Master Database Schema

- [ ] 2.1 Add the `User` model (`id`, `name`, `email` unique, `passwordHash` nullable, `avatarUrl` nullable, `createdAt`, `updatedAt`) to the Prisma schema
- [ ] 2.2 Add the `Household` model (`id`, `name`, `mainCurrency`, `createdByUserId` FK, `createdAt`, `updatedAt`)
- [ ] 2.3 Add the `HouseholdMember` model (`id`, `householdId` FK, `userId` FK, `role` enum ADMIN|MEMBER, `status` enum ACTIVE|REMOVED, `joinedAt`, `removedAt` nullable, unique `[householdId, userId]`)
- [ ] 2.4 Add the `BankAccount` model (`id`, `householdMemberId` FK, `bankName`, `accountType` enum CHECKING|SAVINGS|CREDIT, `encryptedCredentials` text, `lastSyncAt` nullable, `lastSyncStatus` enum SUCCESS|FAILED|PENDING nullable, `disabledAt` nullable timestamp, `createdAt`, `updatedAt`)
- [ ] 2.5 Add the `Transaction` model with `amount` and `amountInMainCurrency` as `Decimal`/`@db.Decimal`, `currency`, `transactionDate`, `description`, `merchantName` nullable, `categoryId` nullable FK, `billingMonth` string, `syncBatchId` nullable, `providerTransactionId` nullable string (idempotency key), `householdId` FK, `bankAccountId` FK, `createdAt`
- [ ] 2.6 Add the `Transaction` indexes: `@@index([householdId, billingMonth])`, `@@index([householdId, billingMonth, transactionDate])`, `@@index([householdId, transactionDate])`, `@@index([bankAccountId, transactionDate])`, and the unique constraint `@@unique([bankAccountId, providerTransactionId])`
- [ ] 2.7 Add the `Category` model (`id`, `householdId` FK, `name`, `type` enum INCOME|EXPENSE, `isSystem` boolean, `createdAt`, unique `[householdId, name, type]`) and document the predefined expense category list
- [ ] 2.8 Add the `ExchangeRate` model (`id`, `baseCurrency`, `quoteCurrency`, `rate` Decimal, `date`, `source`, `createdAt`, unique `[baseCurrency, quoteCurrency, date]`)
- [ ] 2.9 Add the `Settlement` model (`id`, `householdId` FK, `billingMonth`, `calculatedAt`, `status` enum DRAFT|CONFIRMED, unique `[householdId, billingMonth]`)
- [ ] 2.10 Add the `SettlementTransfer` model (`id`, `settlementId` FK, `fromMemberId` FK, `toMemberId` FK, `amount` Decimal, `createdAt`)
- [ ] 2.11 Add the `Bill` model (`id`, `householdId` FK, `title`, `amount` Decimal, `currency`, `amountInMainCurrency` Decimal, `dueDate`, `billingMonth` string derived from `dueDate` via `getBillingMonth`, `paymentUrl` nullable, `status` enum PENDING|PAID, `paidAt` nullable, `paidByUserId` nullable FK, `linkedTransactionId` nullable FK, `createdByUserId` FK, `createdAt`, `updatedAt`, `@@index([householdId, billingMonth])`)
- [ ] 2.12 Add the `Notification` model (`id`, `householdId` nullable FK, `userId` FK, `type` enum BILL_DUE|SYNC_FAILURE|MEMBER_ADDED|SETTLEMENT_READY, `payload` JSON, `emailStatus` enum PENDING|SENT|FAILED, `retryCount` int default 0, `sentAt` nullable, `createdAt`)
- [ ] 2.13 Add the `SyncJob` model (`id`, `bankAccountId` FK, `status` enum PENDING|RUNNING|SUCCESS|FAILED, `startedAt` nullable, `completedAt` nullable, `errorMessage` nullable, `retryCount` int default 0, `createdAt`)
- [ ] 2.14 Add the `AuditLog` model (`id`, `householdId` nullable FK, `userId` nullable FK, `action`, `entity`, `entityId`, `metadata` JSON, `createdAt`)
- [ ] 2.15 Run `prisma migrate` (or `db push`) against a local PostgreSQL and verify all models, enums, indexes, and unique constraints are created

## 3. Theming

- [ ] 3.1 Configure Tailwind `darkMode: 'class'` and define semantic CSS variables under `:root` and `.dark` (background, surface, primary, danger, text, text-muted)
- [ ] 3.2 Map semantic CSS variables to Tailwind utility classes in `tailwind.config`
- [ ] 3.3 Implement a theme initializer that reads `localStorage` and falls back to `prefers-color-scheme`, applying the `dark` class to `<html>` (no flash on load)
- [ ] 3.4 Implement the `ThemeToggle` component (reflects current theme, keyboard operable, persists to `localStorage`)

## 4. i18n

- [ ] 4.1 Install and configure the i18n library (`next-intl` or equivalent) with Spanish (`es`) as the default locale and an i18n provider wrapping the app
- [ ] 4.2 Create namespaced JSON locale files under `i18n/locales/es/` (`common.json`, `auth.json`, `dashboard.json`) with initial keys
- [ ] 4.3 Implement the `useTranslation` hook exposing `t(key, vars?)` with simple `{{name}}` interpolation only
- [ ] 4.4 Audit the scaffold to confirm no hard-coded user-facing text exists outside locale files

## 5. Reusable UI Component Library

- [ ] 5.1 Implement `Button` (variants primary/secondary/ghost/danger, sizes sm/md/lg, `loading` state with spinner, keyboard activatable)
- [ ] 5.2 Implement `Input`/TextField (label associated, `error`, `helper`, `disabled`)
- [ ] 5.3 Implement `Select`/Dropdown (label, value, options, `disabled`, keyboard navigation)
- [ ] 5.4 Implement `Card` (surface container using semantic `surface` token)
- [ ] 5.5 Implement `Spinner`/LoadingIndicator (accessible loading status)
- [ ] 5.6 Implement `EmptyState` (icon, title, description, optional action)
- [ ] 5.7 Implement `ErrorState` (message, retry action)
- [ ] 5.8 Implement `Badge` (statuses pending/paid/failed/active/removed, distinct semantic colors)
- [ ] 5.9 Implement `Tooltip` (hover + focus show, blur/escape hide) for original-currency detail
- [ ] 5.10 Implement `Modal`/Dialog (backdrop + escape close, focus trap, focus restoration to trigger)
- [ ] 5.11 Implement `Amount` (main currency primary, original currency in tooltip, decimal formatting)
- [ ] 5.12 Implement `DataTable` base (dense rows, sortable columns, cursor-based page prop)
- [ ] 5.12a Implement `DataTable` loading skeleton, `EmptyState`, and `ErrorState` integration
- [ ] 5.13 Implement `useInfiniteScroll` hook (`IntersectionObserver` sentinel, appends next page) and wire it into `DataTable`
- [ ] 5.14 Implement the Navigation/Layout shell (sidebar + header, responsive drawer on narrow viewports, keyboard-focusable nav items)

## 6. Storybook

- [ ] 6.1 Initialize Storybook with the Next.js framework preset and import the global Tailwind stylesheet in preview
- [ ] 6.2 Enable `@storybook/addon-a11y` and a dark-mode addon for canvas theme toggling
- [ ] 6.3 Add at least one co-located story per reusable component (Button, Input, Select, Card, Spinner, EmptyState, ErrorState, Badge, Tooltip, Modal, Amount, DataTable, ThemeToggle, Navigation shell)
- [ ] 6.4 Verify each story renders in both light and dark themes via the dark-mode toggle

## 7. Routing Structure

- [ ] 7.1 Create the `(auth)` route group with a placeholder login/register layout (no app shell)
- [ ] 7.2 Create the `(app)` route group with the navigation shell layout and a placeholder dashboard route
- [ ] 7.3 Create the `(settings)` route group with placeholder profile/household settings routes
- [ ] 7.4 Add a route guard/middleware that redirects unauthenticated users from protected routes to the login route

## 8. Core Utilities

- [ ] 8.1 Implement the encryption utility (`encrypt`/`decrypt`, AES-256-GCM, `ENCRYPTION_KEY` from env, composite IV+tag+ciphertext encoding, server-side only, never logs plaintext/key)
- [ ] 8.2 Add a guard preventing the encryption utility from exposing the key on the client
- [ ] 8.3 Implement the email utility foundation (provider config via `EMAIL_PROVIDER_API_KEY`, `sendEmail({ to, templateName, locale, variables })`, HTML/text templates under `i18n/locales/es/emails/`, `{{var}}` interpolation, explicit error on missing template)
- [ ] 8.4 Implement the FX utility `fetchAndStoreRate(base, quote, date)` (calls configured `FX_API_BASE_URL`, upserts `ExchangeRate` row)
- [ ] 8.5 Implement the FX utility `convertAmount(amount, fromCurrency, toCurrency, date) → Decimal` (uses stored rate, decimal arithmetic, errors on missing rate + fetch failure, short-circuits identical currencies)
- [ ] 8.6 Implement the `getBillingMonth(date) → 'YYYY-MM'` utility (day >= 25 → own month; day <= 24 → previous month; January wrap) and verify the July 26 → "07", July 24 → "06", July 25 → "07", Jan 10 → "12" cases

## 9. API Conventions & Middleware

- [ ] 9.1 Define the `{ data, error }` response envelope helper and a typed response writer used by all routes
- [ ] 9.2 Implement the auth middleware that verifies the session and short-circuits with 401 on failure
- [ ] 9.3 Implement the household-membership middleware that verifies an `ACTIVE` `HouseholdMember` row and short-circuits with 403 on failure (403 for `REMOVED` members)
- [ ] 9.4 Document the API status-code convention (401/403/404/400/500) and the cursor pagination contract (`?cursor=&limit=`, `nextCursor` in envelope)
- [ ] 9.5 Add a sample/placeholder household-scoped route demonstrating the envelope, middleware, and rendering-agnostic (JSON-only) response
- [ ] 9.6 Verify API responses serialize monetary values in a decimal-safe form (e.g. string) to avoid float coercion

## 10. Verification & Validation

- [ ] 10.1 Verify the dev server runs, Tailwind applies, and Prisma connects to PostgreSQL with all models migrated
- [ ] 10.2 Verify theme default (prefers-color-scheme), manual toggle persistence, and no flash on load
- [ ] 10.3 Verify i18n: all user-facing text comes from keys and `{{name}}` interpolation works
- [ ] 10.4 Verify every reusable component renders its empty/loading/error states in Storybook (light + dark)
- [ ] 10.5 Verify the encryption round-trip, tampered-ciphertext rejection, and no plaintext in logs
- [ ] 10.6 Verify FX `fetchAndStoreRate` + `convertAmount` (stored rate, identical-currency short-circuit, error on missing+fetch-fail)
- [ ] 10.7 Verify `getBillingMonth` against the four documented boundary cases
- [ ] 10.8 Verify the API envelope, auth (401), membership (403 for removed/foreign), and decimal-safe money serialization
- [ ] 10.9 Run `openspec validate spec-01-system-design` and resolve any reported issues
