## ADDED Requirements

### Requirement: Bill Creation

An ADMIN SHALL be able to create a bill by providing `title`, `amount` (decimal), `currency` (ISO 4217), `dueDate` (date), optional `paymentUrl`, and optional `linkedTransactionId`. If `linkedTransactionId` is provided, the referenced `Transaction.householdId` MUST equal the bill's `householdId`; otherwise the API returns 400 with `bills.errors.invalidLinkedTransaction`. The system MUST auto-derive `billingMonth` from `dueDate` via the Spec 01 `getBillingMonth` utility. If the bill currency differs from the household main currency, `amountInMainCurrency` MUST be computed via `convertAmount(amount, currency, mainCurrency, dueDate)`; if the FX rate is unavailable, the bill is created with `amountInMainCurrency = null` (the original amount is always shown). Money columns MUST be decimal. The `createdByUserId` MUST be set to the admin.

#### Scenario: Create a bill
- **WHEN** an admin creates a bill with a title, amount, currency, and due date
- **THEN** a `Bill` row is created with `billingMonth` derived from `dueDate`, `amountInMainCurrency` converted (or null if FX unavailable), `status = PENDING`, and `createdByUserId` set

#### Scenario: Bill with a payment URL
- **WHEN** an admin creates a bill with a `paymentUrl`
- **THEN** the bill list shows a "Pay" link to that URL

#### Scenario: Bill linked to a transaction
- **WHEN** an admin creates a bill with a `linkedTransactionId` (e.g. a utility bill that appears in card statements)
- **THEN** the bill shows a reference to the linked transaction in the list

#### Scenario: Linked transaction from another household rejected
- **WHEN** an admin creates a bill with a `linkedTransactionId` referencing a transaction in another household
- **THEN** the API returns 400 with `error.code = 'invalidLinkedTransaction'` and no bill is created

#### Scenario: Non-admin cannot create a bill
- **WHEN** a non-admin member attempts to create a bill
- **THEN** the API returns 403

#### Scenario: FX rate unavailable at creation
- **WHEN** an admin creates a foreign-currency bill and the FX rate for the due date is unavailable
- **THEN** the bill is created with `amountInMainCurrency = null` and the original-currency amount is shown with a "conversion pending" note

### Requirement: Bill List View

Any `ACTIVE` household member SHALL be able to view all household bills. The list MUST use the Spec 01 DataTable (dense rows, infinite scroll, loading skeleton, EmptyState, ErrorState). The list MUST support filtering by billing month + year (default: current billing month) and by status (PENDING | PAID | ALL). Each row MUST show: title, amount (Amount component, main currency primary, original in tooltip), due date, status badge with due-date highlighting (OVERDUE = red/danger if past due + PENDING; DUE SOON = yellow/warning if within 3 days + PENDING; PAID = green/success), payment link ("Pay" button if `paymentUrl` exists), and a linked-transaction reference if present.

#### Scenario: View bills for a billing month
- **WHEN** a member opens the bills page
- **THEN** bills for the current billing month are shown with title, amount, due date, status badge, and payment link

#### Scenario: Overdue bill highlighted
- **WHEN** a PENDING bill has a due date before today
- **THEN** it is highlighted with an OVERDUE badge in red/danger

#### Scenario: Due-soon bill highlighted
- **WHEN** a PENDING bill has a due date within 3 days
- **THEN** it is highlighted with a DUE SOON badge in yellow/warning

#### Scenario: Paid bill highlighted
- **WHEN** a bill is PAID
- **THEN** it shows a PAID badge in green/success

#### Scenario: Filter by status
- **WHEN** a member filters by status PENDING
- **THEN** only pending bills are shown

#### Scenario: Empty state
- **WHEN** no bills match the filters
- **THEN** an EmptyState is shown prompting the admin to add a bill

#### Scenario: Loading skeleton
- **WHEN** the bill list is loading
- **THEN** skeleton rows are shown

#### Scenario: Error state
- **WHEN** the bill list fails to load
- **THEN** an ErrorState with retry is shown

### Requirement: Mark Bill as Paid

Any `ACTIVE` household member SHALL be able to mark a PENDING bill as paid. Marking paid MUST set `status = PAID`, `paidAt = now()`, and `paidByUserId` to the current user. An ADMIN SHALL be able to un-mark a paid bill (revert to PENDING, clear `paidAt` and `paidByUserId`) in case of error. The UI MUST show a loading state on the mark-paid action and update the badge immediately on success.

#### Scenario: Mark a bill paid
- **WHEN** a member clicks "Mark paid" on a PENDING bill
- **THEN** `status = PAID`, `paidAt` is set, `paidByUserId` is set to the member, and the badge updates to PAID

#### Scenario: Admin un-marks a paid bill
- **WHEN** an admin reverts a PAID bill to PENDING
- **THEN** `status = PENDING`, `paidAt` and `paidByUserId` are cleared, and the badge reverts

#### Scenario: Non-admin cannot un-mark
- **WHEN** a non-admin member attempts to un-mark a paid bill
- **THEN** the API returns 403

### Requirement: Bill FX Backfill

When a bill is created with `amountInMainCurrency = null` (FX rate unavailable at creation time), the system MUST recompute `amountInMainCurrency` via `convertAmount(amount, currency, mainCurrency, dueDate)` when the bill is subsequently edited (any field change triggers a recompute attempt) AND as a backfill step in the daily sync schedule (`POST /api/sync/run` route, after sync jobs complete, recompute any Bill with null `amountInMainCurrency` whose currency differs from the household main currency). If the rate is still unavailable, the bill remains null until the next attempt. The UI MUST continue to show the original-currency amount with a "conversion pending" note while `amountInMainCurrency` is null.

#### Scenario: Backfill on bill edit
- **WHEN** an admin edits a bill whose `amountInMainCurrency` is null and the FX rate is now available
- **THEN** `amountInMainCurrency` is recomputed and stored, and the "conversion pending" note is removed

#### Scenario: Backfill in daily sync
- **WHEN** the daily sync schedule runs and a bill exists with `amountInMainCurrency = null` and the FX rate is now available
- **THEN** `amountInMainCurrency` is recomputed and stored

#### Scenario: Rate still unavailable
- **WHEN** a backfill attempt runs and the FX rate is still unavailable
- **THEN** `amountInMainCurrency` remains null and the "conversion pending" note persists until the next attempt

### Requirement: Bill Editing

An ADMIN SHALL be able to edit a bill's `title`, `amount`, `currency`, `dueDate`, `paymentUrl`, and `linkedTransactionId`. If `linkedTransactionId` is provided on edit, the referenced `Transaction.householdId` MUST equal the bill's `householdId`; otherwise the API returns 400 with `bills.errors.invalidLinkedTransaction`. Editing the `dueDate` MUST re-derive `billingMonth`. Editing the `amount` or `currency` MUST recompute `amountInMainCurrency`. A PAID bill SHALL NOT be editable (return 400 with `bills.errors.cannotEditPaid`) to preserve payment records.

#### Scenario: Edit a pending bill
- **WHEN** an admin edits a PENDING bill's title or amount
- **THEN** the bill is updated and `amountInMainCurrency` is recomputed if needed

#### Scenario: Edit due date re-derives billing month
- **WHEN** an admin changes a bill's due date
- **THEN** `billingMonth` is re-derived via `getBillingMonth`

#### Scenario: Edit with linked transaction from another household
- **WHEN** an admin edits a bill and sets `linkedTransactionId` to a transaction in another household
- **THEN** the API returns 400 with `error.code = 'invalidLinkedTransaction'`

#### Scenario: Paid bill not editable
- **WHEN** an admin attempts to edit a PAID bill
- **THEN** the API returns 400 with `error.code = 'cannotEditPaid'`

### Requirement: Bill API Routes

The application SHALL expose REST-like bill API routes, all returning the Spec 01 `{ data, error }` envelope:

- `GET /api/households/[id]/bills?billingMonth=YYYY-MM&status=PENDING|PAID|ALL&cursor=&limit=` — list bills (any ACTIVE member; default billingMonth = current, status = ALL)
- `POST /api/households/[id]/bills` — create a bill (ADMIN only; body: `{ title, amount, currency, dueDate, paymentUrl?, linkedTransactionId? }`)
- `GET /api/households/[id]/bills/[billId]` — get a single bill (any ACTIVE member; 404 if not found)
- `PUT /api/households/[id]/bills/[billId]` — edit a PENDING bill (ADMIN only; 400 `cannotEditPaid` for PAID bills)
- `POST /api/households/[id]/bills/[billId]/pay` — mark paid (any ACTIVE member; sets paidAt + paidByUserId)
- `POST /api/households/[id]/bills/[billId]/unpay` — un-mark paid (ADMIN only; reverts to PENDING)

All routes MUST verify ACTIVE membership; monetary values MUST be decimal-safe strings.

#### Scenario: Create bill route
- **WHEN** `POST /api/households/[id]/bills` is called by an admin with a valid body
- **THEN** the response is `{ data: { billId, billingMonth, status }, error: null }` with 201

#### Scenario: Mark paid route
- **WHEN** `POST /api/households/[id]/bills/[billId]/pay` is called by an ACTIVE member
- **THEN** the bill is marked PAID and the response is `{ data: { billId, status: 'PAID', paidAt, paidByUserId }, error: null }`

#### Scenario: Non-admin create rejected
- **WHEN** a non-admin calls `POST /api/households/[id]/bills`
- **THEN** the API returns 403

### Requirement: Bill i18n Keys

All bill text MUST use i18n keys under the `bills` namespace. Spanish default. Keys: `bills.list.*`, `bills.create.*`, `bills.edit.*`, `bills.pay.*`, `bills.status.*` (PENDING, PAID, OVERDUE, DUE_SOON), `bills.empty.*`, `bills.errors.*` (cannotEditPaid, invalidBody).

#### Scenario: All bill text via keys
- **WHEN** any bill screen is rendered
- **THEN** every user-facing string is produced by a `t('bills.*')` call

### Requirement: Bill Audit Logging

The application MUST write `AuditLog` entries for bill mutations: creation, edit, mark paid, un-mark paid. `userId` = actor, `householdId` = household, `entity = 'Bill'`, `entityId` = billId, `metadata` includes the action and changed fields.

#### Scenario: Bill creation audited
- **WHEN** a bill is created
- **THEN** an `AuditLog` row is created with `action = 'billCreated'` and the bill id

#### Scenario: Mark paid audited
- **WHEN** a bill is marked paid
- **THEN** an `AuditLog` row is created with `action = 'billPaid'` and metadata including `paidByUserId`
