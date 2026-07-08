## Context

Spec 01 defined the `Bill` model (householdId, title, amount, currency, amountInMainCurrency, dueDate, billingMonth derived from dueDate, paymentUrl nullable, status PENDING|PAID, paidAt, paidByUserId, linkedTransactionId, createdByUserId, timestamps, index on [householdId, billingMonth]). Spec 05 stores transactions that can be linked. This spec adds bill management + the payment checklist.

Per user decision: "bills are also expenses, some (like utilities) can be already inside the credit card or debit card statements, but others (like the actual credit card payment) are not in the statements but in the actual credit card is paid off." So bills can be linked to a transaction (utilities) or standalone (credit card payment).

## Goals / Non-Goals

**Goals:**
- Admin can create/edit bills (title, amount, currency, due date, payment URL, optional transaction link).
- Auto-derive `billingMonth` from due date.
- FX conversion for foreign-currency bills (amountInMainCurrency).
- Bill list: filter by billing month + status; due-date highlighting (overdue, due soon).
- Mark paid (any member; records paidByUserId + paidAt).
- Link to transaction (optional; shows reference).
- Full UI states.

**Non-Goals:**
- Automatic payment, recurring templates, in-app push reminders, bill splitting, attachments.

## Decisions

### D1: Bill creation permissions
**Decision:** Only ADMIN can create/edit bills. Any ACTIVE member can mark a bill as paid and view the list. Rationale: bills represent household obligations; the admin manages what needs paying, any member can report they paid it.
**Alternatives considered:**
- Any member can create bills: risk of duplicates/clutter. Rejected.
- Only admin can mark paid: too restrictive; any member might pay. Rejected.

### D2: FX conversion on bill creation
**Decision:** When a bill is created with a currency different from the household main currency, `amountInMainCurrency` is computed via `convertAmount(amount, currency, mainCurrency, dueDate)` (using the due date as the reference date). Same-currency short-circuits. If FX rate unavailable, the bill is still created with `amountInMainCurrency = null` (bills are obligations, not historical transactions — null is acceptable until the rate is available; the UI shows the original amount with a "conversion pending" note).
**Backfill:** A null `amountInMainCurrency` is recomputed (a) whenever the bill is edited (any field change triggers a recompute attempt), and (b) as a backfill step in the daily sync schedule (`POST /api/sync/run` recomputes any Bill with null `amountInMainCurrency` after sync jobs complete). If the rate is still unavailable, the bill remains null until the next attempt.
**Rationale:** Unlike transactions (where wrong conversion is worse than missing data), bills are forward-looking obligations. The original amount is always shown; the converted amount is a convenience for aggregation. The two backfill paths (edit + daily sync) ensure the converted amount appears once the rate is available without a dedicated cron job.
**Alternatives considered:**
- Reject the bill if FX unavailable: too strict for an obligation. Rejected.
- A separate cron for bill FX backfill: adds infrastructure. Rejected — reusing the daily sync schedule avoids it.

### D3: Due-date highlighting
**Decision:** The bill list highlights: OVERDUE (due date < today AND status PENDING) in red/danger; DUE SOON (due date within 3 days AND status PENDING) in yellow/warning; PAID in green/success. Uses the Spec 01 Badge component with semantic colors.
**Rationale:** Visual priority for the checklist.
**Alternatives considered:**
- No highlighting: defeats the purpose of a due-date checklist. Rejected.

### D4: Mark paid behavior
**Decision:** Marking a bill paid sets `status = PAID`, `paidAt = now()`, `paidByUserId = current user`. The action is reversible by an admin (un-mark paid → back to PENDING, clears paidAt/paidByUserId) in case of error. Any member can mark paid; only admin can un-mark.
**Rationale:** Anyone can pay; corrections are admin-controlled.
**Alternatives considered:**
- Irreversible: mistakes happen. Rejected.
- Any member can un-mark: risk of undoing a legitimate payment record. Rejected.

## Risks / Trade-offs

- [FX rate unavailable at bill creation] → Mitigation: bill created with null `amountInMainCurrency`; original amount shown; converted amount backfilled when rate is available (on next sync or manual refresh).
- [Linked transaction deleted] → Mitigation: transactions are immutable in MVP (Spec 05), so links don't break.
- [Overdue bills never auto-resolve] → Mitigation: the list surfaces them in red; admin must mark paid or delete. Acceptable for MVP.
