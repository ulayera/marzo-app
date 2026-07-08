## ADDED Requirements

### Requirement: Per-Member Net Calculation

The settlement SHALL compute each ACTIVE household member's net for the selected billing month as `SUM(amountInMainCurrency)` across their transactions (positive = surplus, negative = deficit). Members with no transactions in the month have a net of 0. The household total net (sum of all member nets) MUST be displayed to show whether the household saved or drew from savings. All amounts MUST use the household main currency and decimal arithmetic.

#### Scenario: Member nets computed
- **WHEN** a member views the settlement for billing month `2026-07`
- **THEN** each ACTIVE member's net (income − expenses) is displayed in the household main currency

#### Scenario: Member with no transactions
- **WHEN** a member has no transactions in the billing month
- **THEN** their net is displayed as 0

#### Scenario: Household total net
- **WHEN** the settlement is viewed
- **THEN** the household total net (sum of all member nets) is displayed indicating surplus or deficit

### Requirement: Min-Transfers Settlement Algorithm

The settlement SHALL compute a near-minimal set of suggested transfers to settle debts between members using the greedy min-transfers algorithm: match the largest surplus to the largest deficit (deficits are negative, so ascending numeric order = largest absolute deficit first), create a transfer for the smaller amount, reduce both, and repeat until all nets are zeroed. Each transfer MUST specify `fromMemberId` (surplus member), `toMemberId` (deficit member), and `amount` (in main currency, decimal). The algorithm MUST produce at most N-1 transfers for N members with non-zero nets.

#### Scenario: Surplus transfers to deficit
- **WHEN** Dad has a net of +$1,000 and Mom has a net of −$500
- **THEN** a suggested transfer of $500 from Dad to Mom is generated (Dad keeps $500 surplus; Mom's deficit covered)

#### Scenario: Multiple surplus and deficit members
- **WHEN** members A (+$600), B (+$400) and C (−$500), D (−$500) exist
- **THEN** suggested transfers zero out all nets with at most 3 transfers (e.g., A→C $500, B→D $400, A→D $100)

#### Scenario: No transfers needed
- **WHEN** all members have a net of 0 or the household nets balance perfectly
- **THEN** no suggested transfers are generated and a "settled" state is shown

### Requirement: Settlement Lifecycle

The settlement SHALL support two states: DRAFT (computed on-demand, reflects latest transactions, not persisted) and CONFIRMED (persisted, immutable). `GET /api/households/[id]/settlement?billingMonth=YYYY-MM` MUST return the DRAFT calculation (or the stored CONFIRMED version if one exists). `POST /api/households/[id]/settlement?billingMonth=YYYY-MM` MUST confirm: persist a `Settlement(CONFIRMED)` + `SettlementTransfer` rows; the unique constraint on `(householdId, billingMonth)` MUST prevent re-confirming. Any ACTIVE member can view; only ADMIN can confirm. Once CONFIRMED, the settlement is immutable.

#### Scenario: Draft computed on-demand
- **WHEN** a member GETs the settlement for a month with no stored settlement
- **THEN** the nets and suggested transfers are computed on-the-fly and returned with `status = DRAFT` (not persisted)

#### Scenario: Confirm settlement
- **WHEN** an admin POSTs to confirm the settlement for a month
- **THEN** a `Settlement(CONFIRMED)` + `SettlementTransfer` rows are persisted and the response reflects the confirmed state

#### Scenario: Re-confirm rejected
- **WHEN** an admin attempts to confirm a settlement for a month that is already CONFIRMED
- **THEN** the API returns 400 with `error.code = 'alreadyConfirmed'`

#### Scenario: Non-admin cannot confirm
- **WHEN** a non-admin member attempts to confirm a settlement
- **THEN** the API returns 403

#### Scenario: Confirmed settlement immutable
- **WHEN** a settlement is CONFIRMED and transactions are later added for that month
- **THEN** the stored settlement does NOT change (GET returns the stored CONFIRMED version, not a re-computation)

### Requirement: Settlement Notification on Confirm

When a settlement is confirmed, the system MUST create a `Notification(SETTLEMENT_READY)` row for each **ACTIVE** household member with `userId` = member, `householdId` = household, `emailStatus = PENDING`, and `payload = { householdId, householdName, billingMonth, settlementId }`. The `Settlement`, `SettlementTransfer`, and `Notification` inserts MUST happen in a single DB transaction. `Settlement.calculatedAt` MUST be set to `now()` on confirm. Email dispatch is deferred to Spec 09.

#### Scenario: Notification created on confirm
- **WHEN** an admin confirms a settlement
- **THEN** a `Notification(SETTLEMENT_READY)` row is created for each ACTIVE household member with the documented payload (including `householdName`) and `emailStatus = PENDING`

#### Scenario: Removed members excluded
- **WHEN** a settlement is confirmed and a household has a REMOVED member
- **THEN** no `Notification(SETTLEMENT_READY)` row is created for the REMOVED member

### Requirement: Settlement API Routes

The application SHALL expose REST-like settlement API routes, all returning the Spec 01 `{ data, error }` envelope:

- `GET /api/households/[id]/settlement?billingMonth=YYYY-MM` — return DRAFT calculation or stored CONFIRMED settlement (any ACTIVE member). Response: `{ data: { billingMonth, status, householdNet, memberNets: [{ memberId, name, net }], transfers: [{ fromMemberId, fromName, toMemberId, toName, amount }] }, error: null }` with amounts as decimal-safe strings.
- `POST /api/households/[id]/settlement?billingMonth=YYYY-MM` — confirm (ADMIN only; 400 `alreadyConfirmed` if already confirmed; 403 for non-admin).

#### Scenario: GET settlement route
- **WHEN** `GET /api/households/[id]/settlement?billingMonth=2026-07` is called by an ACTIVE member
- **THEN** the response includes `status`, `householdNet`, `memberNets`, and `transfers` with decimal-safe string amounts

#### Scenario: POST confirm route
- **WHEN** an admin calls `POST /api/households/[id]/settlement?billingMonth=2026-07`
- **THEN** the settlement is persisted as CONFIRMED and the response reflects the confirmed state

### Requirement: Settlement UI

The settlement page SHALL display: the selected billing month + year filter (default current), the household total net, a member-nets list (dense DataTable with member name, net amount, surplus/deficit badge), and a suggested-transfers list (from → to, amount). A "Confirm Settlement" button (ADMIN only) MUST be present with a confirmation Modal. A link from the dashboard to the settlement page MUST exist. Full UI states (empty, loading, error) MUST be defined.

#### Scenario: View settlement
- **WHEN** a member opens the settlement page for a billing month
- **THEN** the household net, member nets, and suggested transfers are displayed

#### Scenario: Confirm with modal
- **WHEN** an admin clicks "Confirm Settlement"
- **THEN** a confirmation Modal appears; on confirm, the settlement is locked and the button is replaced with a "Confirmed" badge

#### Scenario: Settled state (no transfers needed)
- **WHEN** all member nets are 0 or the household nets balance perfectly
- **THEN** a "settled" indicator is shown in place of the transfers list (distinct from the EmptyState for no transactions)

#### Scenario: Empty state
- **WHEN** the billing month has no transactions
- **THEN** an EmptyState is shown indicating no data to settle for the period

#### Scenario: Loading state
- **WHEN** the settlement is being computed
- **THEN** skeleton placeholders appear for the member nets and transfers

#### Scenario: Error state
- **WHEN** the settlement calculation fails
- **THEN** an ErrorState with retry is shown

### Requirement: Settlement i18n Keys

All settlement text MUST use i18n keys under the `settlement` namespace. Spanish default. Keys: `settlement.title`, `settlement.householdNet`, `settlement.memberNets.*`, `settlement.transfers.*`, `settlement.confirm.*`, `settlement.settled.*`, `settlement.status.*` (DRAFT, CONFIRMED), `settlement.empty.*`, `settlement.errors.*`.

#### Scenario: All settlement text via keys
- **WHEN** the settlement page is rendered
- **THEN** every user-facing string is produced by a `t('settlement.*')` call

### Requirement: Settlement Audit Logging

The application MUST write an `AuditLog` entry when a settlement is confirmed. `userId` = admin actor, `householdId` = household, `entity = 'Settlement'`, `entityId` = settlementId, `metadata` includes `billingMonth` and `transferCount`.

#### Scenario: Settlement confirmation audited
- **WHEN** an admin confirms a settlement
- **THEN** an `AuditLog` row is created with `action = 'settlementConfirmed'`, `entity = 'Settlement'`, the settlement id, and metadata including billingMonth and transferCount
