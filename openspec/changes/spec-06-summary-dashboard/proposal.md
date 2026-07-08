## Why

Transactions are stored (Spec 05) but users have no way to see the monthly bottom line. The dashboard is the primary analysis surface — showing total income, total expenses, and net result for a billing month, filterable by member. This is the screen users open first to understand their household's financial health.

## What Changes

- Add a summary dashboard at `/dashboard` (the `(app)` landing route).
- Display three key metrics for the selected billing month: total income (sum of positive `amountInMainCurrency`), total expenses (sum of negative `amountInMainCurrency`, shown as absolute), and net result (income + expenses, showing surplus or deficit).
- Filter by billing month + year (default: current billing month).
- Filter by member (default: all household members aggregated; switch to a specific member to isolate their numbers).
- Breakdown by category: list of expense categories with their totals for the month (dense table).
- All amounts displayed via the Spec 01 Amount component (main currency primary).
- Full UI states (empty, loading, error).
- i18n, theming, Spec 01 components.

## Capabilities

### New Capabilities
- `summary-dashboard`: Monthly financial summary with income/expense/net metrics, category breakdown, month/year + member filters.

### Modified Capabilities
- `system-design`: Consumes the `Transaction` and `Category` models and aggregation queries. No schema changes.

## Impact

- **Code**: `app/(app)/dashboard` route; `GET /api/households/[id]/dashboard?billingMonth=YYYY-MM&memberId=` API route; aggregation query logic.
- **Database**: No schema changes — aggregates `Transaction` rows by `billingMonth` + `householdId` (+ optional `memberId` via BankAccount join).
- **Future specs**: Spec 07 (settlement) links from the dashboard. Spec 08 (bills) may surface pending bill counts on the dashboard.

## Non-goals

- Charts/graphs (MVP is numeric metrics + a category breakdown table; visualizations are future).
- Yearly or multi-month aggregations (MVP is single billing month at a time).
- Budgeting or spending targets (future).
- Export to CSV/PDF (future).
- Real-time updates (data reflects the last sync; no live refresh).
