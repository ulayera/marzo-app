## ADDED Requirements

### Requirement: Project Scaffolding

The project SHALL be a Next.js application using the App Router. The scaffold MUST include `package.json`, `next.config`, `tailwind.config`, `postcss.config`, `tsconfig.json`, a `prisma/` directory, an `app/` directory, a `lib/` directory, a `components/` directory, an `i18n/` directory, and a `.env.example` file listing all required environment variables. The tech stack MUST be Next.js + React + Tailwind CSS + Prisma + PostgreSQL. A `README` or env example MUST document `DATABASE_URL`, `ENCRYPTION_KEY`, `FX_API_BASE_URL`, `EMAIL_PROVIDER_API_KEY`, and `NEXT_PUBLIC_APP_URL`.

#### Scenario: Fresh clone runs install and dev server
- **WHEN** a developer clones the repository and runs the package install command followed by the dev script
- **THEN** the install completes without errors and the Next.js dev server starts and serves the default route

#### Scenario: Missing required environment variable is surfaced
- **WHEN** the application starts without `DATABASE_URL` defined
- **THEN** the application emits a clear startup error naming the missing variable rather than crashing with an opaque connection error

#### Scenario: Environment example lists all keys
- **WHEN** the `.env.example` file is inspected
- **THEN** it lists `DATABASE_URL`, `ENCRYPTION_KEY`, `FX_API_BASE_URL`, `EMAIL_PROVIDER_API_KEY`, and `NEXT_PUBLIC_APP_URL` with placeholder values and no real secrets

### Requirement: Master Database Schema — User

The database MUST define a `User` model with fields: `id` (primary key), `name`, `email` (unique), `passwordHash` (nullable to allow future OAuth), `avatarUrl` (nullable), `createdAt`, `updatedAt`. The `email` field MUST be unique. The `passwordHash` field MUST be nullable.

#### Scenario: User model fields
- **WHEN** the Prisma schema is inspected
- **THEN** the `User` model defines `id`, `name`, `email`, `passwordHash`, `avatarUrl`, `createdAt`, `updatedAt` with `email` unique and `passwordHash` nullable

#### Scenario: Duplicate email rejected
- **WHEN** two users are inserted with the same email
- **THEN** the database rejects the second insert with a unique-constraint violation

### Requirement: Master Database Schema — Household

The database MUST define a `Household` model with fields: `id`, `name`, `mainCurrency` (ISO 4217 code, e.g. `CLP`, `USD`), `createdByUserId` (foreign key to `User`), `createdAt`, `updatedAt`. Each household has exactly one `mainCurrency`.

#### Scenario: Household model fields and relation
- **WHEN** the schema is inspected
- **THEN** `Household` defines `id`, `name`, `mainCurrency`, `createdByUserId`, `createdAt`, `updatedAt` and `createdByUserId` is a foreign key to `User.id`

#### Scenario: Main currency is a single ISO 4217 code
- **WHEN** a household is created
- **THEN** it stores exactly one `mainCurrency` value as an ISO 4217 code

### Requirement: Master Database Schema — HouseholdMember

The database MUST define a `HouseholdMember` model with fields: `id`, `householdId` (FK to `Household`), `userId` (FK to `User`), `role` (enum `ADMIN` | `MEMBER`), `status` (enum `ACTIVE` | `REMOVED`), `joinedAt`, `removedAt` (nullable). The combination of `(householdId, userId)` MUST be unique. A `REMOVED` member retains their data but has access revoked.

#### Scenario: Unique membership per household
- **WHEN** a second `HouseholdMember` row is inserted for the same `(householdId, userId)` pair
- **THEN** the database rejects it with a unique-constraint violation

#### Scenario: Removed member keeps row
- **WHEN** a member is removed
- **THEN** their `status` becomes `REMOVED`, `removedAt` is set, and the row is retained (not deleted)

### Requirement: Master Database Schema — BankAccount

The database MUST define a `BankAccount` model with fields: `id`, `householdMemberId` (FK to `HouseholdMember`), `bankName`, `accountType` (enum `CHECKING` | `SAVINGS` | `CREDIT`), `encryptedCredentials` (text — AES-256 encrypted, never plaintext), `lastSyncAt` (nullable), `lastSyncStatus` (nullable, enum `SUCCESS` | `FAILED` | `PENDING`), `disabledAt` (nullable timestamp — set when the account is disconnected; null while active), `createdAt`, `updatedAt`. The `encryptedCredentials` field MUST store only ciphertext.

#### Scenario: BankAccount model fields
- **WHEN** the schema is inspected
- **THEN** `BankAccount` defines all listed fields with `accountType` as an enum of `CHECKING`, `SAVINGS`, `CREDIT`, `lastSyncStatus` as `SUCCESS`, `FAILED`, `PENDING`, and `disabledAt` as a nullable timestamp

#### Scenario: Credentials stored encrypted only
- **WHEN** a bank account is persisted
- **THEN** the `encryptedCredentials` column contains ciphertext and never contains plaintext credentials

### Requirement: Master Database Schema — Transaction

The database MUST define a `Transaction` model with fields: `id`, `bankAccountId` (FK to `BankAccount`), `householdId` (FK to `Household`), `amount` (decimal/numeric, signed — positive = income, negative = expense), `currency` (ISO 4217), `amountInMainCurrency` (decimal/numeric — converted at the record date), `transactionDate` (date), `description`, `merchantName` (nullable), `categoryId` (nullable, FK to `Category`), `billingMonth` (string `YYYY-MM` derived from the 25th–24th cycle), `syncBatchId` (nullable), `providerTransactionId` (nullable string — idempotency key from the sync provider; null for non-synced rows, though MVP has no manual transactions), `createdAt`. A unique constraint MUST be defined on `(bankAccountId, providerTransactionId)` to prevent duplicate transactions on re-sync. The `amount` and `amountInMainCurrency` fields MUST use PostgreSQL `numeric`/`decimal` types — NEVER floating point.

#### Scenario: Transaction money columns are decimal
- **WHEN** the schema is inspected
- **THEN** `amount` and `amountInMainCurrency` are declared as `decimal`/`numeric` and not as floating-point types

#### Scenario: Income is positive, expense is negative
- **WHEN** a transaction representing income is stored
- **THEN** its `amount` is positive
- **WHEN** a transaction representing an expense is stored
- **THEN** its `amount` is negative

### Requirement: Transaction Indexes

The database MUST define indexes on `Transaction` for `(householdId, billingMonth)`, `(householdId, billingMonth, transactionDate)`, `(householdId, transactionDate)`, and `(bankAccountId, transactionDate)` to optimize the most common list and filter queries. A unique constraint MUST be defined on `(bankAccountId, providerTransactionId)` to enforce idempotent sync inserts.

#### Scenario: Billing-month query uses index
- **WHEN** transactions are queried by `householdId` and `billingMonth`
- **THEN** the query plan uses the composite index on `(householdId, billingMonth)`

#### Scenario: Date-range query uses index
- **WHEN** transactions are queried by `householdId` and a `transactionDate` range
- **THEN** the query plan uses the composite index on `(householdId, transactionDate)`

#### Scenario: Duplicate provider transaction rejected
- **WHEN** a second `Transaction` with the same `(bankAccountId, providerTransactionId)` is inserted
- **THEN** the database rejects it with a unique-constraint violation

### Requirement: Master Database Schema — Category

The database MUST define a `Category` model with fields: `id`, `householdId` (FK to `Household`), `name`, `type` (enum `INCOME` | `EXPENSE`), `isSystem` (boolean — predefined vs custom), `createdAt`. The combination `(householdId, name, type)` MUST be unique to prevent duplicate categories per household. The predefined expense categories MUST include: ATM withdrawal, direct debit, money transfer, bank fee, utilities, credit card payment, other. System categories MUST be created per household (scoped by `householdId`) with `isSystem = true`.

#### Scenario: Predefined expense categories present
- **WHEN** the predefined category list is inspected
- **THEN** it contains ATM withdrawal, direct debit, money transfer, bank fee, utilities, credit card payment, and other as `EXPENSE` types

#### Scenario: System vs custom categories
- **WHEN** a category is created from the predefined list
- **THEN** `isSystem` is `true`
- **WHEN** a user creates a custom category
- **THEN** `isSystem` is `false`

#### Scenario: Duplicate category per household rejected
- **WHEN** a second category with the same `(householdId, name, type)` is inserted
- **THEN** the database rejects it with a unique-constraint violation

### Requirement: Master Database Schema — ExchangeRate

The database MUST define an `ExchangeRate` model with fields: `id`, `baseCurrency` (ISO 4217), `quoteCurrency` (ISO 4217), `rate` (numeric/decimal), `date` (date — the day this rate applies), `source` (string), `createdAt`. The combination `(baseCurrency, quoteCurrency, date)` MUST be unique. The `rate` field MUST use `numeric`/`decimal`.

#### Scenario: One rate per currency pair per day
- **WHEN** a second rate is inserted for the same `(baseCurrency, quoteCurrency, date)`
- **THEN** the database rejects it with a unique-constraint violation

#### Scenario: Rate is decimal
- **WHEN** the schema is inspected
- **THEN** `rate` is declared as `numeric`/`decimal` and not floating point

### Requirement: Master Database Schema — Settlement

The database MUST define a `Settlement` model with fields: `id`, `householdId` (FK), `billingMonth` (`YYYY-MM`), `calculatedAt`, `status` (enum `DRAFT` | `CONFIRMED`). There MUST be at most one `Settlement` per `(householdId, billingMonth)` (unique constraint).

#### Scenario: One settlement per household per billing month
- **WHEN** a second `Settlement` is inserted for the same `(householdId, billingMonth)`
- **THEN** the database rejects it with a unique-constraint violation

#### Scenario: Settlement status values
- **WHEN** the schema is inspected
- **THEN** `status` is an enum of `DRAFT` and `CONFIRMED`

### Requirement: Master Database Schema — SettlementTransfer

The database MUST define a `SettlementTransfer` model with fields: `id`, `settlementId` (FK to `Settlement`), `fromMemberId` (FK to `HouseholdMember`), `toMemberId` (FK to `HouseholdMember`), `amount` (decimal/numeric, in the household main currency), `createdAt`. It represents a suggested transfer to settle debts.

#### Scenario: Transfer references two distinct members
- **WHEN** a settlement transfer is created
- **THEN** `fromMemberId` and `toMemberId` each reference a `HouseholdMember` and `amount` is decimal

### Requirement: Master Database Schema — Bill

The database MUST define a `Bill` model with fields: `id`, `householdId` (FK), `title`, `amount` (decimal/numeric), `currency` (ISO 4217), `amountInMainCurrency` (decimal/numeric), `dueDate` (date), `billingMonth` (string `YYYY-MM` derived from `dueDate` via the 25th–24th cycle), `paymentUrl` (nullable), `status` (enum `PENDING` | `PAID`), `paidAt` (nullable), `paidByUserId` (nullable, FK to `User`), `linkedTransactionId` (nullable, FK to `Transaction` — links to an expense transaction if the bill appears in card statements), `createdByUserId` (FK to `User`), `createdAt`, `updatedAt`. Money columns MUST be decimal. An index MUST be defined on `Bill(householdId, billingMonth)` for billing-cycle filtering.

#### Scenario: Bill links to a transaction
- **WHEN** a bill appears as an expense in card statements
- **THEN** `linkedTransactionId` references the corresponding `Transaction` and `status` can become `PAID` with `paidAt` set

#### Scenario: Bill billing month derived from due date
- **WHEN** a bill is created with a `dueDate`
- **THEN** its `billingMonth` is computed via `getBillingMonth(dueDate)` and stored

#### Scenario: Bill money columns are decimal
- **WHEN** the schema is inspected
- **THEN** `amount` and `amountInMainCurrency` are decimal/numeric, not floating point

### Requirement: Master Database Schema — Notification

The database MUST define a `Notification` model with fields: `id`, `householdId` (nullable, FK), `userId` (FK to `User`), `type` (enum `BILL_DUE` | `SYNC_FAILURE` | `MEMBER_ADDED` | `SETTLEMENT_READY`), `payload` (JSON), `emailStatus` (enum `PENDING` | `SENT` | `FAILED`), `retryCount` (int, default 0 — tracks email send retries), `sentAt` (nullable), `createdAt`.

#### Scenario: Notification type values
- **WHEN** the schema is inspected
- **THEN** `type` is an enum of `BILL_DUE`, `SYNC_FAILURE`, `MEMBER_ADDED`, `SETTLEMENT_READY`, `emailStatus` is `PENDING`, `SENT`, `FAILED`, and `retryCount` is an int with a default of 0

#### Scenario: Notification carries JSON payload
- **WHEN** a notification is created
- **THEN** its `payload` stores arbitrary JSON metadata (e.g. bill id, billing month), `emailStatus` starts as `PENDING`, and `retryCount` starts at 0

### Requirement: Master Database Schema — SyncJob

The database MUST define a `SyncJob` model with fields: `id`, `bankAccountId` (FK), `status` (enum `PENDING` | `RUNNING` | `SUCCESS` | `FAILED`), `startedAt` (nullable — null while `PENDING`), `completedAt` (nullable), `errorMessage` (nullable), `retryCount` (int, default 0), `createdAt`.

#### Scenario: New sync job defaults
- **WHEN** a sync job is created
- **THEN** `status` is `PENDING`, `retryCount` is 0, `startedAt` is null, and `completedAt` is null

#### Scenario: Failed sync job records error
- **WHEN** a sync job fails
- **THEN** `status` becomes `FAILED`, `completedAt` is set, and `errorMessage` stores the failure reason

### Requirement: Master Database Schema — AuditLog

The database MUST define an `AuditLog` model with fields: `id`, `householdId` (nullable, FK), `userId` (nullable, FK to `User` — null when the action targets a non-existent user, e.g. a failed login with an unregistered email), `action` (string), `entity` (string), `entityId` (string), `metadata` (JSON), `createdAt`.

#### Scenario: Audit log entry captures context
- **WHEN** an auditable action occurs
- **THEN** an `AuditLog` row is created with `userId`, `action`, `entity`, `entityId`, and a JSON `metadata` payload

### Requirement: Routing Structure

The application MUST use App Router route groups: `(auth)` for login/register, `(app)` for the authenticated household area, and `(settings)` for profile and household settings. A navigation shell (sidebar + header) MUST wrap the `(app)` and `(settings)` areas. Protected routes MUST redirect unauthenticated users to the login route. The `(auth)` area MUST NOT render the sidebar/header shell.

#### Scenario: Unauthenticated user redirected
- **WHEN** an unauthenticated user navigates to a protected route under `(app)`
- **THEN** they are redirected to the login route

#### Scenario: Auth area has no app shell
- **WHEN** the login page renders
- **THEN** it does not render the sidebar or header navigation shell

#### Scenario: App area renders navigation shell
- **WHEN** an authenticated user views a route under `(app)`
- **THEN** the sidebar and header navigation shell renders around the page content

### Requirement: Theming System

The application MUST support light and dark themes. The default theme MUST follow the user's `prefers-color-scheme` when no manual preference is stored. A manual toggle MUST persist the chosen theme to `localStorage`. Theming MUST use Tailwind's `darkMode: 'class'` strategy with the `dark` class applied to the root `<html>` element. Semantic colors (background, surface, primary, danger, text, text-muted) MUST be defined as CSS variables scoped under `:root` and `.dark`, and mapped to Tailwind utility classes.

#### Scenario: Defaults to device appearance
- **WHEN** a user with no stored theme preference and a dark OS preference opens the app
- **THEN** the dark theme is applied

#### Scenario: Manual toggle persists
- **WHEN** a user toggles the theme to light and reloads the page
- **THEN** the light theme remains applied regardless of OS preference

#### Scenario: Semantic colors switch by class
- **WHEN** the `dark` class is present on `<html>`
- **THEN** semantic CSS variables resolve to the dark palette and Tailwind utilities render dark-appropriate colors

### Requirement: ThemeToggle Component

The application MUST provide a `ThemeToggle` component that switches between light and dark themes, reflects the current theme, and is keyboard operable.

#### Scenario: Toggle changes theme
- **WHEN** a user activates the `ThemeToggle` while in light mode
- **THEN** the application switches to dark mode and persists the choice

#### Scenario: Toggle is keyboard operable
- **WHEN** a keyboard user focuses and activates the `ThemeToggle`
- **THEN** the theme switches without requiring a mouse

### Requirement: i18n Setup

The application MUST use Spanish (`es`) as the default and primary locale. ALL user-facing text MUST be rendered via i18n keys; no hard-coded user-facing strings. Translation files MUST be namespaced JSON under `i18n/locales/es/` (e.g. `common.json`, `auth.json`, `dashboard.json`). Interpolation MUST support simple `{{name}}` replacement only — NO complex pluralization or gender rules. An i18n provider MUST wrap the application and a `useTranslation` hook (or equivalent) MUST expose a `t(key, vars?)` function.

#### Scenario: User-facing text uses keys
- **WHEN** any user-facing string is rendered
- **THEN** it is produced by a `t('namespace.key')` call and the literal string lives in a JSON locale file

#### Scenario: Simple interpolation
- **WHEN** `t('common.welcome', { name: 'María' })` is called with a message `Bienvenida {{name}}`
- **THEN** the rendered output is `Bienvenida María`

#### Scenario: No hard-coded user-facing text
- **WHEN** the codebase is searched for user-facing literals outside locale files
- **THEN** none are found in rendered output (all go through `t()`)

### Requirement: Storybook Configuration

The project MUST configure Storybook with the Next.js framework preset, the global Tailwind stylesheet imported in preview, the `@storybook/addon-a11y` addon enabled, and a dark-mode addon enabled for theme toggling in the canvas. Every reusable UI component MUST have at least one co-located story.

#### Scenario: Storybook loads with Tailwind
- **WHEN** Storybook is started
- **THEN** the preview renders components with Tailwind styles applied

#### Scenario: a11y addon active
- **WHEN** a story is opened in Storybook
- **THEN** the a11y addon panel is available and surfaces accessibility violations

#### Scenario: Dark mode toggle in canvas
- **WHEN** the Storybook dark-mode toggle is activated
- **THEN** stories render with the dark theme applied

### Requirement: Button Component

The component library MUST include a `Button` component with variants `primary`, `secondary`, `ghost`, `danger`; sizes `sm`, `md`, `lg`; and a `loading` state that shows a spinner and disables interaction. It MUST be keyboard focusable and activatable.

#### Scenario: Variants render distinct styles
- **WHEN** buttons of each variant are rendered
- **THEN** `primary`, `secondary`, `ghost`, and `danger` each render visually distinct styles

#### Scenario: Loading state disables interaction
- **WHEN** a `Button` is in the `loading` state
- **THEN** it displays a loading spinner, is not clickable, and is not keyboard-activatable

#### Scenario: Keyboard activation
- **WHEN** a keyboard user focuses a `Button` and presses the activation key
- **THEN** the button's action is triggered

### Requirement: Input / TextField Component

The component library MUST include an `Input` (TextField) component supporting `label`, `error` message, `helper` text, and a `disabled` state. The label MUST be associated with the input for accessibility.

#### Scenario: Error and helper text render
- **WHEN** an `Input` is rendered with an error and helper text
- **THEN** both the error and helper text are visible and the error is announced to assistive technology

#### Scenario: Disabled state
- **WHEN** an `Input` is disabled
- **THEN** it is not editable and is not focusable for input

### Requirement: Select / Dropdown Component

The component library MUST include a `Select` (Dropdown) component supporting a label, selected value, options, a disabled state, and keyboard navigation of options.

#### Scenario: Select renders options
- **WHEN** a `Select` is opened
- **THEN** its options are listed and selectable

#### Scenario: Keyboard navigation
- **WHEN** a keyboard user focuses the `Select` and navigates options with arrow keys
- **THEN** options can be highlighted and selected without a mouse

### Requirement: Card Component

The component library MUST include a `Card` component as a surface container with consistent padding, border, and background driven by semantic theme tokens.

#### Scenario: Card uses surface token
- **WHEN** a `Card` is rendered in light and dark themes
- **THEN** its background and border use the `surface` semantic token and adapt to the theme

### Requirement: DataTable Component

The component library MUST include a `DataTable` component that is dense, supports sortable columns, infinite scroll via a sentinel and cursor-based pages, a loading skeleton, an empty state, and an error state. When `data` is empty and not loading, it MUST render an `EmptyState`. When an error is provided, it MUST render an `ErrorState`. While loading, it MUST render skeleton rows.

#### Scenario: Empty state
- **WHEN** a `DataTable` receives an empty `data` array and is not loading
- **THEN** it renders an `EmptyState` component

#### Scenario: Loading skeleton
- **WHEN** a `DataTable` is loading
- **THEN** it renders skeleton placeholder rows and does not flash empty content

#### Scenario: Error state
- **WHEN** a `DataTable` receives an error
- **THEN** it renders an `ErrorState` with the error message and a retry action

#### Scenario: Infinite scroll loads next page
- **WHEN** the user scrolls near the bottom sentinel of a `DataTable` with a non-null `nextCursor`
- **THEN** the next page is requested and appended to the existing rows without replacing them

#### Scenario: Column sort
- **WHEN** a user activates a sortable column header
- **THEN** the data is sorted by that column and the sort direction is indicated visually

### Requirement: Modal / Dialog Component

The component library MUST include a `Modal` (Dialog) component that overlays the page, traps focus while open, closes on backdrop click and the escape key, and returns focus to the trigger on close.

#### Scenario: Escape closes modal
- **WHEN** a `Modal` is open and the escape key is pressed
- **THEN** the modal closes

#### Scenario: Focus trapped
- **WHEN** a `Modal` is open and the user tabs through focusable elements
- **THEN** focus cycles within the modal and does not reach the background page

### Requirement: Badge Component

The component library MUST include a `Badge` component for status display (e.g. `pending`, `paid`, `failed`, `active`, `removed`). Each status MUST map to a distinct semantic color.

#### Scenario: Status badges distinct
- **WHEN** badges for `pending`, `paid`, and `failed` are rendered together
- **THEN** each renders a visually distinct color tied to its semantic meaning

### Requirement: Tooltip Component

The component library MUST include a `Tooltip` component that shows on hover and keyboard focus, and hides on blur/escape. It MUST be used to display the original-currency amount alongside a main-currency amount.

#### Scenario: Tooltip on hover and focus
- **WHEN** a user hovers or focuses the tooltip trigger
- **THEN** the tooltip content becomes visible
- **WHEN** focus leaves or escape is pressed
- **THEN** the tooltip hides

### Requirement: Spinner / LoadingIndicator Component

The component library MUST include a `Spinner` (LoadingIndicator) component that renders an accessible loading indicator.

#### Scenario: Spinner announces loading
- **WHEN** a `Spinner` is rendered
- **THEN** it has an accessible loading status and is visible

### Requirement: EmptyState Component

The component library MUST include an `EmptyState` component accepting an icon, title, description, and an optional action (e.g. a button). It MUST be used by list/table components when there is no data.

#### Scenario: EmptyState with action
- **WHEN** an `EmptyState` is rendered with an optional action
- **THEN** the icon, title, description, and action button are all visible and the action is activatable

### Requirement: ErrorState Component

The component library MUST include an `ErrorState` component accepting a message and a retry action. It MUST be used by data-fetching surfaces when a load fails.

#### Scenario: ErrorState retry
- **WHEN** an `ErrorState` is rendered with a retry action and the user activates retry
- **THEN** the retry handler is invoked

### Requirement: Amount Display Component

The component library MUST include an `Amount` component that displays a monetary value in the household main currency as the primary text, with the original currency amount shown in a `Tooltip` on hover/focus. It MUST format using decimal values, never floating point.

#### Scenario: Main currency primary
- **WHEN** an `Amount` is rendered for a USD transaction in a CLP household
- **THEN** the main-currency (CLP) value is shown as the primary text

#### Scenario: Original currency in tooltip
- **WHEN** a user hovers or focuses the `Amount`
- **THEN** the tooltip shows the original-currency (USD) value

### Requirement: Navigation / Layout Shell Component

The component library MUST include a Navigation/Layout shell with a sidebar and header, responsive across viewports, that wraps the authenticated app area.

#### Scenario: Responsive shell
- **WHEN** the shell is rendered on a narrow viewport
- **THEN** the sidebar collapses to a drawer or icon-only form and the header remains visible

#### Scenario: Navigation items render
- **WHEN** the shell is rendered
- **THEN** navigation items are present and keyboard focusable

### Requirement: State and Data-Fetching Patterns

The application MUST standardize on loading, error, and empty states for all data-fetching surfaces. An `useInfiniteScroll` hook MUST trigger loading the next page via `IntersectionObserver` when a sentinel approaches the viewport. API list endpoints MUST accept a cursor parameter and return a `nextCursor` inside the `{ data, error }` envelope. Optimistic updates MAY be applied where a clear rollback is possible.

#### Scenario: Loading state shown during fetch
- **WHEN** a data-fetching surface is awaiting its first response
- **THEN** a loading indicator or skeleton is shown, not empty content

#### Scenario: Infinite scroll hook triggers next page
- **WHEN** the scroll sentinel enters the viewport and `nextCursor` is present
- **THEN** the next page is fetched and appended

#### Scenario: Optimistic update rolls back on error
- **WHEN** an optimistic update is applied and the server returns an error
- **THEN** the client reverts to the prior state and surfaces the error

### Requirement: API Response Envelope

Every `/api/*` route MUST return a JSON body of shape `{ data: <T|null>, error: { code: string, message: string, details?: object } | null }`. Successful responses MUST set `error: null` with an appropriate 2xx status. Error responses MUST populate `error` with a 4xx/5xx status. Auth failures MUST use 401; forbidden access (wrong household) MUST use 403; not found MUST use 404; validation errors MUST use 400; server errors MUST use 500.

#### Scenario: Success shape
- **WHEN** an API route returns successfully
- **THEN** the body is `{ data: <value>, error: null }` with a 2xx status

#### Scenario: Error shape with code
- **WHEN** an API route returns a validation error
- **THEN** the body is `{ data: null, error: { code, message, details? } }` with status 400

#### Scenario: Auth failure status
- **WHEN** an unauthenticated request hits a protected route
- **THEN** the response status is 401 and the body contains an `error` with a code

#### Scenario: Forbidden household access
- **WHEN** an authenticated user requests a resource in a household they are not an active member of
- **THEN** the response status is 403 and the body contains an `error` with a code

### Requirement: API Auth and Membership Middleware

Protected API routes MUST be wrapped by an auth middleware that verifies the session. Household-scoped routes MUST additionally verify that the current user is an `ACTIVE` `HouseholdMember` of the target household before executing the handler.

#### Scenario: Missing session rejected
- **WHEN** a request without a valid session hits a protected route
- **THEN** the middleware short-circuits with a 401 and the handler is not invoked

#### Scenario: Removed member rejected
- **WHEN** a request from a `REMOVED` member targets a household-scoped route
- **THEN** the middleware short-circuits with a 403 and the handler is not invoked

#### Scenario: Active member allowed
- **WHEN** an `ACTIVE` member requests a household-scoped route for their household
- **THEN** the handler is invoked

### Requirement: Encryption Utility

The application MUST provide an AES-256 encryption utility with `encrypt(plaintext) → string` and `decrypt(ciphertext) → string` functions using AES-256-GCM (authenticated encryption). The key MUST be loaded from the `ENCRYPTION_KEY` environment variable and MUST be 256 bits. The utility MUST run server-side only. The stored ciphertext MUST encode the IV and auth tag alongside the ciphertext. The utility MUST NEVER log plaintext or the key.

#### Scenario: Round-trip encrypt/decrypt
- **WHEN** a plaintext value is encrypted and then decrypted
- **THEN** the decrypted output equals the original plaintext

#### Scenario: Tampered ciphertext rejected
- **WHEN** a ciphertext is modified before decryption
- **THEN** decryption fails due to the GCM auth tag and no plaintext is returned

#### Scenario: Plaintext never logged
- **WHEN** the encryption utility is exercised
- **THEN** neither the plaintext input nor the key appears in any log output

#### Scenario: Client-side access prevented
- **WHEN** the utility module is imported in a client component
- **THEN** the import is prevented or the module no-ops rather than exposing the key

### Requirement: Email Utility Foundation

The application MUST provide an email infrastructure with a configured provider (via `EMAIL_PROVIDER_API_KEY`), a `sendEmail({ to, templateName, locale, variables })` function, and simple HTML/text templates stored under the i18n locale folder. Templates MUST use `{{var}}` interpolation. This spec establishes infrastructure only; notification triggers are wired in the notifications spec.

#### Scenario: Send email with interpolated template
- **WHEN** `sendEmail` is called with a `templateName`, `locale`, and `variables`
- **THEN** the corresponding HTML and text templates are loaded, variables are interpolated, and the email is handed to the provider

#### Scenario: Missing template errors clearly
- **WHEN** `sendEmail` is called with a `templateName` that has no template file for the locale
- **THEN** an explicit error is returned and no partial email is sent

#### Scenario: Email text is i18n-keyed
- **WHEN** an email template is inspected
- **THEN** its subject and body text come from locale files (no hard-coded user-facing text)

### Requirement: FX Rate Utility

The application MUST provide an FX utility with `fetchAndStoreRate(base, quote, date)` and `convertAmount(amount, fromCurrency, toCurrency, date) → Decimal`. Rates MUST be fetched from an external API configured via `FX_API_BASE_URL` and stored as snapshots in `ExchangeRate` (unique on `baseCurrency, quoteCurrency, date`). `convertAmount` MUST look up the stored rate for the date and convert using decimal arithmetic. If no rate is available and the fetch fails, `convertAmount` MUST return an explicit error and MUST NOT silently fall back to a wrong rate.

#### Scenario: Fetch and store a daily rate
- **WHEN** `fetchAndStoreRate('USD', 'CLP', '2026-07-06')` is called
- **THEN** the rate is fetched from the configured provider and upserted into `ExchangeRate` for that date

#### Scenario: Convert using stored rate
- **WHEN** `convertAmount` is called for a date with a stored rate
- **THEN** the amount is converted using the stored rate and returned as a `Decimal`

#### Scenario: Missing rate with fetch failure errors
- **WHEN** `convertAmount` is called for a date with no stored rate and the fetch fails
- **THEN** an explicit error is returned and no converted amount is produced

#### Scenario: Identical currencies return the same amount
- **WHEN** `convertAmount(amount, 'USD', 'USD', date)` is called
- **THEN** the original amount is returned unchanged without any API call

### Requirement: Billing Cycle Utility

The application MUST provide a `getBillingMonth(date) → 'YYYY-MM'` utility implementing the 25th–24th rule. A date on day-of-month >= 25 belongs to its own calendar month. A date on day-of-month <= 24 belongs to the previous calendar month. The output MUST be a `YYYY-MM` string.

#### Scenario: Late-month date maps to its own month
- **WHEN** `getBillingMonth` is called with July 26
- **THEN** it returns `"07"`

#### Scenario: Early-month date maps to previous month
- **WHEN** `getBillingMonth` is called with July 24
- **THEN** it returns `"06"`

#### Scenario: Boundary day 25 belongs to the opening month
- **WHEN** `getBillingMonth` is called with July 25
- **THEN** it returns `"07"`

#### Scenario: January early-month wraps to previous year
- **WHEN** `getBillingMonth` is called with January 10
- **THEN** it returns `"12"` of the previous year

### Requirement: API Route Conventions

API routes MUST live under `/api/` and be REST-like. Routes MUST use the `{ data, error }` envelope, proper HTTP methods, and the auth/membership middleware. Household-scoped routes MUST include a household identifier in the path or query and MUST enforce membership. Routes MUST be rendering-agnostic (no React in responses) so a future React Native client can consume them.

#### Scenario: REST-like household route
- **WHEN** a client requests a household-scoped resource via the documented path and method
- **THEN** the route returns the `{ data, error }` envelope with the appropriate status and contains no rendering-specific payload

#### Scenario: Rendering-agnostic responses
- **WHEN** any `/api/` route response body is inspected
- **THEN** it contains only data/error JSON and no React components or markup

### Requirement: Component Empty/Loading/Error States

Every reusable component that renders data or asynchronous results MUST define an empty state, a loading state, and an error state. `DataTable`, list views, and any data-fetching surface MUST use `EmptyState`, a loading skeleton/spinner, and `ErrorState` respectively.

#### Scenario: Each state is reachable
- **WHEN** a data component is rendered with no data, then with a load in progress, then with an error
- **THEN** it renders `EmptyState`, then a loading indicator, then `ErrorState` in sequence

### Requirement: Accessibility Baseline

The component library and routes MUST provide a basic accessibility baseline: keyboard navigability, semantic HTML, and reasonable focus management. Interactive elements MUST be reachable and operable by keyboard. `Modal` MUST trap focus and restore it on close.

#### Scenario: Keyboard-only operation
- **WHEN** a user navigates the app using only the keyboard
- **THEN** all interactive elements are reachable and operable

#### Scenario: Modal focus restoration
- **WHEN** a `Modal` that was opened from a trigger is closed
- **THEN** focus returns to the trigger element

### Requirement: Financial Precision Guarantee

All monetary values — across the database, API layer, and UI — MUST use `numeric`/`decimal` representation. Floating-point numbers MUST NOT be used for money anywhere. The `Amount` component and any money rendering MUST format from decimal values.

#### Scenario: No floating-point money columns
- **WHEN** the schema is inspected for all money fields
- **THEN** every monetary column is `numeric`/`decimal` and none is a floating-point type

#### Scenario: API returns decimal-safe values
- **WHEN** an API route returns a monetary value
- **THEN** the value is serialized in a decimal-safe form (e.g. string) to avoid float coercion in transit

### Requirement: Dense UI and Infinite Scroll

List and table surfaces MUST be dense (maximizing visible information) and use generous infinite scroll via cursor pagination and an `IntersectionObserver` sentinel, not sparse numbered pagination.

#### Scenario: Dense table default
- **WHEN** a `DataTable` renders a list of transactions
- **THEN** rows use compact vertical spacing and multiple columns to maximize information density

#### Scenario: No numbered pagination UI
- **WHEN** a list surface is rendered
- **THEN** no page-number pagination control is shown; infinite scroll is used instead

### Requirement: Server-Side Secret Safety

Secrets (`ENCRYPTION_KEY`, `EMAIL_PROVIDER_API_KEY`, `DATABASE_URL`, bank credentials) MUST NEVER be exposed to the client, logged, or returned in API responses. Encryption keys and credentials MUST be used server-side only.

#### Scenario: Secrets not in client bundle
- **WHEN** the client bundle is inspected
- **THEN** no server-only secret (`ENCRYPTION_KEY`, `EMAIL_PROVIDER_API_KEY`, `DATABASE_URL`) is present

#### Scenario: Errors never leak secrets
- **WHEN** an API route returns an error
- **THEN** the error `message` and `details` contain no secret values, stack traces, or credentials
