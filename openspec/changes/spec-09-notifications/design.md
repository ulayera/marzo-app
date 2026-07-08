## Context

Spec 01 defined the `Notification` model (userId, householdId nullable, type, payload JSON, emailStatus PENDING|SENT|FAILED, sentAt nullable) and the `sendEmail({ to, templateName, locale, variables })` utility with i18n templates under `i18n/locales/es/emails/`. Specs 03 (MEMBER_ADDED), 04 (SYNC_FAILURE), and 07 (SETTLEMENT_READY) create PENDING notifications with defined payloads. Spec 08 provides bills with due dates for the BILL_DUE trigger. This spec wires the dispatch.

## Goals / Non-Goals

**Goals:**
- Notification processor: read PENDING, render template, send email, update emailStatus.
- BILL_DUE trigger: daily check for bills due within N days.
- Email templates for all 4 types (i18n, Spanish, `{{var}}` interpolation).
- Retry on failure (3 attempts).
- Scheduled via external scheduler + secret.

**Non-Goals:**
- In-app notification center, SMS/push, preferences UI, real-time, digests, non-Spanish templates.

## Decisions

### D1: Notification processor trigger
**Decision:** `POST /api/notifications/process` with a `NOTIFICATIONS_SECRET` header (constant-time comparison), triggered by the external scheduler (same pattern as Spec 04's sync route, can share the cron schedule or run separately). The route reads PENDING notifications, processes them, and updates emailStatus. This keeps the app serverless-friendly.
**Rationale:** Consistent with the sync trigger pattern. No long-running process.
**Alternatives considered:**
- Process immediately on creation (in the creating request): couples email delivery to the request lifecycle; if the email provider is slow, the request stalls. Rejected.
- Queue-based worker: overkill for MVP. Deferred.

### D2: BILL_DUE trigger
**Decision:** A daily check (triggered by the external scheduler hitting `POST /api/notifications/bill-due-check` with `NOTIFICATIONS_SECRET`, or combined into the process route) that finds PENDING bills with `dueDate` within `BILL_DUE_REMINDER_DAYS` (default 3) of today. For each, create a `Notification(BILL_DUE)` row for each household member with `userId` = member, `householdId`, `emailStatus = PENDING`, `payload = { billId, title, amount, dueDate, householdId }`. Idempotency: do NOT create a duplicate BILL_DUE for the same `(billId, member)` if one already exists with emailStatus PENDING or SENT (check before insert).
**Rationale:** Daily check is sufficient for a 3-day window. Idempotency prevents duplicate reminders.
**Alternatives considered:**
- Per-bill scheduling: complex. Rejected.
- No BILL_DUE (only the other 3 types): user specified bill-due alerts. Rejected.

### D3: Email templates
**Decision:** One HTML + text template pair per notification type under `i18n/locales/es/emails/`:
- `memberAdded.html` / `.txt` — variables: `{{householdName}}`, `{{addedByName}}`
- `syncFailure.html` / `.txt` — variables: `{{bankName}}`, `{{errorMessage}}`
- `settlementReady.html` / `.txt` — variables: `{{billingMonth}}`, `{{householdName}}`
- `billDue.html` / `.txt` — variables: `{{billTitle}}`, `{{amount}}`, `{{dueDate}}`
All text via i18n keys (no hard-coded user-facing text in templates). Subject lines also i18n-keyed.
**Rationale:** Consistent with Spec 01 email utility. Simple `{{var}}` interpolation.
**Alternatives considered:**
- Single template with conditional sections: harder to maintain. Rejected.

### D4: Retry on send failure
**Decision:** If `sendEmail` fails, the notification's `emailStatus` stays PENDING and a `retryCount` is tracked (add a `retryCount` int field to Notification — schema addition, or track via metadata). Retry up to `NOTIFICATION_MAX_RETRIES` (default 3) on subsequent processor runs. After max retries, `emailStatus = FAILED`.
**Rationale:** Transient email provider failures deserve retries. Capping prevents infinite loops.
**Note:** The Spec 01 Notification model does not have a `retryCount` field. This spec adds a nullable `retryCount` (int, default 0) to the Notification model — a schema addition acknowledged in the proposal.

### D5: Recipient resolution
**Decision:** The processor resolves the recipient email by looking up the `User` by `Notification.userId`. If the user has no email or is deleted (future), the notification is marked FAILED with a logged reason. The email is sent to the user's email address.
**Rationale:** Direct and simple.

## Risks / Trade-offs

- [`retryCount` schema addition] → Mitigation: acknowledged in the proposal; a Prisma migration adds the nullable field.
- [Email provider outage] → Mitigation: retries with backoff (on subsequent processor runs); after max retries, FAILED. No user-facing impact beyond a missed email.
- [Duplicate BILL_DUE reminders] → Mitigation: idempotency check before insert (same billId + member with PENDING/SENT notification).
- [No in-app notification center] → Mitigation: email is the MVP channel; users check email. A future spec can add an in-app bell.
- [No preferences UI] → Mitigation: MVP sends all operational alerts. Preferences are future.
