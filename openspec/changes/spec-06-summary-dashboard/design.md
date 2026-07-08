## Context

Spec 05 stores `Transaction` rows with `billingMonth`, `amountInMainCurrency` (signed decimal), `householdId`, and `categoryId`. The dashboard aggregates these by billing month. The billing month is the 25th–24th cycle (Spec 01). The dashboard is the `/dashboard` route — the landing page after login (Spec 02 redirect logic).

## Goals / Non-Goals

**Goals:**
- Three metrics: total income, total expenses (absolute), net result for the selected billing month.
- Filter by billing month + year (default current).
- Filter by member (default all household; switch to isolate one member).
- Category breakdown table (expense categories with totals).
- Full UI states.
- Decimal-safe aggregation (never float).

**Non-Goals:**
- Charts, yearly aggregations, budgeting, export, real-time.

## Decisions

### D1: Aggregation query
**Decision:** A single API route `GET /api/households/[id]/dashboard?billingMonth=YYYY-MM&memberId=` computes all metrics server-side via SQL aggregation (SUM with CASE for income/expense) on `Transaction` joined to `BankAccount` (for member filtering). Returns `{ income, expenses, net, categoryBreakdown: [{ categoryId, name, total }] }`. When `memberId` is omitted, aggregates all household transactions. All amounts are `amountInMainCurrency` (already converted in Spec 05), returned as decimal-safe strings.
**Rationale:** Server-side aggregation avoids transferring all transactions to the client. Using `amountInMainCurrency` ensures consistent currency. Decimal-safe strings prevent float coercion.
**Alternatives considered:**
- Client-side aggregation from the transaction list: transfers too much data, slower. Rejected.

### D2: Default billing month
**Decision:** When `billingMonth` is omitted, default to the current billing month via `getBillingMonth(today)`. Same as Spec 05's transaction list default.
**Rationale:** Consistency with the transactions page; users see "this month" first.

### D3: Category breakdown scope
**Decision:** The category breakdown shows EXPENSE categories only (income is sign-based, not category-based, in MVP). Uncategorized expenses appear as an "Uncategorized" row. Only categories with non-zero totals for the month are shown (no empty rows).
**Rationale:** Income doesn't use categories in MVP (Spec 05 decision). Showing zero-total categories clutters the table.
**Alternatives considered:**
- Show all categories including zero: cluttered. Rejected.
- Include income categories: none are predefined in MVP. Rejected.

## Risks / Trade-offs

- [Aggregation on large transaction sets] → Mitigation: the `(householdId, billingMonth, transactionDate)` index from Spec 01 supports the query. Monthly volume is bounded.
- [Member filter requires BankAccount join] → Mitigation: the join is indexed (`bankAccountId` FK + the BankAccount→HouseholdMember relation). Acceptable for MVP volumes.
