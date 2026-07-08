## MODIFIED Requirements

### Requirement: Master Database Schema — Transaction

The `Transaction` model is defined canonically in Spec 01 (system-design), which includes the `providerTransactionId` (nullable string) field and the unique constraint on `(bankAccountId, providerTransactionId)` for idempotent transaction inserts from bank sync. This spec CONSUMES that field for idempotent storage in `storeRawTransactions`. The `storeRawTransactions` function MUST use `(bankAccountId, providerTransactionId)` as the idempotency key: if a `Transaction` with that key already exists, the insert MUST be skipped (no update — transactions are immutable in MVP). No additional schema changes are made by this spec. See Spec 01 for the canonical field list and model definition.

#### Scenario: Idempotent insert uses providerTransactionId
- **WHEN** `storeRawTransactions` inserts a transaction with a `providerTransactionId` that already exists for the same `bankAccountId`
- **THEN** the database rejects the duplicate via the Spec 01 unique constraint on `(bankAccountId, providerTransactionId)`

## ADDED Requirements

### Requirement: storeRawTransactions Handoff

The application MUST implement `storeRawTransactions(bankAccountId, householdId, rawTransactions[]) → Promise<void>` — the handoff function Spec 04's sync job calls. For each `RawTransaction`, the function MUST: (1) check idempotency by `(bankAccountId, providerTransactionId)` — skip if a `Transaction` with that key already exists (no update, transactions are immutable in MVP); (2) compute `amountInMainCurrency` via the Spec 01 FX utility; (3) derive `billingMonth` via the Spec 01 `getBillingMonth` utility; (4) insert a new `Transaction` row. A database unique constraint on `(bankAccountId, providerTransactionId)` MUST enforce idempotency at the DB level.

#### Scenario: New transactions inserted
- **WHEN** `storeRawTransactions` is called with raw transactions not already in the DB
- **THEN** a `Transaction` row is created for each with `amount`, `currency`, `amountInMainCurrency`, `billingMonth`, `transactionDate`, `description`, `merchantName`, `bankAccountId`, `householdId`, `providerTransactionId`, and `syncBatchId` left null in MVP (the handoff signature does not pass a batch id; the SyncJob itself is the audit trail)

#### Scenario: Duplicate transactions skipped
- **WHEN** `storeRawTransactions` is called with a raw transaction whose `(bankAccountId, providerTransactionId)` already exists
- **THEN** the existing row is NOT updated and no duplicate is created

#### Scenario: DB unique constraint enforced
- **WHEN** a second `Transaction` with the same `(bankAccountId, providerTransactionId)` is inserted
- **THEN** the database rejects it with a unique-constraint violation

### Requirement: FX Conversion on Insert

For each raw transaction, if `rawTransaction.currency === household.mainCurrency`, `amountInMainCurrency` MUST be set equal to `rawTransaction.amount` (short-circuit, no API call). Otherwise, `amountInMainCurrency` MUST be computed via `convertAmount(amount, rawTransaction.currency, household.mainCurrency, transactionDate)` using the Spec 01 FX utility. If the FX rate is unavailable and the fetch fails, the transaction MUST be skipped (not inserted with a null or wrong conversion) and the error logged; the next sync will retry. Both `amount` (original) and `amountInMainCurrency` (converted) MUST use decimal/numeric types — never floating point.

#### Scenario: Same-currency short-circuit
- **WHEN** a raw transaction's currency equals the household main currency
- **THEN** `amountInMainCurrency` is set equal to `amount` and no FX API call is made

#### Scenario: Foreign-currency conversion
- **WHEN** a raw transaction is in USD and the household main currency is CLP
- **THEN** `amountInMainCurrency` is computed by `convertAmount` using the exchange rate for the transaction date and stored as a decimal

#### Scenario: FX rate unavailable skips insert
- **WHEN** the FX rate for a transaction's date is unavailable and the fetch fails
- **THEN** the transaction is not inserted, the error is logged, and the next sync will retry

#### Scenario: Decimal types for money
- **WHEN** a `Transaction` row is inspected
- **THEN** `amount` and `amountInMainCurrency` are decimal/numeric, not floating point

### Requirement: Billing Month Derivation

Each transaction's `billingMonth` MUST be computed via the Spec 01 `getBillingMonth(transactionDate)` utility implementing the 25th–24th rule. A transaction on day-of-month >= 25 belongs to its own calendar month; a transaction on day-of-month <= 24 belongs to the previous calendar month. The result MUST be a `YYYY-MM` string stored on the `Transaction`.

#### Scenario: Late-month transaction
- **WHEN** a transaction occurs on July 26, 2026
- **THEN** `billingMonth` is `"2026-07"`

#### Scenario: Early-month transaction
- **WHEN** a transaction occurs on July 24, 2026
- **THEN** `billingMonth` is `"2026-06"`

#### Scenario: January early-month wrap
- **WHEN** a transaction occurs on January 10, 2026
- **THEN** `billingMonth` is `"2025-12"`

### Requirement: Income and Expense by Sign

Income (positive `amount`) and expense (negative `amount`) MUST be determined by the sign of the `Transaction.amount` field. There MUST be no explicit `type` field on `Transaction`. Credit card activity from sync is included as expenses (negative amounts). Stock sales and savings withdrawals are positive amounts and therefore count as income.

#### Scenario: Positive amount is income
- **WHEN** a transaction has a positive `amount`
- **THEN** it is classified as income

#### Scenario: Negative amount is expense
- **WHEN** a transaction has a negative `amount`
- **THEN** it is classified as an expense

### Requirement: Transaction List View

Any `ACTIVE` household member SHALL be able to view all transactions for the household. The list MUST use the Spec 01 DataTable (dense rows, sortable columns, infinite scroll via cursor + sentinel, loading skeleton, EmptyState, ErrorState). The list MUST show per row: transaction date, description, merchant name (if present), Amount (main currency primary, original in tooltip via the Spec 01 Amount component), category badge (if assigned), and owning member name. The list MUST support filtering by billing month + year (default: current billing month based on today's date) and by member (default: all household members). The list MUST NOT show numbered pagination (infinite scroll only, per Spec 01 dense-UI rule).

#### Scenario: View transactions for current billing month
- **WHEN** a member opens the transactions page
- **THEN** transactions for the current billing month (based on today's date) are shown for all household members

#### Scenario: Filter by a specific member
- **WHEN** a member selects a specific household member from the filter
- **THEN** only transactions from that member's bank accounts are shown

#### Scenario: Filter by a different billing month
- **WHEN** a member selects a different month/year
- **THEN** transactions for that billing month are shown

#### Scenario: Infinite scroll loads more
- **WHEN** a member scrolls near the bottom and `nextCursor` is present
- **THEN** the next page of transactions is appended without replacing existing rows

#### Scenario: Empty state
- **WHEN** no transactions match the filters
- **THEN** an `EmptyState` is shown with a message indicating no transactions for the selected period

#### Scenario: Loading skeleton
- **WHEN** the transaction list is loading
- **THEN** skeleton rows are shown (not empty content)

#### Scenario: Error state
- **WHEN** the transaction list fails to load
- **THEN** an `ErrorState` with a retry action is shown

#### Scenario: Amount display with original currency tooltip
- **WHEN** a transaction in a foreign currency is rendered
- **THEN** the main-currency amount is shown as primary text and the original-currency amount appears in a tooltip on hover/focus

### Requirement: Manual Categorization

Any `ACTIVE` household member SHALL be able to assign or reassign a category to any transaction in the household. The category MUST belong to the same household (predefined or custom). The API route `PUT /api/households/[id]/transactions/[transactionId]/category` accepts `{ categoryId }` and updates `Transaction.categoryId`. Setting `categoryId = null` (uncategorized) MUST be allowed. The UI MUST offer a category selector (Spec 01 Select component) per transaction row or in a detail view. An audit log entry MUST be written for categorization changes.

#### Scenario: Assign a category
- **WHEN** a member selects a category for an uncategorized transaction
- **THEN** `Transaction.categoryId` is updated and a category badge appears on the row

#### Scenario: Reassign a category
- **WHEN** a member selects a different category for a categorized transaction
- **THEN** `Transaction.categoryId` is updated and the badge changes

#### Scenario: Uncategorized (null category)
- **WHEN** a member clears the category on a transaction
- **THEN** `Transaction.categoryId` is set to null and no badge is shown

#### Scenario: Category from another household rejected
- **WHEN** a member assigns a `categoryId` belonging to a different household
- **THEN** the API returns 400 with `error.code = 'invalidCategory'`

#### Scenario: Categorization audited
- **WHEN** a transaction's category is changed
- **THEN** an `AuditLog` entry is created with `action = 'categoryChanged'`, `entity = 'Transaction'`, `entityId` = transactionId, and metadata including the old and new `categoryId`

### Requirement: Category List View

Any `ACTIVE` household member SHALL be able to view the household's categories. The list MUST show: category name, type (INCOME | EXPENSE), system/custom flag, and transaction count (number of transactions currently assigned to that category). The list MUST use the Spec 01 DataTable (dense, loading skeleton, EmptyState — empty only if the household has no categories, which should not happen since onboarding seeds predefined categories).

#### Scenario: View categories
- **WHEN** a member opens the categories page
- **THEN** all household categories are listed with name, type, system/custom flag, and transaction count

#### Scenario: Transaction count displayed
- **WHEN** a category has 15 transactions assigned
- **THEN** the count "15" is shown next to the category

### Requirement: Transaction API Routes

The application SHALL expose REST-like transaction API routes, all returning the Spec 01 `{ data, error }` envelope and enforcing auth + membership:

- `GET /api/households/[id]/transactions?billingMonth=YYYY-MM&memberId=&cursor=&limit=&sort=` — list transactions (any ACTIVE member). When `billingMonth` is omitted, the API MUST default to the current billing month (based on today's date). Valid `sort` values are `date` (default, ascending), `-date` (descending), `amount`, `-amount`. Returns `{ data: [ ... ], nextCursor, error: null }`.
- `GET /api/households/[id]/transactions/[transactionId]` — get a single transaction (any ACTIVE member). Returns 404 if the transaction does not exist or does not belong to the household.
- `PUT /api/households/[id]/transactions/[transactionId]/category` — assign/reassign category (any ACTIVE member; body: `{ categoryId }`).
- `GET /api/households/[id]/categories` — list household categories with transaction counts (any ACTIVE member).

All routes MUST verify the caller is an ACTIVE member. Monetary values MUST be serialized in a decimal-safe form (e.g. string) to avoid float coercion in transit (per Spec 01).

#### Scenario: List transactions route
- **WHEN** `GET /api/households/[id]/transactions?billingMonth=2026-07` is called by an ACTIVE member
- **THEN** the response is `{ data: [ { transactionId, date, description, merchantName, amount, currency, amountInMainCurrency, categoryId, memberName } ], nextCursor, error: null }` with amounts as decimal-safe strings

#### Scenario: Billing month defaults to current when omitted
- **WHEN** `GET /api/households/[id]/transactions` is called without a `billingMonth` parameter
- **THEN** the API returns transactions for the current billing month (based on today's date)

#### Scenario: Filter by member
- **WHEN** `GET /api/households/[id]/transactions?memberId=<userId>` is called
- **THEN** only transactions from that member's bank accounts are returned

#### Scenario: Sort by date descending
- **WHEN** `GET /api/households/[id]/transactions?sort=-date` is called
- **THEN** transactions are returned ordered by `transactionDate` descending

#### Scenario: Transaction not found
- **WHEN** `GET /api/households/[id]/transactions/[nonExistentId]` is called
- **THEN** the response is 404 with `error.code = 'notFound'`

#### Scenario: Assign category route
- **WHEN** `PUT /api/households/[id]/transactions/[transactionId]/category` is called with a valid `categoryId`
- **THEN** the transaction's `categoryId` is updated and the response is `{ data: { transactionId, categoryId }, error: null }`

#### Scenario: Non-member rejected
- **WHEN** a user who is not an ACTIVE member calls any transaction route
- **THEN** the API returns 403

#### Scenario: Decimal-safe money serialization
- **WHEN** any transaction API response is inspected
- **THEN** monetary values are serialized as strings (not JSON numbers) to avoid float coercion

### Requirement: Transaction i18n Keys

All transaction user-facing text MUST be rendered via i18n keys under the `transactions` and `categories` namespaces. Spanish default. Keys MUST cover: `transactions.list.*` (title, filters, columns), `transactions.category.*`, `transactions.empty.*`, `transactions.errors.*`, `categories.list.*`, `categories.type.*` (INCOME, EXPENSE), `categories.flag.*` (system, custom).

#### Scenario: All transaction text via keys
- **WHEN** any transaction or category screen is rendered
- **THEN** every user-facing string is produced by a `t('transactions.*')` or `t('categories.*')` call

### Requirement: Transaction Audit Logging

The application MUST write `AuditLog` entries for transaction mutations (categorization changes). `userId` = actor, `householdId` = household, `entity = 'Transaction'`, `entityId` = transactionId, `metadata` includes old and new `categoryId`. Transaction inserts from sync do not write per-transaction audit logs (they are system-generated; the SyncJob itself is the audit trail).

#### Scenario: Categorization audited
- **WHEN** a transaction's category is changed
- **THEN** an `AuditLog` row is created with the actor's userId, householdId, and old/new category in metadata
