## Context

Spec 01 defined the `BankAccount` (householdMemberId, bankName, accountType, encryptedCredentials, lastSyncAt, lastSyncStatus), `SyncJob` (bankAccountId, status, startedAt nullable, completedAt nullable, errorMessage, retryCount), and `Notification` (type SYNC_FAILURE) models, plus the AES-256 encryption utility. Spec 03 provides household + member context. This spec adds bank account management and the sync pipeline.

The critical MVP decision (from user Q&A): sync is **mocked via JSON fixtures**. The `SyncProvider` interface is provider-ready so a real aggregator can replace the mock without changing consumers. The daily sync job runs (via cron or external scheduler hitting an API route), creates a SyncJob, calls the provider, and outputs raw transactions. Spec 05 is responsible for storing those as `Transaction` rows (billing-month derivation, FX conversion, categorization). This separation keeps the sync interface focused on data acquisition.

## Goals / Non-Goals

**Goals:**
- Members can connect bank accounts (bank name, account type, credentials encrypted).
- Define `SyncProvider` interface + `MockSyncProvider` (JSON fixtures).
- Daily sync job (configurable cron) + manual "sync now".
- SyncJob lifecycle: PENDING → RUNNING → SUCCESS/FAILED with retry (3 attempts, backoff).
- On persistent failure: create Notification(SYNC_FAILURE), update BankAccount.lastSyncStatus = FAILED.
- Bank account list + connect form with full UI states.
- All credentials encrypted at rest; never logged or returned in API responses.

**Non-Goals:**
- Real bank aggregator integration (interface-ready, mock only in MVP).
- Manual transaction entry or editing (global rule).
- Deleting bank accounts (disconnect only, retain for history).
- Real-time sync.
- Transaction storage (Spec 05 stores raw output as Transaction rows).

## Decisions

### D1: SyncProvider interface
**Decision:** Define a `SyncProvider` interface:
```
interface SyncProvider {
  fetchTransactions(credentials: DecryptedCredentials, providerAccountId: string, fromDate: Date, toDate: Date): Promise<RawTransaction[]>
}
interface RawTransaction {
  amount: number (signed)
  currency: string (ISO 4217)
  date: string (ISO date)
  description: string
  merchantName?: string
  providerTransactionId: string (idempotency key)
}
```
The `MockSyncProvider` implements this by reading JSON fixture files under `lib/sync/fixtures/<bankName>.json` and returning transactions within the date range. The interface is async and provider-agnostic so a real aggregator (Belvo/Plaid) can implement it later. `getAccounts` is intentionally omitted from the MVP interface (the connect flow stores credentials without validating them via the provider; account discovery is a future enhancement when a real provider is integrated).
**Rationale:** Clean separation: sync acquires raw data; Spec 05 stores + converts it. The `providerTransactionId` ensures idempotent inserts (no duplicate transactions on re-sync). Keeping the interface minimal avoids dead surface area.
**Alternatives considered:**
- Include `getAccounts` for credential validation on connect: adds complexity for a mock that has no real auth. Deferred to when a real provider is integrated.
- Couple sync directly to Transaction storage: violates separation of concerns, harder to test. Rejected.
- No interface (hardcode mock): cannot swap providers later. Rejected.

### D1a: Spec 05 handoff interface
**Decision:** The sync job hands raw transactions to Spec 05 via a typed function: `storeRawTransactions(bankAccountId: string, householdId: string, rawTransactions: RawTransaction[]): Promise<void>`. Spec 05 implements this function (billing-month derivation, FX conversion, idempotent insert by `providerTransactionId`, category assignment). Spec 04 defines the signature and calls it; if Spec 05 is not yet implemented, a stub that no-ops (or logs) is used so Spec 04 can be verified independently.
**Rationale:** A defined handoff boundary lets Spec 04 and Spec 05 be implemented and tested independently. The `providerTransactionId` in `RawTransaction` enables idempotent inserts in Spec 05.

### D2: Mock fixture format
**Decision:** JSON files at `lib/sync/fixtures/<bankName>.json` with shape:
```
{
  "bankName": "Banco de Chile",
  "accounts": [
    { "providerAccountId": "acc-1", "type": "CHECKING", "currency": "CLP" }
  ],
  "transactions": [
    { "providerTransactionId": "tx-001", "accountId": "acc-1", "amount": -15000, "currency": "CLP", "date": "2026-07-01", "description": "Supermercado", "merchantName": "Lider" }
  ]
}
```
The mock returns transactions filtered by date range. A seed fixture covers a few months of data for 2-3 sample banks so the app is demonstrable end-to-end.
**Rationale:** Simple, file-based, version-controllable, easy to extend with more banks.
**Alternatives considered:**
- Database-seeded mock data: couples mock to DB, harder to reset. Rejected.
- Inline hardcoded arrays: not extensible. Rejected.

### D3: Sync job scheduling
**Decision:** A daily sync is triggered by an external scheduler (cron, Vercel Cron, or systemd timer) hitting a protected `POST /api/sync/run` route with a shared secret (`SYNC_SECRET` env var). The route iterates all active BankAccounts and creates a SyncJob for each. Manual "sync now" hits `POST /api/households/[id]/bank-accounts/[accountId]/sync` (member-authenticated). Both paths reuse the same `runSyncJob(syncJobId)` core function.
**Route-level scale strategy:** Each SyncJob has a 5-minute timeout, but the `POST /api/sync/run` route itself iterates all active accounts in a single request. To stay within serverless function limits (Vercel default ~10-60s), the route MUST process accounts in batches with a configurable `SYNC_BATCH_SIZE` (default 10 accounts per invocation) and a `SYNC_ROUTE_TIMEOUT_MS` (default 50,000ms). If the route approaches the timeout before all accounts are processed, it MUST stop creating new jobs and return `{ data: { processed, remaining }, error: null }`; the external scheduler is expected to re-invoke the route (or the next daily tick) to process the remaining accounts. For MVP with the fast mock provider and a small number of accounts per household, a single invocation suffices; the batch/timeout strategy is in place for production scale. The `SYNC_BATCH_SIZE` and `SYNC_ROUTE_TIMEOUT_MS` env vars MUST be documented in `.env.example`.
**Rationale:** Keeps the app serverless-friendly (no long-running cron process inside Next.js). Shared secret prevents unauthorized sync triggers. Reuse avoids drift between manual and scheduled paths. The batch + route-timeout strategy prevents serverless function limits from breaking the sync at scale without requiring a queue infrastructure in MVP.
**Alternatives considered:**
- In-process setInterval: doesn't survive serverless cold starts. Rejected.
- Queue-based (BullMQ): the right production solution, but overkill for MVP single-instance. Deferred to when real providers + scale demand it.
- Synchronous iteration with no timeout: breaks on serverless at ~60s. Rejected.

### D4: Retry strategy
**Decision:** On sync failure, the SyncJob retries up to `SYNC_MAX_RETRIES` (default 3) with exponential backoff (`SYNC_RETRY_BACKOFF_MS` base, default 60s; 60s, 120s, 240s). Retries are handled by the daily `POST /api/sync/run` route: in addition to creating new jobs for active accounts, the route sweeps `FAILED` SyncJobs where `retryCount < SYNC_MAX_RETRIES` AND `completedAt + backoff < now()`, and re-invokes `runSyncJob` for them (incrementing `retryCount`). This avoids in-process `setTimeout` (infeasible in serverless Next.js). After max retries, `lastSyncStatus = FAILED`, a `Notification(SYNC_FAILURE)` is created (email dispatched in Spec 09), and an in-app banner shows on the bank account list.
**Rationale:** Exponential backoff is standard for transient failures. Sweeping on the daily route reuses the existing trigger and is serverless-compatible. Capping retries prevents infinite loops. Notification + banner ensures visibility.
**Alternatives considered:**
- In-process setTimeout retry: infeasible in serverless (function invocation ends). Rejected.
- Dedicated retry queue (BullMQ): overkill for MVP single-instance. Deferred.
- No retry: too fragile for a daily job. Rejected.
- Immediate retry (no backoff): hammers a failing provider. Rejected.

### D5: Credential handling in sync
**Decision:** The sync job decrypts credentials server-side using the Spec 01 encryption utility immediately before calling the provider, and never logs or persists the decrypted form. The decrypted credentials exist only in-memory for the duration of the provider call.
**Rationale:** Credentials must be usable for sync but never exposed in logs or responses.
**Alternatives considered:**
- Pass encrypted creds to provider: provider can't use them. Rejected.

### D6: Disconnect vs delete
**Decision:** MVP does not delete BankAccount rows. "Disconnecting" a bank account sets a `disabledAt` timestamp (nullable field — note: this requires adding `disabledAt` to BankAccount, or repurposing an existing field; see Risks). Disabled accounts are excluded from the daily sync but retained for historical transactions. The spec requires this field.
**Rationale:** Deleting bank accounts would orphan transactions and break historical reports. Disconnect preserves history.
**Alternatives considered:**
- Hard delete: orphans transactions. Rejected.
- No disconnect (once connected, always syncing): user can't stop a broken account from retrying. Rejected.

## Risks / Trade-offs

- [Mock fixtures diverge from real provider shape] → Mitigation: the `RawTransaction` interface is the contract; mock fixtures conform to it. When a real provider is integrated, only its adapter changes, not consumers.
- [External scheduler availability] → Mitigation: Vercel Cron (if deployed on Vercel) or a simple cron job. Documented as a deployment requirement. Manual sync is always available as a fallback.
- [`disabledAt` field not in Spec 01 schema] → Mitigation: this spec requires adding a nullable `disabledAt` field to `BankAccount`. This is a schema addition; coordinate by noting it in the proposal Impact section. Spec 01 defined `lastSyncStatus` with a PENDING value which can approximate, but a dedicated `disabledAt` is cleaner for the disconnect feature.
- [Sync job timeout] → Mitigation: each SyncJob should have a reasonable timeout (e.g. 5 min); if exceeded, mark FAILED and retry. Document in the sync requirement.
- [Shared secret leakage] → Mitigation: `SYNC_SECRET` is server-only, never in client bundle (per Spec 01 secret safety).
