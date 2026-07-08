## Context

Spec 01 defined the `Household` (id, name, mainCurrency, createdByUserId), `HouseholdMember` (householdId, userId, role ADMIN|MEMBER, status ACTIVE|REMOVED, joinedAt, removedAt, unique [householdId, userId]), `Category` (householdId, name, type, isSystem, unique [householdId, name, type]), and `Notification` (type MEMBER_ADDED) models. Spec 02 provides the authenticated session and redirects first-time users to `/onboarding`. This spec fills in the household creation flow, member management, and settings.

The billing cycle (25th–24th) is irrelevant here — households are created independent of billing months. Currency selection at household creation determines the `mainCurrency` used by all downstream FX conversions (Spec 05).

## Goals / Non-Goals

**Goals:**
- Post-registration onboarding prompt to create a household with name + main currency.
- Creator becomes ADMIN HouseholdMember; predefined expense categories seeded.
- Admin can add members by email (member must have an existing account), remove members (data retained, access revoked), and edit household name/currency.
- Members can view household info but only admin can mutate settings/members.
- All UI states (empty, loading, error, success) defined for every screen.
- Create a `Notification` row (type `MEMBER_ADDED`) when a member is added — email dispatch is Spec 09.

**Non-Goals:**
- Email delivery of invitations (Spec 09).
- Invite-link self-registration (MVP: admin adds existing users by email).
- Household deletion (MVP).
- Multiple households per user in the UI (MVP: one household per user; schema supports more).
- Custom category creation UI (MVP seeds predefined only).

## Decisions

### D1: Onboarding enforcement and route placement
**Decision:** The onboarding page lives under a dedicated `app/(onboarding)` route group — NOT under `(app)`. This group does NOT render the nav shell (sidebar/header) and does NOT run the household-membership redirect guard. The `(app)` route group's layout checks for household membership: if an authenticated user has no `ACTIVE` `HouseholdMember` row, redirect to `/onboarding`. The onboarding page is a single-step form (household name + main currency select). On submit → create Household → create HouseholdMember(ADMIN) → seed categories → redirect to `/dashboard`. An authenticated user who already has an `ACTIVE` household visiting `/onboarding` is redirected to `/dashboard`.
**Rationale:** Placing `/onboarding` under `(app)` would cause an infinite redirect loop (the `(app)` guard redirects no-household users to `/onboarding`, which is under `(app)`, which redirects again). A dedicated route group avoids the loop and the broken UX of rendering an empty nav shell for a pre-household user. Single-step form avoids wizard complexity for 2 fields.
**Alternatives considered:**
- Keep `/onboarding` in `(app)` with an exemption clause: fragile, couples the guard to a route path. Rejected.
- Multi-step wizard: overkill for 2 fields. Rejected.
- Let users skip onboarding: would leave them in a broken state (no household context). Rejected.

### D2: Member addition — by email, existing accounts only
**Decision:** Admin enters an email; the system looks up a `User` by email. If found, create a `HouseholdMember(MEMBER)`. If not found, return 404 with `households.errors.userNotFound` and an i18n message suggesting the person register first. No invite email or link in MVP.
**Rationale:** Avoids email-delivery dependency in this spec. The member-added Notification row is still created for Spec 09 to email later.
**Alternatives considered:**
- Email invitation with a token link: requires email delivery (Spec 09 dependency) and token model. Deferred.
- Username-based lookup: users don't have usernames. Rejected.

### D3: Main currency selection
**Decision:** The onboarding form presents a curated list of common ISO 4217 currencies (CLP, USD, EUR, MXN, ARS, BRL, COP, PEN, GBP, JPY) plus a free-text ISO 4217 input for others. Stored as a 3-letter code on `Household.mainCurrency`. Changing the main currency later is allowed (admin setting) but does NOT retroactively re-convert existing transactions (those keep their `amountInMainCurrency` as-is; only future transactions use the new main currency).
**Rationale:** Common currencies first for UX; free-text for completeness. No retroactive conversion avoids historical data mutation complexity.
**Alternatives considered:**
- Lock currency after creation: too rigid for a household that realizes they picked wrong. Rejected.
- Retroactive re-conversion on currency change: complex, mutates historical financial records. Rejected for MVP.

### D4: Predefined category seeding
**Decision:** On household creation, seed the predefined expense categories from Spec 01 (ATM withdrawal, direct debit, money transfer, bank fee, utilities, credit card payment, other) as `Category` rows with `householdId`, `type = EXPENSE`, `isSystem = true`. No income categories are predefined (income is determined by amount sign, not category, in MVP).
**Rationale:** Ensures every household has a consistent baseline for expense categorization.
**Alternatives considered:**
- Seed income categories too: income is sign-based in MVP, no categories needed. Rejected.

### D5: Member removal — soft delete with data retention
**Decision:** Removing a member sets `HouseholdMember.status = REMOVED` and `removedAt = now()`. The row and all their transactions, bills, etc. remain. The removed user can no longer access the household (Spec 01 membership middleware returns 403 for REMOVED members). Re-adding the same user reactivates the existing row (status → ACTIVE, cleared removedAt) rather than creating a duplicate (the unique constraint prevents a second row anyway).
**Rationale:** Preserves historical financial data integrity (settlements, transactions) while revoking access.
**Alternatives considered:**
- Hard delete: destroys financial history. Rejected.
- Anonymize removed member's transactions: over-engineered for MVP. Rejected.

### D6: Permission enforcement
**Decision:** Household-scoped mutation routes (add/remove member, edit settings) require `role = ADMIN`. Read routes (view members, view household info) allow any `ACTIVE` member. Enforced at the API middleware layer (extending Spec 01's membership middleware with a role check).
**Rationale:** Clear, simple RBAC. All members can see financial data; only admin can manage.
**Alternatives considered:**
- All members can manage: no accountability for settings changes. Rejected.
- Three-tier (admin/member/viewer): overview says two tiers. Rejected.

## Risks / Trade-offs

- [Admin leaves without transferring ownership] → Mitigation: MVP does not support ownership transfer; if the admin removes themselves, the household has no admin. Document this as a known limitation; a future spec can add ownership transfer. For MVP, recommend the admin not remove themselves.
- [Member email not found] → Mitigation: clear error message directing the admin to have the person register first. Acceptable for MVP.
- [Currency change does not re-convert history] → Mitigation: documented in the settings UI with an i18n warning when changing currency.
- [No invite email] → Mitigation: admin can communicate externally; the Member-Added notification (Spec 09) will email the added member once notifications are wired.
- [User in multiple households] → Mitigation: onboarding creates one household; the UI does not offer creating/joining a second. The schema supports it for future expansion.
