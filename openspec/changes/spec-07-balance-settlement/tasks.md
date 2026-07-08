## 1. i18n Keys

- [ ] 1.1 Add the `settlement` namespace JSON under `i18n/locales/es/settlement.json` with keys for `title`, `householdNet`, `memberNets.*`, `transfers.*`, `confirm.*`, `status.*` (DRAFT, CONFIRMED), `empty.*`, `errors.*`

## 2. Min-Transfers Algorithm

- [ ] 2.1 Implement the greedy min-transfers algorithm in `lib/settlement/minTransfers.ts` (sort surpluses desc, deficits asc, match largest to largest, create transfer for smaller amount, reduce, repeat; max N-1 transfers)

## 3. Settlement API Routes

- [ ] 3.1 Implement `GET /api/households/[id]/settlement?billingMonth=YYYY-MM` (if CONFIRMED settlement exists, return stored; otherwise compute DRAFT on-demand: per-member net via SUM(amountInMainCurrency) joined to BankAccount→HouseholdMember, run minTransfers; any ACTIVE member; return `{ data: { billingMonth, status, householdNet, memberNets, transfers }, error }` with decimal-safe strings)
- [ ] 3.2 Implement `POST /api/households/[id]/settlement?billingMonth=YYYY-MM` (ADMIN only; persist Settlement CONFIRMED with calculatedAt=now() + SettlementTransfer rows + Notification(SETTLEMENT_READY) for each ACTIVE member with payload {householdId, householdName, billingMonth, settlementId} in a single DB transaction; 400 `alreadyConfirmed` if exists; 403 for non-admin; write AuditLog)

## 4. Settlement UI

- [ ] 4.1 Build the settlement page at `app/(app)/settlement` with the billing-month + year filter (default current)
- [ ] 4.2 Display the household total net, member-nets DataTable (name, net, surplus/deficit Badge), and suggested-transfers list (from → to, Amount component)
- [ ] 4.3 Add the "Confirm Settlement" button (ADMIN only) with a confirmation Modal; on confirm, lock the settlement and show a "Confirmed" badge
- [ ] 4.4 Add a link from the dashboard to the settlement page
- [ ] 4.5 Add loading skeleton, EmptyState (no transactions), and ErrorState with retry

## 5. Verification & Validation

- [ ] 5.1 Verify per-member nets are computed correctly (income − expenses per member for the billing month)
- [ ] 5.2 Verify the min-transfers algorithm produces the correct minimal transfer set (test the Dad/Mom example: +1000/−500 → $500 transfer)
- [ ] 5.3 Verify GET returns DRAFT (not persisted) for a month with no stored settlement
- [ ] 5.4 Verify POST confirm persists Settlement + SettlementTransfer rows and creates Notification(SETTLEMENT_READY) for each member
- [ ] 5.5 Verify re-confirming a confirmed month returns 400 `alreadyConfirmed`
- [ ] 5.6 Verify non-admin confirm returns 403
- [ ] 5.7 Verify a confirmed settlement is immutable (adding transactions does not change it)
- [ ] 5.8 Verify the UI shows surplus/deficit badges, the confirm modal, and the confirmed badge
- [ ] 5.9 Verify the empty state for a month with no transactions
- [ ] 5.10 Verify all monetary values are decimal-safe strings in the API
- [ ] 5.11 Verify all settlement text uses i18n keys (Spanish default)
- [ ] 5.12 Run `openspec validate spec-07-balance-settlement` and resolve any reported issues
