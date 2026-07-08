## 1. i18n Keys

- [ ] 1.1 Add the `bills` namespace JSON under `i18n/locales/es/bills.json` with keys for `list.*`, `create.*`, `edit.*`, `pay.*`, `status.*` (PENDING, PAID, OVERDUE, DUE_SOON), `empty.*`, `errors.*` (cannotEditPaid, invalidBody)

## 2. Bill API Routes

- [ ] 2.1 Implement `GET /api/households/[id]/bills` (filter by billingMonth — default current, status — default ALL, cursor, limit; any ACTIVE member; return `{ data, nextCursor, error }` with decimal-safe strings)
- [ ] 2.2 Implement `POST /api/households/[id]/bills` (ADMIN only; validate body; if linkedTransactionId provided, verify Transaction.householdId == bill householdId else 400 `invalidLinkedTransaction`; auto-derive billingMonth via getBillingMonth; compute amountInMainCurrency via convertAmount (null if FX unavailable); return 201)
- [ ] 2.3 Implement `GET /api/households/[id]/bills/[billId]` (any ACTIVE member; 404 if not found)
- [ ] 2.4 Implement `PUT /api/households/[id]/bills/[billId]` (ADMIN only; if linkedTransactionId provided, verify same-household else 400 `invalidLinkedTransaction`; re-derive billingMonth on dueDate change; recompute amountInMainCurrency on amount/currency change OR if it is currently null and the rate is now available; 400 `cannotEditPaid` for PAID bills)
- [ ] 2.5 Implement `POST /api/households/[id]/bills/[billId]/pay` (any ACTIVE member; set status PAID, paidAt, paidByUserId)
- [ ] 2.6 Implement `POST /api/households/[id]/bills/[billId]/unpay` (ADMIN only; revert to PENDING, clear paidAt/paidByUserId)
- [ ] 2.7 Implement the bill FX backfill step in the daily sync schedule (after sync jobs complete in `POST /api/sync/run`, recompute any Bill with null `amountInMainCurrency` whose currency differs from the household main currency via `convertAmount`; leave null if rate still unavailable) — coordinate with Spec 04's sync route

## 3. Bills UI

- [ ] 3.1 Build the bills list page at `app/(app)/bills` using the Spec 01 DataTable (dense rows, infinite scroll, loading skeleton, EmptyState, ErrorState)
- [ ] 3.2 Add the billing-month + year filter (default current) and status filter (PENDING | PAID | ALL) using Spec 01 Select components
- [ ] 3.3 Render each row with: title, Amount component (main currency primary, original in tooltip), due date, status Badge with due-date highlighting (OVERDUE red, DUE SOON yellow, PAID green), "Pay" link button (if paymentUrl), linked-transaction reference (if present), "Mark paid" action
- [ ] 3.4 Build the create-bill form (admin only): title, amount, currency, dueDate, paymentUrl, linkedTransactionId (optional Select of household transactions); with inline validation, loading, and error states
- [ ] 3.5 Build the edit-bill form (admin only; disabled for PAID bills with an i18n message)
- [ ] 3.6 Add the mark-paid action (any member, loading state, immediate badge update) and the admin un-mark-paid action

## 4. Audit Logging

- [ ] 4.1 Write AuditLog entries for bill creation, edit, mark paid, un-mark paid (userId=actor, householdId, entity=Bill, metadata with action + changed fields)

## 5. Verification & Validation

- [ ] 5.1 Verify an admin can create a bill with billingMonth auto-derived from dueDate and amountInMainCurrency converted (or null if FX unavailable)
- [ ] 5.2 Verify a non-admin cannot create a bill (403)
- [ ] 5.3 Verify the bill list shows OVERDUE (red), DUE SOON (yellow, within 3 days), and PAID (green) badges
- [ ] 5.4 Verify filtering by billing month and status
- [ ] 5.5 Verify any member can mark a bill paid (status PAID, paidAt, paidByUserId set)
- [ ] 5.6 Verify only admin can un-mark a paid bill; non-admin gets 403
- [ ] 5.7 Verify editing a PAID bill returns 400 cannotEditPaid
- [ ] 5.8 Verify editing the due date re-derives billingMonth
- [ ] 5.9 Verify a bill with a paymentUrl shows a "Pay" link
- [ ] 5.10 Verify a bill linked to a transaction shows the reference
- [ ] 5.11 Verify the EmptyState shows when no bills match the filters
- [ ] 5.12 Verify all monetary values are decimal-safe strings in the API
- [ ] 5.13 Verify all bill text uses i18n keys (Spanish default)
- [ ] 5.14 Run `openspec validate spec-08-pay-bills-checklist` and resolve any reported issues
- [ ] 5.15 Verify a bill with null amountInMainCurrency is backfilled when edited and the FX rate is now available
- [ ] 5.16 Verify the daily sync schedule backfills bills with null amountInMainCurrency when the rate becomes available
- [ ] 5.17 Verify a bill whose FX rate is still unavailable remains null with the "conversion pending" note
