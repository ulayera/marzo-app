## 1. i18n Keys

- [ ] 1.1 Add the `bankAccounts` namespace JSON under `i18n/locales/es/bankAccounts.json` with keys for `list.*`, `connect.*`, `sync.*`, `disconnect.*`, `errors.*` (syncInProgress, syncFailed, requiredFields), and `status.*` (SUCCESS, FAILED, PENDING, NEVER, DISABLED)
- [ ] 1.2 Audit all bank-account screens to confirm no hard-coded user-facing text

## 2. Schema Verification

- [ ] 2.1 Verify the `BankAccount` model (from Spec 01) includes the nullable `disabledAt` field for the disconnect feature; no additional migration is needed from this spec

## 3. SyncProvider Interface & Mock

- [ ] 3.1 Define the `SyncProvider` interface (`fetchTransactions`) and `RawTransaction` type (amount, currency, date, description, merchantName?, providerTransactionId) in `lib/sync/types.ts`. Also define the `storeRawTransactions(bankAccountId, householdId, rawTransactions)` handoff signature (Spec 05 implements; stub no-ops until then).
- [ ] 3.2 Implement `MockSyncProvider` reading from `lib/sync/fixtures/<bankName>.json`, filtering transactions by date range and account
- [ ] 3.3 Create 2-3 sample JSON fixture files covering multiple months of transactions for sample banks
- [ ] 3.4 Implement a provider resolver that selects the provider based on `SYNC_PROVIDER` env (default `mock`) and is swappable without changing consumers

## 4. Bank Account API Routes

- [ ] 4.1 Implement `GET /api/households/[id]/bank-accounts` (list all household bank accounts with owner name, last sync, status; no credentials; any ACTIVE member)
- [ ] 4.2 Implement `POST /api/households/[id]/bank-accounts` (encrypt credentials via Spec 01 utility, create BankAccount with householdMemberId, return 201 with bankAccountId; any ACTIVE member)
- [ ] 4.3 Implement `PATCH /api/households/[id]/bank-accounts/[accountId]` (disconnect: set disabledAt; admin or owner only; 403 for others)
- [ ] 4.4 Verify no bank-account API response ever includes `encryptedCredentials` or plaintext credentials

## 5. Sync Core & Jobs

- [ ] 5.1 Implement `runSyncJob(syncJobId)` core function: set status RUNNING + startedAt, decrypt credentials, call provider.fetchTransactions since lastSyncAt (or fixture earliest date), hand raw transactions to `storeRawTransactions` (Spec 05; no-op stub until then), set status SUCCESS + completedAt + update BankAccount.lastSyncAt/lastSyncStatus
- [ ] 5.2 Implement `POST /api/sync/run` (require `SYNC_SECRET` header with constant-time comparison; iterate active BankAccounts where disabledAt is null AND owning HouseholdMember.status is ACTIVE in batches of `SYNC_BATCH_SIZE` (default 10); stop and return `{ data: { processed, remaining }, error: null }` if approaching `SYNC_ROUTE_TIMEOUT_MS` (default 50,000ms); also sweep FAILED SyncJobs with retryCount < SYNC_MAX_RETRIES whose completedAt + backoff < now() and re-run them; 401 on missing/invalid secret)
- [ ] 5.3 Implement `POST /api/households/[id]/bank-accounts/[accountId]/sync` (manual sync; verify ACTIVE membership; create SyncJob + runSyncJob; 409 syncInProgress if a job is RUNNING/PENDING within last 5 min)
- [ ] 5.4 Implement the 5-minute SyncJob timeout (mark FAILED if exceeded)

## 6. Retry & Failure Handling

- [ ] 6.1 Implement retry logic: on FAILED with `retryCount < SYNC_MAX_RETRIES`, schedule retry with exponential backoff (base `SYNC_RETRY_BACKOFF_MS`); increment retryCount
- [ ] 6.2 On exhausting retries: set `BankAccount.lastSyncStatus = FAILED` and create a `Notification(SYNC_FAILURE)` row for each ADMIN of the household with userId=admin, householdId, emailStatus=PENDING, payload={bankAccountId, bankName, errorMessage, failedAt}
- [ ] 6.3 Add `SYNC_SCHEDULE_CRON`, `SYNC_SECRET`, `SYNC_MAX_RETRIES`, `SYNC_RETRY_BACKOFF_MS`, `SYNC_BATCH_SIZE`, `SYNC_ROUTE_TIMEOUT_MS` to `.env.example`

## 7. Bank Accounts UI

- [ ] 7.1 Build the bank accounts list page at `app/(app)/bank-accounts` using Spec 01 DataTable (dense rows, status Badge, loading skeleton, EmptyState when no accounts)
- [ ] 7.2 Build the connect-bank-account form (bankName — dropdown of known fixture bank names when SYNC_PROVIDER=mock, with free-text fallback; accountType select, credentials) with inline validation, loading state, and error state
- [ ] 7.3 Add the "Sync now" button per row with loading state and status badge update on completion
- [ ] 7.4 Add the "Disconnect" action (admin or owner only) with a confirmation Modal before executing
- [ ] 7.5 Add the failed-sync in-app banner for accounts with `lastSyncStatus = FAILED` with a retry action
- [ ] 7.6 Show a DISABLED badge for disconnected accounts

## 8. Audit Logging

- [ ] 8.1 Write `AuditLog` entries for bank account connection, disconnect, and manual sync trigger (userId=actor, householdId, entity=BankAccount, metadata never contains credentials)
- [ ] 8.2 Verify audit entries never contain credentials or encryptedCredentials

## 9. Verification & Validation

- [ ] 9.1 Verify a member can connect a bank account and credentials are encrypted (inspect DB for ciphertext, no plaintext)
- [ ] 9.2 Verify no API response ever includes credentials
- [ ] 9.3 Verify the MockSyncProvider returns transactions in the requested date range with providerTransactionIds
- [ ] 9.4 Verify `POST /api/sync/run` without SYNC_SECRET returns 401 and creates no jobs
- [ ] 9.5 Verify the daily sync creates SyncJobs for all active (non-disabled) accounts and skips disabled ones
- [ ] 9.6 Verify manual "Sync now" creates and runs a job, updates lastSyncAt/lastSyncStatus, and shows loading + badge update in the UI
- [ ] 9.7 Verify a second manual sync within 5 minutes returns 409 syncInProgress
- [ ] 9.8 Verify sync failure marks the job FAILED, retries with backoff, and after max retries creates a Notification(SYNC_FAILURE) and sets lastSyncStatus=FAILED
- [ ] 9.9 Verify the failed-sync banner appears for FAILED accounts with a retry action
- [ ] 9.10 Verify disconnect sets disabledAt, excludes the account from sync, shows a DISABLED badge, and retains the row + history
- [ ] 9.11 Verify non-owner non-admin gets 403 on disconnect
- [ ] 9.12 Verify all bank-account text uses i18n keys (Spanish default)
- [ ] 9.13 Run `openspec validate spec-04-bank-accounts-sync` and resolve any reported issues
- [ ] 9.14 Verify the connect form offers a dropdown of known fixture bank names when SYNC_PROVIDER=mock
- [ ] 9.15 Verify an unknown bank name syncs with SUCCESS + zero transactions and shows a "no data found" note
- [ ] 9.16 Verify the daily sync route stops before `SYNC_ROUTE_TIMEOUT_MS` and returns `{ processed, remaining }`
