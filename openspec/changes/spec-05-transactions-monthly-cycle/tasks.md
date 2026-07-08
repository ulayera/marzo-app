## 1. i18n Keys

- [ ] 1.1 Add the `transactions` namespace JSON under `i18n/locales/es/transactions.json` with keys for `list.*` (title, filters, columns, monthLabels), `category.*`, `empty.*`, `errors.*`
- [ ] 1.2 Add the `categories` namespace JSON under `i18n/locales/es/categories.json` with keys for `list.*`, `type.*` (INCOME, EXPENSE), `flag.*` (system, custom)
- [ ] 1.3 Audit all transaction/category screens to confirm no hard-coded user-facing text

## 2. Schema Verification

- [ ] 2.1 Verify the `Transaction` model (from Spec 01) includes the nullable `providerTransactionId` field and the `@@unique([bankAccountId, providerTransactionId])` constraint; no additional migration is needed from this spec

## 3. storeRawTransactions Implementation

- [ ] 3.1 Implement `storeRawTransactions(bankAccountId, householdId, rawTransactions[])` in `lib/transactions/storeRawTransactions.ts`: for each raw transaction, check idempotency by `(bankAccountId, providerTransactionId)`, skip if exists (immutable in MVP)
- [ ] 3.2 Implement FX conversion: if `rawTransaction.currency === household.mainCurrency`, set `amountInMainCurrency = amount` (short-circuit); otherwise call `convertAmount(amount, currency, mainCurrency, transactionDate)` from Spec 01 FX utility. On FX rate unavailable + fetch failure, skip the transaction and log the error
- [ ] 3.3 Implement billing-month derivation: call `getBillingMonth(transactionDate)` from Spec 01 and store the result in `Transaction.billingMonth`
- [ ] 3.4 Insert the `Transaction` row with all fields (amount, currency, amountInMainCurrency as decimals, transactionDate, description, merchantName, bankAccountId, householdId, categoryId null, billingMonth, providerTransactionId, syncBatchId left null in MVP)
- [ ] 3.5 Verify the DB unique constraint on `(bankAccountId, providerTransactionId)` prevents duplicates on re-sync

## 4. Transaction API Routes

- [ ] 4.1 Implement `GET /api/households/[id]/transactions` (filter by billingMonth — default to current billing month when omitted, memberId, cursor, limit, sort — valid values: date, -date, amount, -amount; any ACTIVE member; returns `{ data, nextCursor, error }` with amounts as decimal-safe strings)
- [ ] 4.2 Implement `GET /api/households/[id]/transactions/[transactionId]` (single transaction; any ACTIVE member; 404 if not found or not in household)
- [ ] 4.3 Implement `PUT /api/households/[id]/transactions/[transactionId]/category` (accept `{ categoryId }`; validate category belongs to same household; allow null for uncategorized; any ACTIVE member; write AuditLog)
- [ ] 4.4 Implement `GET /api/households/[id]/categories` (list household categories with transaction counts; any ACTIVE member)
- [ ] 4.5 Verify all monetary values in responses are serialized as strings (decimal-safe)

## 5. Transaction List UI

- [ ] 5.1 Build the transactions list page at `app/(app)/transactions` using the Spec 01 DataTable (dense rows, sortable columns, infinite scroll, loading skeleton, EmptyState, ErrorState)
- [ ] 5.2 Add the billing-month + year filter (default: current billing month based on today) and the member filter (default: all household members) using Spec 01 Select components
- [ ] 5.3 Render each row with: date, description, merchant name (if present), Amount component (main currency primary, original in tooltip), category badge (if assigned), owning member name
- [ ] 5.4 Add the category selector per row (Spec 01 Select) to assign/reassign/clear categories, with loading state on save and inline error on failure
- [ ] 5.5 Implement infinite scroll via `useInfiniteScroll` (Spec 01) appending pages by `nextCursor`

## 6. Category List UI

- [ ] 6.1 Build the categories list page at `app/(app)/categories` using the Spec 01 DataTable (dense rows, loading skeleton, EmptyState)
- [ ] 6.2 Render each row with: category name, type badge (INCOME/EXPENSE), system/custom flag, transaction count

## 7. Audit Logging

- [ ] 7.1 Write `AuditLog` entries for transaction categorization changes (userId=actor, householdId, entity=Transaction, entityId, metadata with old + new categoryId)
- [ ] 7.2 Verify audit entries are created for each categorization change

## 8. Verification & Validation

- [ ] 8.1 Verify `storeRawTransactions` inserts new transactions with correct `amountInMainCurrency` (same-currency short-circuit + foreign-currency conversion) and `billingMonth` (25th-24th rule)
- [ ] 8.2 Verify re-syncing the same date range does NOT create duplicates (idempotency by `providerTransactionId`)
- [ ] 8.3 Verify a transaction whose FX rate is unavailable is skipped (not inserted with null/wrong conversion) and the error is logged
- [ ] 8.4 Verify the transaction list defaults to the current billing month and all members, with infinite scroll
- [ ] 8.5 Verify filtering by a specific member shows only that member's transactions
- [ ] 8.6 Verify filtering by a different billing month shows that month's transactions
- [ ] 8.7 Verify the EmptyState shows when no transactions match the filters
- [ ] 8.8 Verify the ErrorState shows on load failure with a retry action
- [ ] 8.9 Verify the Amount component shows the main currency primary and original currency in a tooltip
- [ ] 8.10 Verify a member can assign, reassign, and clear a category on any transaction
- [ ] 8.11 Verify assigning a category from another household returns 400 `invalidCategory`
- [ ] 8.12 Verify the category list shows transaction counts
- [ ] 8.13 Verify all monetary values in API responses are decimal-safe strings
- [ ] 8.14 Verify all transaction/category text uses i18n keys (Spanish default)
- [ ] 8.15 Run `openspec validate spec-05-transactions-monthly-cycle` and resolve any reported issues
