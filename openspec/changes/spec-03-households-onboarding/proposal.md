## Why

After authentication (Spec 02), users have no household context — every downstream feature (bank accounts, transactions, settlements, bills, dashboard) is scoped to a household. Onboarding is the bridge that creates a household, assigns the user as Admin, and lets them add members. Without this spec, the app has no container for financial data.

## What Changes

- Add a household onboarding flow: after registration/first-login, prompt the user to create a household ("hogar") with a name and main currency.
- Add household creation API + UI (the creator becomes `ADMIN` `HouseholdMember`).
- Seed predefined expense `Category` rows for the new household (per Spec 01).
- Add member management: admin can invite/add members by email and remove members.
- Add household settings: edit household name, main currency.
- Enforce Admin vs Member permissions on household-scoped mutations.
- Removed members retain their data but lose access (per Spec 01 HouseholdMember `status` = `REMOVED`).
- All household UI in the `(app)` route group (onboarding itself is in a dedicated `(onboarding)` group without the app shell or household guard) using Spec 01 components, i18n, theming, and empty/loading/error states.
- Email notification on member-added is wired in Spec 09 (notifications); this spec only creates the `Notification` row with type `MEMBER_ADDED` and leaves email dispatch to Spec 09.

## Capabilities

### New Capabilities
- `households-onboarding`: Household creation during onboarding, member management (add/remove/role), household settings, and Admin/Member permission enforcement.

### Modified Capabilities
- `system-design`: Consumes the `Household`, `HouseholdMember`, and `Category` models from Spec 01. No requirement changes to `system-design` itself.

## Impact

- **Code**: `app/(onboarding)/onboarding` route (no app shell, no household guard); `app/(settings)/household` settings routes; `app/(app)/household` member-list route; `/api/households/*` and `/api/households/[id]/members/*` API routes.
- **Database**: No schema changes — uses `Household`, `HouseholdMember`, `Category`, `Notification` models from Spec 01.
- **Dependencies**: None new.
- **Future specs**: Spec 04 (bank-accounts) requires the user to have a household + member row. Spec 09 (notifications) consumes the `MEMBER_ADDED` Notification rows this spec creates.

## Non-goals

- Role changes / ownership transfer (MVP: members are created as MEMBER; no promote/demote path; admins cannot remove themselves, ensuring at least one admin persists).
- Email delivery of invitations (Spec 09 wires the email trigger; this spec only creates the Notification record).
- Member self-registration via invite link (MVP: admin adds members by email; the member must already have an account or creates one via the standard auth flow, then the admin adds them).
- Household deletion (MVP: households are not deleted; members can be removed but the household persists).
- Multiple households per user (MVP: a user belongs to one household; the schema supports multiple but the UI and onboarding enforce one for simplicity).
- Custom category creation (the spec seeds predefined categories only; custom category creation is deferred to a future spec).
