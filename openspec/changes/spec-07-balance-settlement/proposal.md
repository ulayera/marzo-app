## Why

Household members have individual incomes and expenses. Some end the month with a surplus, others with a deficit. The settlement feature calculates each member's net (income − expenses) and suggests the minimal set of transfers to settle debts between members. Without this, users must manually figure out who owes whom — the exact spreadsheet problem this app replaces.

## What Changes

- Add a settlement calculation: for a given billing month, compute each member's net (`sum(income) - sum(expenses)` in the household main currency).
- Implement the min-transfers algorithm: given members with surpluses and deficits, compute the minimal set of transfers to settle all debts.
- Store the result as a `Settlement` (DRAFT → CONFIRMED) with `SettlementTransfer` rows (fromMember, toMember, amount).
- UI: settlement view per billing month showing each member's net + suggested transfers; a "Confirm" action to lock the settlement.
- Link from the dashboard to the settlement view.
- Full UI states (empty, loading, error).
- i18n, theming, Spec 01 components.

## Capabilities

### New Capabilities
- `balance-settlement`: Per-member net calculation, min-transfers settlement algorithm, suggested transfers, settlement confirmation.

### Modified Capabilities
- `system-design`: Consumes the `Settlement`, `SettlementTransfer`, `Transaction` models. No schema changes.
- `summary-dashboard`: Adds a navigation link from the dashboard to the settlement page. No schema changes.

## Impact

- **Code**: `app/(app)/settlement` route; `/api/households/[id]/settlement?billingMonth=YYYY-MM` API routes (GET calculate, POST confirm); min-transfers algorithm in `lib/settlement/`.
- **Database**: No schema changes — uses `Settlement`, `SettlementTransfer` from Spec 01.
- **Future specs**: Spec 08 (bills) is the post-settlement action. Spec 09 (notifications) sends SETTLEMENT_READY.

## Non-goals

- Automatic bank transfers (MVP suggests transfers; users execute them manually).
- Recalculating a confirmed settlement (once CONFIRMED, the settlement is immutable; corrections require a new billing month).
- Splitting expenses by percentage (MVP is per-member net, not proportional split).
- Settlement history across multiple months (MVP shows one month at a time).
