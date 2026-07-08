## Why

Specs 03, 04, 07, and 08 create `Notification` rows (MEMBER_ADDED, SYNC_FAILURE, SETTLEMENT_READY, BILL_DUE) with `emailStatus = PENDING`, but no emails are sent. This spec wires the email dispatch: a notification processor that reads PENDING notifications, renders i18n email templates (Spec 01 email utility), sends them via the configured provider, and updates `emailStatus` to SENT or FAILED. It also defines the BILL_DUE trigger (the only notification not yet created by a prior spec — bills with due dates within N days generate BILL_DUE notifications).

## What Changes

- Add a notification processor: reads PENDING `Notification` rows, renders the appropriate email template per type, sends via the Spec 01 `sendEmail` function, updates `emailStatus` (SENT | FAILED) and `sentAt`.
- Add the BILL_DUE trigger: a scheduled check (daily, alongside or within the sync schedule) that finds PENDING bills with due dates within a configurable window (`BILL_DUE_REMINDER_DAYS`, default 3) and creates `Notification(BILL_DUE)` rows for household members (if not already created for the same bill + due window).
- Define email templates for each notification type (MEMBER_ADDED, SYNC_FAILURE, SETTLEMENT_READY, BILL_DUE) using i18n keys and `{{var}}` interpolation (Spec 01 email utility).
- Add a notification processor API route (`POST /api/notifications/process` with `NOTIFICATIONS_SECRET`) triggered by the external scheduler.
- Add retry on send failure (up to `NOTIFICATION_MAX_RETRIES`, default 3).
- All templates in Spanish (default locale).
- No in-app notification center UI in MVP (email only; a future spec can add an in-app bell).

## Capabilities

### New Capabilities
- `notifications`: Email dispatch for PENDING notifications (MEMBER_ADDED, SYNC_FAILURE, SETTLEMENT_READY, BILL_DUE), BILL_DUE trigger, i18n email templates, notification processor with retry.

### Modified Capabilities
- `system-design`: CONSUMES the `Notification` model's `retryCount` field (nullable int, default 0, defined canonically in Spec 01) for email retry tracking. Consumes the `sendEmail` utility from Spec 01. No additional schema changes.

## Impact

- **Code**: `lib/notifications/processNotifications.ts`; `POST /api/notifications/process` route; `POST /api/notifications/bill-due-check` route (or combined into the sync schedule); email templates under `i18n/locales/es/emails/`.
- **Database**: No additional schema changes — `retryCount` (int, default 0) on the `Notification` model is defined canonically in Spec 01. This spec consumes it for email retry tracking.
- **Environment variables**: `NOTIFICATIONS_SECRET`, `NOTIFICATION_MAX_RETRIES`, `BILL_DUE_REMINDER_DAYS`, `NOTIFICATIONS_SCHEDULE_CRON`.
- **Prior specs consumed**: Spec 03 creates MEMBER_ADDED; Spec 04 creates SYNC_FAILURE; Spec 07 creates SETTLEMENT_READY; Spec 08 provides the bills for BILL_DUE.

## Non-goals

- In-app notification center / bell icon (MVP is email only; future spec).
- SMS or push notifications (future).
- User notification preferences UI (MVP sends all operational alerts to all relevant members; preferences are future).
- Real-time notifications (batch processing on a schedule).
- Digest emails (MVP sends individual emails per notification).
- Notification templates in languages other than Spanish (MVP is Spanish-only per i18n rule).
