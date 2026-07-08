## 1. i18n Keys

- [ ] 1.1 Add the `dashboard` namespace JSON under `i18n/locales/es/dashboard.json` with keys for `title`, `metrics.*` (income, expenses, net, surplus, deficit), `filters.*`, `breakdown.*`, `empty.*`, `errors.*`

## 2. Dashboard API Route

- [ ] 2.1 Implement `GET /api/households/[id]/dashboard?billingMonth=YYYY-MM&memberId=` (server-side SQL aggregation: SUM amountInMainCurrency with CASE for income/expense; join BankAccount for member filter; default billingMonth to current via getBillingMonth; any ACTIVE member; return `{ data: { income, expenses, net, categoryBreakdown }, error }` with decimal-safe strings)
- [ ] 2.2 Implement the category breakdown aggregation (expense categories with non-zero totals; "Uncategorized" row for null categoryId; only non-zero rows)

## 3. Dashboard UI

- [ ] 3.1 Build the dashboard page at `app/(app)/dashboard` with three metric cards (income, expenses absolute, net) using the Amount component
- [ ] 3.2 Add the billing-month + year filter (default: current) and member filter (default: all) using Spec 01 Select components
- [ ] 3.3 Build the category breakdown table using the Spec 01 DataTable (dense rows, category name + total amount)
- [ ] 3.4 Add loading skeleton (metric cards + breakdown), ErrorState with retry, and EmptyState for months with no transactions

## 4. Verification & Validation

- [ ] 4.1 Verify the dashboard shows correct income, expenses (absolute), and net for a billing month with mixed transactions
- [ ] 4.2 Verify the dashboard defaults to the current billing month
- [ ] 4.3 Verify filtering by a specific member isolates their transactions
- [ ] 4.4 Verify the category breakdown shows only non-zero expense categories + an Uncategorized row
- [ ] 4.5 Verify an empty month shows an EmptyState
- [ ] 4.6 Verify all monetary values are decimal-safe strings in the API response
- [ ] 4.7 Verify non-members get 403
- [ ] 4.8 Verify all dashboard text uses i18n keys (Spanish default)
- [ ] 4.9 Run `openspec validate spec-06-summary-dashboard` and resolve any reported issues
