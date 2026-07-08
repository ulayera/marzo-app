## Why

Households exist (Spec 03) but have no financial data. Bank accounts are the source of all transactions (income/expenses). This spec adds the ability to connect bank accounts with encrypted credentials and defines the sync interface + daily job that pulls transactions. In MVP, the sync is mocked via JSON fixtures so the app is testable end-to-end without a real bank aggregator; the interface is provider-ready so a real provider (Belvo, Plaid) can be plugged in later.

## What Changes

- Add bank account management: a member can connect a bank account (bank name, account type, credentials encrypted via Spec 01 AES-256 utility).
- Define a provider-ready `SyncProvider` interface (`getAccounts`, `fetchTransactions`) with a `MockSyncProvider` implementation that reads from JSON fixtures.
- Add a daily sync job that creates a `SyncJob`, calls the provider, stores resulting transactions (transaction storage is Spec 05's concern; this spec defines the job + interface + raw transaction output), updates `BankAccount.lastSyncAt/lastSyncStatus`.
- Add sync failure handling: retry with backoff (3 attempts), email admin + in-app banner on persistent failure, create `Notification(SYNC_FAILURE)`.
- Add manual "sync now" button per bank account (triggers a SyncJob on demand).
- Add bank account list UI and connect-account form with empty/loading/error states.
- All UI in `(app)` route group, i18n, theming, Spec 01 components.

## Capabilities

### New Capabilities
- `bank-accounts-sync`: Bank account CRUD, provider-ready sync interface (mock JSON in MVP), daily sync job, manual sync, failure retry + notification.

### Modified Capabilities
- `system-design`: CONSUMES the `BankAccount` model's `disabledAt` field (nullable timestamp, defined canonically in Spec 01) for the disconnect feature. Consumes the encryption utility, `SyncJob`, and `Notification` models from Spec 01. No additional schema changes.

## Impact

- **Code**: `app/(app)/bank-accounts` routes; `/api/households/[id]/bank-accounts/*` API routes; `lib/sync/SyncProvider` interface + `MockSyncProvider`; daily sync job (cron or external scheduler invoking an API route); JSON fixtures under `lib/sync/fixtures/`.
- **Database**: No additional schema changes — `disabledAt` (nullable timestamp on `BankAccount`) is defined canonically in Spec 01. This spec consumes it for the disconnect feature.
- **Dependencies**: None new (no real bank SDK in MVP).
- **Environment variables**: `SYNC_SCHEDULE_CRON` (default daily), `SYNC_MAX_RETRIES` (default 3), `SYNC_RETRY_BACKOFF_MS` (default 60000).
- **Future specs**: Spec 05 (transactions) consumes the raw transaction output from the sync job and stores `Transaction` rows with billing-month + FX conversion. Spec 09 (notifications) consumes `SYNC_FAILURE` Notification rows.

## Non-goals

- Real bank aggregator integration (Belvo/Plaid) — the interface is provider-ready but only the mock is implemented in MVP.
- Manual transaction entry — transactions come from sync only (per global rules).
- Editing or deleting synced transactions — synced data is immutable in MVP (corrections are a future spec).
- Deleting bank accounts — MVP supports disconnecting (disabling sync) but retains the account record for historical data.
- Real-time sync — daily + manual on-demand only.
- Multi-currency per account — the account's currency is inferred from the synced transactions; the account itself has no currency field in MVP.
