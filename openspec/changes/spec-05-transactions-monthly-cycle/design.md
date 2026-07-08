## Context

Spec 01 defined the `Transaction` model (bankAccountId, householdId, amount signed decimal, currency, amountInMainCurrency decimal, transactionDate, description, merchantName nullable, categoryId nullable, billingMonth YYYY-MM, syncBatchId nullable), `Category` (householdId, name, type, isSystem, unique [householdId, name, type]), `ExchangeRate`, the `getBillingMonth(date)` utility (25th–24th rule), and the `convertAmount(amount, from, to, date) → Decimal` FX utility. Spec 04 defined the `storeRawTransactions(bankAccountId, householdId, rawTransactions[])` handoff signature and the `RawTransaction` shape (amount, currency, date, description, merchantName?, providerTransactionId). This spec implements that handoff and the transaction UI.

Key rules: income = positive amount, expense = negative amount (by sign). Pre-existing account balances are ignored — only in-window events are tracked. Manual categorization only. Dense UI with infinite scroll.

## Goals / Non-Goals

**Goals:**
- Implement `storeRawTransactions`: idempotent insert by `providerTransactionId`, FX conversion to main currency, billing-month derivation.
- Transaction list: dense DataTable, infinite scroll, filter by billing month + year, filter by member (all or specific), sortable, Amount display component.
- Manual categorization: assign/reassign category on any transaction.
- Category list: view household categories.
- Full UI states for every screen.
- All monetary values decimal (never float).

**Non-Goals:**
- Manual transaction creation/editing (global rule).
- Deleting or splitting transactions (immutable in MVP).
- Automatic categorization.
- Custom category creation UI (future).
- Text search (future).

## Decisions

### D1: Idempotent insert strategy
**Decision:** Use `providerTransactionId` (from `RawTransaction`) as the idempotency key. On `storeRawTransactions`, for each raw transaction, check if a `Transaction` with the same `bankAccountId` + `providerTransactionId` (stored in a `providerTransactionId` field — note: this requires adding a nullable `providerTransactionId` field to the Transaction model, or using `syncBatchId` composite; see D2) already exists. If yes, skip (no update — transactions are immutable in MVP). If no, insert. Use a database unique constraint on `(bankAccountId, providerTransactionId)` to enforce idempotency at the DB level.
**Rationale:** Re-syncs (manual or daily) will overlap date ranges. Without idempotency, transactions duplicate. DB-level constraint is the strongest guarantee.
**Alternatives considered:**
- Application-level check only: race conditions possible. Rejected.
- UPSERT (update on conflict): transactions are immutable in MVP, so skip-on-conflict is correct.

### D2: `providerTransactionId` storage
**Decision:** Add a nullable `providerTransactionId` field (string) to the `Transaction` model, with a unique constraint on `(bankAccountId, providerTransactionId)`. This is a schema addition to Spec 01's Transaction model. The field is nullable for forward-compat (future manual transactions wouldn't have it, though MVP has no manual transactions).
**Rationale:** Cleanest way to enforce idempotency. The `syncBatchId` from Spec 01 is a batch identifier, not per-transaction.
**Alternatives considered:**
- Hash of (date, amount, description) as idempotency key: fragile (same-day same-amount same-description collisions). Rejected.
- Use `syncBatchId` compositely: it's a batch id, not unique per transaction. Rejected.

### D3: FX conversion in storeRawTransactions
**Decision:** For each raw transaction, if `rawTransaction.currency === household.mainCurrency`, set `amountInMainCurrency = rawTransaction.amount` (no API call, short-circuit per Spec 01 FX utility). Otherwise, call `convertAmount(amount, rawTransaction.currency, household.mainCurrency, transactionDate)` to get the converted decimal. Store both `amount` (original) and `amountInMainCurrency` (converted). If the FX rate is unavailable and fetch fails, log the error and skip that transaction (do NOT insert with a null/wrong conversion) — the next sync will retry.
**Rationale:** Correctness over completeness. A transaction with a wrong conversion is worse than a delayed insert. The next sync covers the gap.
**Alternatives considered:**
- Insert with `amountInMainCurrency = null` and backfill later: complex, breaks dashboard aggregations. Rejected.
- Insert with a guessed rate: dangerous for financial data. Rejected.

### D4: Billing month derivation
**Decision:** Call `getBillingMonth(transactionDate)` (Spec 01 utility) for each transaction and store the result in `Transaction.billingMonth`. This is computed once at insert time and stored for query efficiency (the `(householdId, billingMonth, transactionDate)` index from Spec 01 supports the common query).
**Rationale:** Storing the derived value avoids recomputing per query and enables indexed filtering.
**Alternatives considered:**
- Compute at query time: slower, non-indexable. Rejected.

### D5: Transaction list filtering
**Decision:** The transaction list UI offers three filters: (1) billing month + year (default: the current billing month based on today's date), (2) member (default: all household members), (3) optional sort by date/amount. The API route `GET /api/households/[id]/transactions?billingMonth=YYYY-MM&memberId=&cursor=&limit=&sort=` returns paginated transactions. When `memberId` is omitted, returns transactions for all household members. The response uses the `{ data, error }` envelope with `nextCursor` for infinite scroll. Dense rows show: date, description, merchant, amount (Amount component), category badge, member name.
**Rationale:** Month + member are the two analysis dimensions the user specified. Defaulting to the current billing month avoids an overwhelming initial load.
**Alternatives considered:**
- No default month filter: loads all transactions ever — overwhelming. Rejected.
- Calendar-month filter (not billing month): contradicts the 25th–24th cycle rule. Rejected.

### D6: Categorization permissions
**Decision:** Any ACTIVE household member can assign or reassign a category to any transaction in the household (not just the transaction owner). Categories are household-scoped (predefined + future custom). The API route `PUT /api/households/[id]/transactions/[transactionId]/category` accepts `{ categoryId }` and updates the `Transaction.categoryId`. The category must belong to the same household.
**Rationale:** All members see all household data; categorization is collaborative. Restricting to the owner would block analysis.
**Alternatives considered:**
- Admin-only categorization: too restrictive for a collaborative household. Rejected.
- Owner-only: same. Rejected.

## Risks / Trade-offs

- [`providerTransactionId` schema addition] → Mitigation: acknowledged in the proposal Impact + Modified Capabilities. A Prisma migration adds the nullable field + unique constraint. Existing rows (none yet in MVP) are unaffected.
- [FX rate unavailable on first sync] → Mitigation: the transaction is skipped and retried on the next sync. The user sees fewer transactions initially but no incorrect conversions. No UI indicator is surfaced (the skipped transaction is not inserted, so there is nothing to display); the next sync covers the gap.
- [Re-sync does not update existing transactions] → Mitigation: transactions are immutable in MVP. If a provider corrects a transaction, the correction won't appear until a future "transaction update" spec. Acceptable for MVP.
- [No text search] → Mitigation: month + member filters cover the primary analysis use case. Text search is a future enhancement.
- [All members can recategorize] → Mitigation: audit log tracks who changed what. Collaborative editing is the intended UX.
