## ADDED Requirements

### Requirement: Dashboard Metrics

The dashboard SHALL display three key metrics for the selected billing month: **total income** (sum of `amountInMainCurrency` where `amount > 0`), **total expenses** (sum of `amountInMainCurrency` where `amount < 0`, displayed as an absolute value), and **net result** (income + expenses, showing surplus as positive or deficit as negative). All metrics MUST use the household main currency and be displayed via the Spec 01 Amount component. Aggregation MUST use decimal arithmetic — never floating point.

#### Scenario: Metrics for a billing month
- **WHEN** a member views the dashboard for billing month `2026-07`
- **THEN** total income, total expenses (absolute), and net result are displayed in the household main currency

#### Scenario: Surplus vs deficit
- **WHEN** income exceeds expenses for the month
- **THEN** the net result is displayed as a positive surplus
- **WHEN** expenses exceed income for the month
- **THEN** the net result is displayed as a negative deficit (e.g. "taken from savings")

#### Scenario: Decimal aggregation
- **WHEN** the aggregation is computed
- **THEN** SQL SUM on `amountInMainCurrency` (decimal/numeric) is used and no floating-point arithmetic is involved

### Requirement: Dashboard Filters

The dashboard SHALL support filtering by billing month + year (default: current billing month based on today's date via `getBillingMonth`) and by member (default: all household members aggregated). When a specific member is selected, metrics MUST reflect only transactions from that member's bank accounts. Filters MUST use the Spec 01 Select component.

#### Scenario: Default to current billing month
- **WHEN** a member opens the dashboard without selecting a month
- **THEN** metrics for the current billing month are shown

#### Scenario: Filter by a specific member
- **WHEN** a member selects a specific household member
- **THEN** metrics reflect only that member's transactions for the selected billing month

#### Scenario: Switch to a different billing month
- **WHEN** a member selects a different month/year
- **THEN** metrics update to reflect that billing month

### Requirement: Category Breakdown

The dashboard SHALL display a category breakdown table showing each EXPENSE category with non-zero totals for the selected billing month. Uncategorized expenses MUST appear as an "Uncategorized" row. The breakdown MUST use the Spec 01 DataTable (dense rows). Each row shows: category name, total amount (Amount component). Only categories with non-zero totals for the month are shown.

#### Scenario: Category breakdown displayed
- **WHEN** a member views the dashboard for a month with categorized and uncategorized expenses
- **THEN** each expense category with a non-zero total is listed with its total, plus an "Uncategorized" row if applicable

#### Scenario: No expenses for the month
- **WHEN** the selected billing month has no expenses
- **THEN** the category breakdown shows an EmptyState indicating no expenses for the period

### Requirement: Dashboard API Route

The application SHALL expose `GET /api/households/[id]/dashboard?billingMonth=YYYY-MM&memberId=` (any ACTIVE member). When `billingMonth` is omitted, defaults to the current billing month. Returns `{ data: { income, expenses, net, categoryBreakdown: [{ categoryId, name, total }] }, error: null }` with all monetary values as decimal-safe strings. The route MUST verify ACTIVE membership.

#### Scenario: Dashboard route response
- **WHEN** `GET /api/households/[id]/dashboard?billingMonth=2026-07` is called by an ACTIVE member
- **THEN** the response is `{ data: { income, expenses, net, categoryBreakdown }, error: null }` with amounts as decimal-safe strings

#### Scenario: Dashboard defaults to current month
- **WHEN** `GET /api/households/[id]/dashboard` is called without `billingMonth`
- **THEN** the response reflects the current billing month

#### Scenario: Non-member rejected
- **WHEN** a non-member calls the dashboard route
- **THEN** the API returns 403

### Requirement: Dashboard UI States

The dashboard MUST define loading, error, and empty states. While metrics are loading, skeleton placeholders MUST be shown for the three metric cards and the category breakdown. On error, an `ErrorState` with retry is shown. When the selected billing month has no transactions at all, an `EmptyState` is shown indicating no data for the period.

#### Scenario: Loading state
- **WHEN** the dashboard is loading
- **THEN** skeleton placeholders appear for the metric cards and category breakdown

#### Scenario: Error state
- **WHEN** the dashboard data fails to load
- **THEN** an `ErrorState` with a retry action is shown

#### Scenario: Empty month
- **WHEN** the selected billing month has no transactions
- **THEN** an `EmptyState` is shown indicating no financial activity for the period

### Requirement: Dashboard i18n Keys

All dashboard text MUST use i18n keys under the `dashboard` namespace. Spanish default. Keys MUST cover: `dashboard.title`, `dashboard.metrics.income`, `dashboard.metrics.expenses`, `dashboard.metrics.net`, `dashboard.metrics.surplus`, `dashboard.metrics.deficit`, `dashboard.filters.*`, `dashboard.breakdown.*`, `dashboard.empty.*`, `dashboard.errors.*`.

#### Scenario: All dashboard text via keys
- **WHEN** the dashboard is rendered
- **THEN** every user-facing string is produced by a `t('dashboard.*')` call
