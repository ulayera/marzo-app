## Why

After the household balance is settled (Spec 07), members need to manually pay pending bills. The pay-bills checklist tracks bills with due dates and optional payment links, lets members mark them paid, and optionally links a bill to an expense transaction (if the bill appears in card statements). This replaces the spreadsheet column where users tracked "did I pay this?"

## What Changes

- Add bill CRUD: an admin can create a bill with title, amount, currency, due date, optional payment URL, and optional link to a transaction.
- Bills are auto-tagged with `billingMonth` (derived from due date via `getBillingMonth`).
- Bill list: filterable by billing month + status (PENDING | PAID); dense DataTable; due-date highlighting (overdue, due soon).
- Mark paid: any member can mark a bill as PAID (optionally recording who paid).
- Link to expense: a bill can be linked to a `Transaction` (if the bill is an expense that appears in card statements — e.g., utilities). Linked bills show the transaction reference.
- The credit-card-payment bill is NOT in statements (it's the act of paying the card) — the spec accommodates both linked and unlinked bills.
- Full UI states (empty, loading, error).
- i18n, theming, Spec 01 components.

## Capabilities

### New Capabilities
- `pay-bills-checklist`: Bill CRUD, due-date tracking, payment links, mark-paid, transaction linking, bill list with month + status filters.

### Modified Capabilities
- `system-design`: Consumes the `Bill` model (which includes `billingMonth` per the Spec 01 QA fix). No schema changes.

## Impact

- **Code**: `app/(app)/bills` route; `/api/households/[id]/bills/*` API routes.
- **Database**: No schema changes — uses `Bill` from Spec 01 (includes `billingMonth`, `linkedTransactionId`).
- **Future specs**: Spec 09 (notifications) sends BILL_DUE reminders.

## Non-goals

- Automatic bill payment (MVP is manual; automation is future).
- Recurring bill templates (future).
- Bill reminders via in-app push (email notifications only in MVP via Spec 09).
- Splitting a bill across members (a bill is a household expense; who pays is tracked via `paidByUserId` but the amount is not split).
- Bill attachments/receipts (future).
