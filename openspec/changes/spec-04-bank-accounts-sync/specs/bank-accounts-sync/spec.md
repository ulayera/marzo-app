## ADDED Requirements

### Requirement: Bank Account Connection

A household member SHALL be able to connect a bank account by providing a `bankName`, `accountType` (CHECKING | SAVINGS | CREDIT), and credentials. The credentials MUST be encrypted using the Spec 01 AES-256 encryption utility before storage and MUST NEVER be stored in plaintext. The `encryptedCredentials` column MUST contain only ciphertext. The bank account MUST be linked to the connecting member's `HouseholdMember` row. Any `ACTIVE` household member (not just admin) can connect a bank account to their own membership.

#### Scenario: Connect a bank account
- **WHEN** a member submits a bank name, account type, and credentials
- **THEN** a `BankAccount` row is created with `householdMemberId` set to the member, `encryptedCredentials` containing AES-256 ciphertext, `lastSyncStatus = PENDING`, and the credentials are never logged

#### Scenario: Credentials encrypted at rest
- **WHEN** the `BankAccount` row is inspected
- **THEN** `encryptedCredentials` contains ciphertext and the plaintext credentials are not present in the database or logs

#### Scenario: Credentials never in API response
- **WHEN** any bank-account API response is inspected
- **THEN** no `encryptedCredentials` or plaintext credentials field is present

### Requirement: Bank Account List View

Any `ACTIVE` household member SHALL be able to view all bank accounts connected by any member of the household (not just their own). The list MUST show: bank name, account type, owner member name, last sync timestamp, last sync status badge (SUCCESS | FAILED | PENDING | NEVER). The list MUST use the Spec 01 DataTable (dense rows, loading skeleton, EmptyState when no accounts connected). Each row MUST offer a "Sync now" action (for any member) and a "Disconnect" action (admin only, or the account owner).

#### Scenario: View all household bank accounts
- **WHEN** any `ACTIVE` member opens the bank accounts page
- **THEN** all bank accounts connected by any household member are listed with bank name, type, owner, last sync time, and status badge

#### Scenario: Empty state when no accounts
- **WHEN** the household has no connected bank accounts
- **THEN** an `EmptyState` is shown prompting the user to connect their first account

#### Scenario: Loading skeleton
- **WHEN** the bank account list is loading
- **THEN** skeleton rows are shown (not empty content)

#### Scenario: List load error
- **WHEN** the bank account list fails to load
- **THEN** an `ErrorState` with a retry action is shown in place of the list

### Requirement: Bank Account Disconnect

An admin or the account owner SHALL be able to disconnect a bank account. Disconnection MUST set `disabledAt` (nullable timestamp on `BankAccount`) to the current time. A disabled bank account MUST be excluded from the daily sync job. The bank account row and its historical transactions MUST be retained (not deleted). A disabled account MUST still appear in the list with a `DISABLED` badge. A disconnected account SHALL NOT be re-enabled in MVP (re-connecting requires creating a new BankAccount row).

#### Scenario: Disconnect a bank account
- **WHEN** an admin or the account owner disconnects a bank account
- **THEN** `disabledAt` is set to now, the account is excluded from future syncs, and it appears with a `DISABLED` badge

#### Scenario: Disabled account excluded from sync
- **WHEN** the daily sync job runs
- **THEN** bank accounts with a non-null `disabledAt` are skipped

#### Scenario: Non-owner non-admin cannot disconnect
- **WHEN** a member who is neither the account owner nor an admin attempts to disconnect another member's bank account
- **THEN** the API returns 403

### Requirement: SyncProvider Interface

The application MUST define a `SyncProvider` interface with `fetchTransactions(credentials, providerAccountId, fromDate, toDate) → Promise<RawTransaction[]>`. A `RawTransaction` MUST include: `amount` (signed number), `currency` (ISO 4217), `date` (ISO date string), `description`, optional `merchantName`, and `providerTransactionId` (unique idempotency key). The interface MUST be async and provider-agnostic so a real aggregator (Belvo, Plaid) can implement it without changing consumers. The system MUST decrypt credentials server-side (via Spec 01 encryption utility) immediately before calling the provider and MUST NOT log or persist the decrypted form. The sync job MUST hand raw transactions to the transaction-storage service via `storeRawTransactions(bankAccountId, householdId, rawTransactions) → Promise<void>` (implemented in Spec 05; a no-op stub is used until Spec 05 is implemented).

#### Scenario: Provider interface is async and typed
- **WHEN** the `SyncProvider` interface is inspected
- **THEN** it declares `getAccounts` and `fetchTransactions` returning Promises with the documented RawTransaction shape

#### Scenario: Decrypted credentials not logged
- **WHEN** the sync job calls the provider
- **THEN** decrypted credentials exist only in-memory for the provider call and never appear in logs

### Requirement: Mock Sync Provider

The application MUST implement a `MockSyncProvider` that reads from JSON fixture files under `lib/sync/fixtures/<bankName>.json`. Each fixture MUST contain `bankName`, `accounts` (with `providerAccountId`, `type`, `currency`), and `transactions` (with `providerTransactionId`, `accountId`, `amount`, `currency`, `date`, `description`, optional `merchantName`). The mock MUST return transactions filtered by the requested date range and account. At least 2-3 sample bank fixtures MUST be provided covering a few months of data so the app is demonstrable end-to-end. The `MockSyncProvider` MUST be the default provider, swappable via env (`SYNC_PROVIDER = mock` default).

When `SYNC_PROVIDER = mock`, the connect-bank-account form's `bankName` input MUST offer a dropdown of known fixture bank names (derived from the fixture files present in `lib/sync/fixtures/`), with a free-text fallback for other names. If the mock provider is asked for a `bankName` with no matching fixture file, it MUST return an empty transaction list (not an error), and the bank account's first sync MUST result in `lastSyncStatus = SUCCESS` with zero transactions; the UI MUST surface a subtle "no data found for this bank — check the bank name matches a fixture" note on the account row so the user understands the empty result is a name mismatch, not a sync failure.

#### Scenario: Mock returns transactions in date range
- **WHEN** `MockSyncProvider.fetchTransactions` is called with a date range
- **THEN** it returns only transactions whose `date` falls within the range, each with a `providerTransactionId`

#### Scenario: Mock provider is swappable
- **WHEN** the `SYNC_PROVIDER` env variable is set to a different value
- **THEN** the app resolves the corresponding provider implementation without changing consumers

#### Scenario: Sample fixtures present
- **WHEN** the `lib/sync/fixtures/` directory is inspected
- **THEN** at least 2-3 JSON fixture files exist covering multiple months of transactions

#### Scenario: Connect form offers known fixture bank names
- **WHEN** `SYNC_PROVIDER = mock` and a member opens the connect-bank-account form
- **THEN** the `bankName` input offers a dropdown of bank names derived from the fixture files in `lib/sync/fixtures/`, with a free-text fallback

#### Scenario: Unknown bank name returns empty with a note
- **WHEN** a bank account is connected with a `bankName` that has no matching fixture file and a sync runs
- **THEN** the sync completes with `lastSyncStatus = SUCCESS` and zero transactions, and the account row shows a "no data found for this bank — check the bank name matches a fixture" note

### Requirement: Daily Sync Job

The application MUST run a daily sync job triggered by an external scheduler hitting `POST /api/sync/run` with a `SYNC_SECRET` header (server-only env var, compared with constant-time comparison). The route MUST iterate all `BankAccount` rows where `disabledAt` is null AND the owning `HouseholdMember.status` is `ACTIVE` (skipping accounts owned by removed members), and create a `SyncJob(PENDING)` for each. The route MUST also sweep `FAILED` SyncJobs where `retryCount < SYNC_MAX_RETRIES` AND `completedAt + backoff < now()`, re-invoking `runSyncJob` for them (incrementing `retryCount`). The sync MUST decrypt credentials, call the configured `SyncProvider.fetchTransactions` for the date range since `lastSyncAt` (or the fixture's earliest date on first sync), and hand the raw transactions to `storeRawTransactions` (Spec 05). On success, the SyncJob status becomes `SUCCESS`, `startedAt` and `completedAt` are set, and `BankAccount.lastSyncAt` + `lastSyncStatus = SUCCESS` are updated. The route MUST reject requests without a valid `SYNC_SECRET` with 401. Each SyncJob MUST have a timeout (default 5 minutes; note: serverless platforms may require a longer-running worker or lower default — document as a deployment requirement); if exceeded, the job is marked FAILED. The route MUST process accounts in batches of `SYNC_BATCH_SIZE` (default 10) and stop creating new jobs if the route approaches `SYNC_ROUTE_TIMEOUT_MS` (default 50,000ms), returning `{ data: { processed, remaining }, error: null }` so the scheduler can re-invoke for remaining accounts.

#### Scenario: Daily sync creates jobs for all active accounts
- **WHEN** `POST /api/sync/run` is called with a valid `SYNC_SECRET`
- **THEN** a `SyncJob(PENDING)` is created for every BankAccount with null `disabledAt`

#### Scenario: Sync secret required
- **WHEN** `POST /api/sync/run` is called without a valid `SYNC_SECRET`
- **THEN** the response is 401 and no jobs are created

#### Scenario: Successful sync updates account
- **WHEN** a SyncJob completes successfully
- **THEN** `status = SUCCESS`, `startedAt` and `completedAt` are set, `BankAccount.lastSyncAt` is updated, and `lastSyncStatus = SUCCESS`

#### Scenario: Disabled accounts skipped
- **WHEN** the daily sync runs
- **THEN** bank accounts with a non-null `disabledAt` are not synced

#### Scenario: Removed-member accounts skipped
- **WHEN** the daily sync runs
- **THEN** bank accounts whose owning `HouseholdMember.status` is `REMOVED` are not synced

#### Scenario: Failed jobs swept for retry
- **WHEN** the daily sync route runs AND a previously-failed SyncJob has `retryCount < SYNC_MAX_RETRIES` AND `completedAt + backoff < now()`
- **THEN** the job is re-run via `runSyncJob` and `retryCount` is incremented

#### Scenario: Sync timeout marks failure
- **WHEN** a SyncJob exceeds the 5-minute timeout
- **THEN** it is marked FAILED and becomes eligible for retry

#### Scenario: Route stops before serverless timeout
- **WHEN** the daily sync route approaches `SYNC_ROUTE_TIMEOUT_MS` before all active accounts are processed
- **THEN** it stops creating new jobs and returns `{ data: { processed, remaining }, error: null }` so the scheduler can re-invoke for the remaining accounts

### Requirement: Manual Sync Now

Any `ACTIVE` household member SHALL be able to trigger a manual sync for a specific bank account via `POST /api/households/[id]/bank-accounts/[accountId]/sync`. This MUST create a `SyncJob(PENDING)` and run the same `runSyncJob` core function as the daily job. The route MUST verify the caller is an ACTIVE member of the household. The UI MUST show a loading state on the "Sync now" button and update the status badge on completion. A sync already in progress (RUNNING/PENDING for the same account within the last 5 minutes) MUST return 409 with `bank.errors.syncInProgress`.

#### Scenario: Manual sync triggers job
- **WHEN** a member clicks "Sync now" on a bank account
- **THEN** a SyncJob is created and run, the button shows a loading state, and the status badge updates on completion

#### Scenario: Sync already in progress
- **WHEN** a member triggers a manual sync while a SyncJob for the same account is RUNNING or was PENDING within the last 5 minutes
- **THEN** the API returns 409 with `error.code = 'syncInProgress'`

#### Scenario: Non-member cannot trigger sync
- **WHEN** a user who is not an ACTIVE member of the household triggers a sync
- **THEN** the API returns 403

### Requirement: Sync Failure Handling and Retry

On sync failure, the SyncJob MUST be marked `FAILED` with `errorMessage` set and `completedAt` set. The system MUST retry up to `SYNC_MAX_RETRIES` (default 3) with exponential backoff (`SYNC_RETRY_BACKOFF_MS` base, default 60s). After exhausting retries, `BankAccount.lastSyncStatus` MUST be set to `FAILED` and a `Notification(SYNC_FAILURE)` row MUST be created for each ADMIN of the household (`userId` per admin, `householdId` set to the household, `emailStatus = PENDING`, and `payload = { bankAccountId, bankName, errorMessage, failedAt }`). An in-app banner MUST appear on the bank accounts page for accounts with `lastSyncStatus = FAILED`. Email dispatch is deferred to Spec 09.

#### Scenario: Failed sync records error
- **WHEN** a SyncJob fails
- **THEN** `status = FAILED`, `completedAt` is set, and `errorMessage` stores the failure reason

#### Scenario: Retry with backoff
- **WHEN** a SyncJob fails and `retryCount < SYNC_MAX_RETRIES`
- **THEN** the job is retried after exponential backoff and `retryCount` is incremented

#### Scenario: Max retries exhausted creates notification
- **WHEN** a SyncJob fails and `retryCount >= SYNC_MAX_RETRIES`
- **THEN** `BankAccount.lastSyncStatus = FAILED`, a `Notification(SYNC_FAILURE)` row is created for each ADMIN of the household with the documented payload, and `emailStatus = PENDING`

#### Scenario: In-app banner for failed accounts
- **WHEN** the bank accounts page is viewed and an account has `lastSyncStatus = FAILED`
- **THEN** an in-app banner is displayed indicating the account failed to sync with a retry action

### Requirement: Bank Account API Routes

The application SHALL expose REST-like bank account API routes, all returning the Spec 01 `{ data, error }` envelope and enforcing auth + membership:

- `GET /api/households/[id]/bank-accounts` — list household bank accounts (any ACTIVE member)
- `POST /api/households/[id]/bank-accounts` — connect a bank account (any ACTIVE member; body: `{ bankName, accountType, credentials }`)
- `POST /api/households/[id]/bank-accounts/[accountId]/sync` — manual sync (any ACTIVE member)
- `PATCH /api/households/[id]/bank-accounts/[accountId]` — disconnect (admin or owner; body: `{ disabled: true }`)
- `POST /api/sync/run` — daily sync trigger (SYNC_SECRET required, not member-authenticated)

All member-facing routes MUST verify ACTIVE membership. No route MAY return `encryptedCredentials` or plaintext credentials.

#### Scenario: List bank accounts
- **WHEN** `GET /api/households/[id]/bank-accounts` is called by an ACTIVE member
- **THEN** the response is `{ data: [ { bankAccountId, bankName, accountType, ownerMemberId, ownerName, lastSyncAt, lastSyncStatus, disabledAt } ], error: null }` with no credentials field

#### Scenario: Connect bank account route
- **WHEN** `POST /api/households/[id]/bank-accounts` is called with a valid body
- **THEN** the response is `{ data: { bankAccountId }, error: null }` with 201 and the credentials are encrypted before storage

#### Scenario: Invalid connect body rejected
- **WHEN** `POST /api/households/[id]/bank-accounts` is called with missing `bankName`, `accountType`, or `credentials`
- **THEN** the response is 400 with `error.code = 'invalidBody'` and no `BankAccount` is created

#### Scenario: No credentials in any response
- **WHEN** any bank-account API response body is inspected
- **THEN** no `encryptedCredentials` or `credentials` field is present

### Requirement: Bank Account UI States

The bank accounts page and connect form MUST define empty, loading, error, and success states. The connect form MUST show inline validation (required fields), a disabled+spinner submit during creation, and an error state on failure. The sync-now button MUST show loading and update the status badge on completion. The disconnect action MUST confirm before proceeding. A failed-sync banner MUST appear for accounts with `lastSyncStatus = FAILED` with a retry action.

#### Scenario: Connect form loading
- **WHEN** a member submits the connect form
- **THEN** the submit button is disabled and shows a spinner until the response arrives

#### Scenario: Connect form validation error
- **WHEN** a member submits with missing bank name or credentials
- **THEN** inline i18n-keyed errors appear and the API is not called

#### Scenario: Connect form server error
- **WHEN** the connect API call fails (server error)
- **THEN** an `ErrorState` or inline error is shown with a retry action and the form retains the user's input

#### Scenario: Sync button loading
- **WHEN** a member clicks "Sync now"
- **THEN** the button shows a loading state until the sync completes and the status badge updates

#### Scenario: Disconnect confirmation
- **WHEN** a member initiates a disconnect
- **THEN** a confirmation dialog appears before the disconnect is executed

### Requirement: Bank Account i18n Keys

All bank-account user-facing text MUST be rendered via i18n keys under the `bankAccounts` namespace. Spanish default. Keys MUST cover: `bankAccounts.list.*`, `bankAccounts.connect.*`, `bankAccounts.sync.*`, `bankAccounts.disconnect.*`, `bankAccounts.errors.*` (including `syncInProgress`, `syncFailed`, `requiredFields`), and `bankAccounts.status.*` (SUCCESS, FAILED, PENDING, NEVER, DISABLED).

#### Scenario: All bank text via keys
- **WHEN** any bank-account screen is rendered
- **THEN** every user-facing string is produced by a `t('bankAccounts.*')` call

### Requirement: Bank Account Audit Logging

The application MUST write `AuditLog` entries for bank account mutations: connection, disconnect, manual sync trigger. `userId` = actor, `householdId` = household, `entity = 'BankAccount'`, `entityId` = bankAccountId, `metadata` includes the action and relevant fields (never credentials).

#### Scenario: Connection audited
- **WHEN** a bank account is connected
- **THEN** an `AuditLog` row is created with `action = 'bankAccountConnected'` and the bank account id

#### Scenario: Audit never contains credentials
- **WHEN** a bank-account audit entry is inspected
- **THEN** `metadata` does not contain credentials or `encryptedCredentials`
