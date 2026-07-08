## Why

Bank sync (Spec 04) acquires raw transactions but does not store them. This spec implements the `storeRawTransactions` handoff: converts raw amounts to the household main currency via the Spec 01 FX utility, derives the billing month (25th–24th cycle) for each transaction, idempotently inserts `Transaction` rows (by `providerTransactionId`), and exposes a transaction list UI with month/year + household/member filters. Without this spec, the app has no financial data to analyze, settle, or display on the dashboard.

## What Changes

- Implement `storeRawTransactions(bankAccountId, householdId, rawTransactions[])` — the handoff function Spec 04 calls.
- Idempotent transaction insert keyed on `providerTransactionId` (no duplicates on re-sync).
- FX conversion: for each raw transaction, compute `amountInMainCurrency` using the Spec 01 FX utility (`convertAmount`) at the transaction date; short-circuit if the transaction currency equals the household main currency.
- Billing month derivation: compute `billingMonth` (YYYY-MM) from `transactionDate` via the Spec 01 `getBillingMonth` utility (25th–24th rule).
- Income = positive `amount`, expense = negative `amount` (by sign — no explicit type field).
- Manual categorization: any ACTIVE member can assign/reassign a category to a transaction from the household's predefined + custom categories.
- Transaction list UI: dense DataTable with infinite scroll, filter by billing month + year, filter by member (all household or a specific member), sortable columns, Amount display (main currency primary, original in tooltip).
- Category list view: shows household categories with transaction counts.
- Full UI states (empty, loading, error) for every screen.
- All UI in `(app)`, i18n, theming, Spec 01 components.

## Capabilities

### New Capabilities
- `transactions-monthly-cycle`: Transaction storage from sync (idempotent, FX-converted, billing-month-tagged), manual categorization, transaction list with month/year + member filters, category list.

### Modified Capabilities
- `system-design`: CONSUMES the `Transaction` model's `providerTransactionId` field (nullable string) and the unique constraint on `(bankAccountId, providerTransactionId)`, both defined canonically in Spec 01. No additional schema changes.

## Impact

- **Code**: `app/(app)/transactions` list route; `app/(app)/categories` list route; `/api/households/[id]/transactions/*` and `/api/households/[id]/categories/*` API routes; `lib/transactions/storeRawTransactions.ts` (the Spec 04 handoff implementation).
- **Database**: No additional schema changes — `providerTransactionId` (nullable string) and the unique constraint on `(bankAccountId, providerTransactionId)` are defined canonically in Spec 01. This spec consumes them for idempotent inserts.
- **Dependencies**: None new.
- **Future specs**: Spec 06 (dashboard) aggregates transactions by billing month. Spec 07 (settlement) computes per-member nets. Spec 08 (bills) links bills to transactions.

## Non-goals

- Manual transaction creation or editing (global rule: transactions from sync only).
- Deleting or splitting synced transactions (immutable in MVP).
- Automatic categorization (manual only in MVP per user decision).
- Custom category creation (the foundation seeds predefined categories; custom category creation is deferred to a future spec).
- Real-time transaction updates (transactions appear after sync, not live).
- Transaction search by merchant/description (MVP filters by month/year/member only; text search is a future enhancement).
