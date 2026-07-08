## MODIFIED Requirements

### Requirement: Master Database Schema — Notification

The `Notification` model is defined canonically in Spec 01 (system-design), which includes the `retryCount` (int, default 0) field for tracking email send retries. This spec CONSUMES that field in the notification processor. The notification processor MUST increment `retryCount` on each failed send attempt, and after `retryCount >= NOTIFICATION_MAX_RETRIES` the `emailStatus` MUST be set to `FAILED`. No additional schema changes are made by this spec. See Spec 01 for the canonical field list and model definition.

#### Scenario: Retry count used by processor
- **WHEN** the notification processor retries a failed send
- **THEN** the `retryCount` field (defined in Spec 01) is incremented, and after `NOTIFICATION_MAX_RETRIES` the `emailStatus` becomes `FAILED`

## ADDED Requirements

### Requirement: Notification Processor

The application MUST implement a notification processor that reads `Notification` rows with `emailStatus = PENDING`, resolves the recipient email via `Notification.userId` → `User.email`, renders the appropriate i18n email template per `Notification.type`, sends the email via the Spec 01 `sendEmail` function, and updates `emailStatus` to `SENT` (with `sentAt = now()`) on success or increments `retryCount` on failure. After `retryCount >= NOTIFICATION_MAX_RETRIES` (default 3), `emailStatus` MUST be set to `FAILED`. The processor MUST be triggered by `POST /api/notifications/process` with a `NOTIFICATIONS_SECRET` header (constant-time comparison); requests without a valid secret MUST return 401.

#### Scenario: Process pending notification
- **WHEN** the processor runs and a PENDING notification exists
- **THEN** the email is rendered from the template, sent via the provider, and `emailStatus` becomes `SENT` with `sentAt` set

#### Scenario: Send failure increments retry
- **WHEN** `sendEmail` fails for a PENDING notification and `retryCount < NOTIFICATION_MAX_RETRIES`
- **THEN** `retryCount` is incremented and `emailStatus` remains PENDING for the next processor run

#### Scenario: Max retries exhausted
- **WHEN** `sendEmail` fails and `retryCount >= NOTIFICATION_MAX_RETRIES`
- **THEN** `emailStatus` becomes `FAILED`

#### Scenario: Secret required
- **WHEN** `POST /api/notifications/process` is called without a valid `NOTIFICATIONS_SECRET`
- **THEN** the response is 401 and no notifications are processed

#### Scenario: Recipient email resolved
- **WHEN** the processor processes a notification
- **THEN** the email is sent to the `User.email` of the `Notification.userId`

### Requirement: BILL_DUE Trigger

The application MUST run a daily check for bills due within `BILL_DUE_REMINDER_DAYS` (default 3) of today. For each PENDING bill with `dueDate` within the window, the system MUST create a `Notification(BILL_DUE)` row for each ACTIVE household member with `userId` = member, `householdId`, `emailStatus = PENDING`, and `payload = { billId, billTitle, amount, dueDate, householdId }`. The check MUST be idempotent: do NOT create a duplicate BILL_DUE notification for the same `(billId, userId)` if one already exists with `emailStatus` PENDING or SENT. The trigger MUST be invoked via `POST /api/notifications/bill-due-check` with a `NOTIFICATIONS_SECRET` header (or combined into the process route).

#### Scenario: Bill due soon creates notification
- **WHEN** the daily check runs and a PENDING bill has a due date within 3 days
- **THEN** a `Notification(BILL_DUE)` row is created for each ACTIVE household member with the documented payload

#### Scenario: No duplicate BILL_DUE
- **WHEN** the daily check runs and a BILL_DUE notification already exists for a `(billId, userId)` with PENDING or SENT status
- **THEN** no duplicate notification is created

#### Scenario: Paid bills skipped
- **WHEN** the daily check runs and a bill is already PAID
- **THEN** no BILL_DUE notification is created for it

#### Scenario: Secret required for bill-due check
- **WHEN** `POST /api/notifications/bill-due-check` is called without a valid `NOTIFICATIONS_SECRET`
- **THEN** the response is 401

### Requirement: Email Templates

The application MUST provide HTML + text email templates for each notification type under `i18n/locales/es/emails/`:
- `memberAdded` — variables: `{{householdName}}`, `{{addedByName}}`
- `syncFailure` — variables: `{{bankName}}`, `{{errorMessage}}`
- `settlementReady` — variables: `{{billingMonth}}`, `{{householdName}}`
- `billDue` — variables: `{{billTitle}}`, `{{amount}}`, `{{dueDate}}`
All template text (subject and body) MUST use i18n keys with `{{var}}` interpolation (per Spec 01 email utility). No hard-coded user-facing text in templates. Spanish is the default locale.

#### Scenario: Member-added template
- **WHEN** a MEMBER_ADDED notification is processed
- **THEN** the `memberAdded` template is rendered with `{{householdName}}` and `{{addedByName}}` interpolated from the notification payload

#### Scenario: Sync-failure template
- **WHEN** a SYNC_FAILURE notification is processed
- **THEN** the `syncFailure` template is rendered with `{{bankName}}` and `{{errorMessage}}` interpolated

#### Scenario: Settlement-ready template
- **WHEN** a SETTLEMENT_READY notification is processed
- **THEN** the `settlementReady` template is rendered with `{{billingMonth}}` and `{{householdName}}`

#### Scenario: Bill-due template
- **WHEN** a BILL_DUE notification is processed
- **THEN** the `billDue` template is rendered with `{{billTitle}}`, `{{amount}}`, and `{{dueDate}}`

#### Scenario: No hard-coded text in templates
- **WHEN** an email template file is inspected
- **THEN** all user-facing text comes from i18n locale keys, not inline literals

### Requirement: Notification Environment Variables

The `.env.example` MUST list `NOTIFICATIONS_SECRET`, `NOTIFICATION_MAX_RETRIES`, `BILL_DUE_REMINDER_DAYS`, and `NOTIFICATIONS_SCHEDULE_CRON` with placeholder values. The application MUST emit a clear startup error if `NOTIFICATIONS_SECRET` is missing (consistent with Spec 01's env guard).

#### Scenario: Env example lists notification keys
- **WHEN** the `.env.example` is inspected
- **THEN** it includes `NOTIFICATIONS_SECRET`, `NOTIFICATION_MAX_RETRIES`, `BILL_DUE_REMINDER_DAYS`, `NOTIFICATIONS_SCHEDULE_CRON` with placeholders

#### Scenario: Missing secret surfaced
- **WHEN** the application starts without `NOTIFICATIONS_SECRET`
- **THEN** a clear startup error names the missing variable

### Requirement: Notification Audit Logging

The application MUST write `AuditLog` entries for notification processing outcomes: SENT and FAILED. `userId` = null (system action), `householdId` = notification's householdId (if set), `entity = 'Notification'`, `entityId` = notification id, `metadata` includes the notification type and outcome.

#### Scenario: Sent notification audited
- **WHEN** a notification is successfully sent
- **THEN** an `AuditLog` row is created with `action = 'notificationSent'` and the notification id

#### Scenario: Failed notification audited
- **WHEN** a notification is marked FAILED after max retries
- **THEN** an `AuditLog` row is created with `action = 'notificationFailed'` and metadata including the failure reason
