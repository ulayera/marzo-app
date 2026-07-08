## 1. i18n Email Templates

- [ ] 1.1 Create `i18n/locales/es/emails/memberAdded.html` + `.txt` with i18n-keyed text and `{{householdName}}`, `{{addedByName}}` interpolation
- [ ] 1.2 Create `i18n/locales/es/emails/syncFailure.html` + `.txt` with `{{bankName}}`, `{{errorMessage}}` interpolation
- [ ] 1.3 Create `i18n/locales/es/emails/settlementReady.html` + `.txt` with `{{billingMonth}}`, `{{householdName}}` interpolation
- [ ] 1.4 Create `i18n/locales/es/emails/billDue.html` + `.txt` with `{{billTitle}}`, `{{amount}}`, `{{dueDate}}` interpolation
- [ ] 1.5 Audit all email templates to confirm no hard-coded user-facing text (all via i18n keys)

## 2. Schema Verification

- [ ] 2.1 Verify the `Notification` model (from Spec 01) includes the `retryCount` int field (default 0) for email retry tracking; no additional migration is needed from this spec

## 3. Notification Processor

- [ ] 3.1 Implement `processNotifications()` in `lib/notifications/processNotifications.ts`: read PENDING notifications, resolve recipient email via User.email, render template per type, call Spec 01 `sendEmail`, update emailStatus (SENT + sentAt on success; increment retryCount on failure; FAILED after max retries)
- [ ] 3.2 Implement `POST /api/notifications/process` (require `NOTIFICATIONS_SECRET` with constant-time comparison; 401 on missing/invalid; call processNotifications)

## 4. BILL_DUE Trigger

- [ ] 4.1 Implement `POST /api/notifications/bill-due-check` (require `NOTIFICATIONS_SECRET`; find PENDING bills with dueDate within `BILL_DUE_REMINDER_DAYS`; for each, create Notification(BILL_DUE) for each ACTIVE household member with payload {billId, billTitle, amount, dueDate, householdId}; idempotency check — no duplicate for same (billId, userId) with PENDING/SENT status)
- [ ] 4.2 Skip PAID bills in the daily check

## 5. Environment Variables

- [ ] 5.1 Add `NOTIFICATIONS_SECRET`, `NOTIFICATION_MAX_RETRIES`, `BILL_DUE_REMINDER_DAYS`, `NOTIFICATIONS_SCHEDULE_CRON` to `.env.example` with placeholder values
- [ ] 5.2 Extend the startup env guard to validate `NOTIFICATIONS_SECRET` and emit a clear error if missing

## 6. Audit Logging

- [ ] 6.1 Write AuditLog entries for notification SENT and FAILED outcomes (userId=null system action, householdId from notification, entity=Notification, metadata with type + outcome)

## 7. Verification & Validation

- [ ] 7.1 Verify the processor reads PENDING notifications, sends emails, and updates emailStatus to SENT with sentAt
- [ ] 7.2 Verify a send failure increments retryCount and keeps emailStatus PENDING
- [ ] 7.3 Verify after NOTIFICATION_MAX_RETRIES the emailStatus becomes FAILED
- [ ] 7.4 Verify `POST /api/notifications/process` without a valid NOTIFICATIONS_SECRET returns 401
- [ ] 7.5 Verify the BILL_DUE check creates notifications for PENDING bills due within 3 days
- [ ] 7.6 Verify no duplicate BILL_DUE notifications are created for the same (billId, userId)
- [ ] 7.7 Verify PAID bills are skipped by the BILL_DUE check
- [ ] 7.8 Verify each notification type renders the correct template with the correct variables
- [ ] 7.9 Verify all email template text uses i18n keys (no hard-coded literals)
- [ ] 7.10 Verify the `.env.example` lists all notification env vars
- [ ] 7.11 Verify AuditLog entries are written for SENT and FAILED outcomes
- [ ] 7.12 Run `openspec validate spec-09-notifications` and resolve any reported issues
