## ADDED Requirements

### Requirement: Household Onboarding Enforcement

The application SHALL redirect any authenticated user with no `ACTIVE` `HouseholdMember` row to `/onboarding` when they navigate to any route under the `(app)` route group. The `/onboarding` route SHALL live under a dedicated `(onboarding)` route group that does NOT render the navigation shell (sidebar/header) and does NOT run the household-membership redirect guard (to avoid an infinite redirect loop). On successful household creation, the user MUST be redirected to `/dashboard`. An authenticated user who already has an `ACTIVE` household membership visiting `/onboarding` MUST be redirected to `/dashboard`.

#### Scenario: No household redirects to onboarding
- **WHEN** an authenticated user with no `ACTIVE` `HouseholdMember` navigates to a route under `(app)`
- **THEN** they are redirected to `/onboarding`

#### Scenario: Onboarding route does not loop
- **WHEN** an authenticated user with no `ACTIVE` `HouseholdMember` is on `/onboarding`
- **THEN** no redirect occurs (the `(onboarding)` group is exempt from the household guard) and the page renders without the navigation shell

#### Scenario: Existing household skips onboarding
- **WHEN** an authenticated user with an `ACTIVE` household visits `/onboarding`
- **THEN** they are redirected to `/dashboard`

### Requirement: Household Creation

An authenticated user SHALL be able to create a household by providing a `name` and `mainCurrency` (ISO 4217 code). The system MUST reject creation if the caller already has an `ACTIVE` `HouseholdMember` row (return 400 with `households.errors.alreadyHasHousehold`) — MVP enforces one household per user at the API level, not just the UI. On creation, the system MUST create a `Household` row with `createdByUserId` set to the current user, create a `HouseholdMember` row with `role = ADMIN` and `status = ACTIVE` linking the creator to the household, and seed the predefined expense `Category` rows (ATM withdrawal, direct debit, money transfer, bank fee, utilities, credit card payment, other) with `isSystem = true` for the new household. The `name` MUST be non-empty; the `mainCurrency` MUST be a valid 3-letter ISO 4217 code validated against a hardcoded ISO 4217 code list in `lib/`.

#### Scenario: Successful household creation
- **WHEN** a user submits a valid household name and main currency
- **THEN** a `Household` is created, the user becomes an `ADMIN` `HouseholdMember`, predefined expense categories are seeded, and they are redirected to `/dashboard`

#### Scenario: User with existing household rejected
- **WHEN** a user who already has an `ACTIVE` `HouseholdMember` calls `POST /api/households`
- **THEN** the API returns 400 with `error.code = 'alreadyHasHousehold'` and no new household is created

#### Scenario: Empty household name rejected
- **WHEN** a user submits an empty household name
- **THEN** an inline i18n-keyed validation error is shown and no household is created

#### Scenario: Invalid currency code rejected
- **WHEN** a user submits a code not in the ISO 4217 code list
- **THEN** an inline i18n-keyed validation error is shown and no household is created

#### Scenario: Predefined categories seeded
- **WHEN** a household is created
- **THEN** `Category` rows exist for that `householdId` with names ATM withdrawal, direct debit, money transfer, bank fee, utilities, credit card payment, other; all with `type = EXPENSE` and `isSystem = true`

### Requirement: Main Currency Selection

The onboarding form SHALL present a curated list of common ISO 4217 currencies (including CLP, USD, EUR, MXN, ARS, BRL, COP, PEN, GBP, JPY) as quick-select options, plus a free-text input for any other valid 3-letter ISO 4217 code. The selected currency is stored as `Household.mainCurrency`. Changing the main currency via household settings SHALL NOT retroactively re-convert existing transactions (existing `amountInMainCurrency` values are preserved; only future transactions use the new currency).

#### Scenario: Quick-select common currency
- **WHEN** a user selects CLP from the curated list
- **THEN** `Household.mainCurrency` is set to `CLP`

#### Scenario: Free-text currency code
- **WHEN** a user types `CAD` in the free-text currency input
- **THEN** `Household.mainCurrency` is set to `CAD` if it is a valid ISO 4217 code

#### Scenario: Currency change does not re-convert history
- **WHEN** an admin changes the household main currency from CLP to USD
- **THEN** existing transactions retain their `amountInMainCurrency` values (computed under the old currency) and only future transactions use USD

### Requirement: Household Settings — View and Edit

An `ACTIVE` household member SHALL be able to view the household name, main currency, and member list. Only an `ADMIN` SHALL be able to edit the household name and main currency. The settings page MUST show loading, error, and success states. When an admin changes the main currency, the UI MUST display an i18n-keyed warning that existing transactions will not be re-converted.

#### Scenario: Member views household info
- **WHEN** any `ACTIVE` member opens the household settings page
- **THEN** the household name, main currency, and member list are displayed

#### Scenario: Admin edits household name
- **WHEN** an admin changes the household name and saves
- **THEN** the `Household.name` is updated and a success state is shown

#### Scenario: Member cannot edit settings
- **WHEN** a non-admin member attempts to edit the household name or currency
- **THEN** the edit controls are disabled or hidden and the API returns 403 if called directly

#### Scenario: Currency change warning displayed
- **WHEN** an admin opens the currency edit control
- **THEN** an i18n-keyed warning is shown stating that existing transactions will not be re-converted

### Requirement: Member Addition

An `ADMIN` SHALL be able to add a member to the household by providing an email address. The system MUST look up a `User` by email. If found, create a `HouseholdMember` with `role = MEMBER` and `status = ACTIVE`. If the user is already an `ACTIVE` member of this household, return 400 with `households.errors.alreadyMember`. If the user is already a `REMOVED` member, reactivate the existing row (`status = ACTIVE`, clear `removedAt`) without creating a duplicate, and return 200 (not 201, since no new row is created). If no `User` exists with that email, return 404 with `households.errors.userNotFound`. On successful addition (new or reactivated), a `Notification` row with `type = MEMBER_ADDED` MUST be created with: `userId` set to the added member, `householdId` set to the household, `emailStatus = PENDING`, and `payload` set to `{ householdId, householdName, addedByUserId, addedByName }`. Email dispatch is deferred to Spec 09.

#### Scenario: Add a new member successfully
- **WHEN** an admin adds a user by email who has an account and is not already a member
- **THEN** a `HouseholdMember(MEMBER, ACTIVE)` row is created (201 response) and a `Notification(MEMBER_ADDED)` row is created with `userId` = added member, `householdId`, `emailStatus = PENDING`, and `payload = { householdId, householdName, addedByUserId, addedByName }`

#### Scenario: Add an already-active member
- **WHEN** an admin adds a user who is already an `ACTIVE` member of this household
- **THEN** the API returns 400 with `error.code = 'alreadyMember'` and no notification is created

#### Scenario: Reactivate a removed member
- **WHEN** an admin adds a user who is a `REMOVED` member of this household
- **THEN** the existing `HouseholdMember` row is updated to `status = ACTIVE` and `removedAt` is cleared (no duplicate, 200 response), and a fresh `Notification(MEMBER_ADDED)` row is created

#### Scenario: Add a user with no account
- **WHEN** an admin adds an email that does not match any `User`
- **THEN** the API returns 404 with `error.code = 'userNotFound'` and an i18n message suggesting the person register first

### Requirement: Member Removal

An `ADMIN` SHALL be able to remove a household member. Removal MUST set `HouseholdMember.status = REMOVED` and `removedAt = now()`. The removed member's transactions, bills, and other data MUST be retained (not deleted). A removed member MUST NOT be able to access any household-scoped API route (403 per Spec 01 membership middleware). An admin SHALL NOT be able to remove themselves (return 400 with `households.errors.cannotRemoveSelf`) to prevent leaving the household without an admin.

#### Scenario: Remove a member
- **WHEN** an admin removes a member
- **THEN** the member's `HouseholdMember.status` becomes `REMOVED`, `removedAt` is set, and their data is retained

#### Scenario: Removed member loses access
- **WHEN** a removed member attempts to access a household-scoped API route
- **THEN** the API returns 403

#### Scenario: Admin cannot remove themselves
- **WHEN** an admin attempts to remove their own membership
- **THEN** the API returns 400 with `error.code = 'cannotRemoveSelf'`

#### Scenario: Non-admin cannot remove members
- **WHEN** a non-admin member attempts to remove another member
- **THEN** the API returns 403

### Requirement: Member List View

Any `ACTIVE` household member SHALL be able to view the household member list showing each member's name, email, role, and status (ACTIVE/REMOVED). Removed members MUST appear in the list with a `REMOVED` badge (using the Spec 01 Badge component) so admins can see historical membership. The list MUST use the Spec 01 DataTable with dense rows, infinite scroll (if the list grows), loading skeleton, and empty state (empty only if the admin is the sole member and just created the household — show a prompt to add members).

#### Scenario: View member list
- **WHEN** any `ACTIVE` member opens the member list
- **THEN** all members (ACTIVE and REMOVED) are displayed with name, email, role, and status badge

#### Scenario: Removed members visible with badge
- **WHEN** the member list includes a removed member
- **THEN** they appear with a `REMOVED` badge (distinct semantic color)

#### Scenario: Sole admin sees add-member prompt
- **WHEN** the only member is the admin
- **THEN** an `EmptyState` or prompt is shown encouraging the admin to add members

### Requirement: Household API Routes

The application SHALL expose REST-like household API routes, all returning the Spec 01 `{ data, error }` envelope and enforcing auth + membership + role middleware:

- `POST /api/households` — create a household (body: `{ name, mainCurrency }`) → creator becomes ADMIN
- `GET /api/households/[id]` — get household info (any ACTIVE member)
- `PUT /api/households/[id]` — update name/mainCurrency (ADMIN only)
- `GET /api/households/[id]/members` — list members (any ACTIVE member)
- `POST /api/households/[id]/members` — add member by email (ADMIN only, body: `{ email }`)
- `DELETE /api/households/[id]/members/[userId]` — remove member (ADMIN only, cannot remove self)

All routes MUST use the `{ data, error }` envelope, 401/403/400/404/500 status codes per Spec 01, and verify the caller is an ACTIVE member (and ADMIN for mutation routes) before executing.

#### Scenario: Create household route
- **WHEN** `POST /api/households` is called with a valid body by an authenticated user
- **THEN** the response is `{ data: { householdId, name, mainCurrency }, error: null }` with 201 status

#### Scenario: Non-admin mutation rejected
- **WHEN** a non-admin calls `PUT /api/households/[id]` or member-mutation routes
- **THEN** the response is 403 with `error.code = 'forbidden'`

#### Scenario: Add member route shape
- **WHEN** `POST /api/households/[id]/members` is called by an admin with a valid email
- **THEN** the response is `{ data: { memberId, userId, role, status }, error: null }` with 201 status

#### Scenario: Remove member route
- **WHEN** `DELETE /api/households/[id]/members/[userId]` is called by an admin
- **THEN** the member is soft-deleted and the response is `{ data: null, error: null }` with 200 status

### Requirement: Onboarding UI States

The onboarding page MUST define empty/initial, loading, error, and success states. The submit button MUST be disabled and show a spinner during creation. Validation errors (empty name, invalid currency) MUST appear inline. Server errors (e.g. database failure) MUST render in an `ErrorState` with a retry affordance. On success, the user is redirected to `/dashboard`.

#### Scenario: Onboarding loading state
- **WHEN** a user submits the household creation form
- **THEN** the submit button is disabled and shows a spinner until the response arrives

#### Scenario: Onboarding validation error
- **WHEN** a user submits an empty name or invalid currency
- **THEN** inline i18n-keyed errors appear next to the relevant fields and the API is not called

#### Scenario: Onboarding server error
- **WHEN** household creation fails due to a server error
- **THEN** an `ErrorState` is shown with a retry action and the form retains the user's input

### Requirement: Household Settings UI States

The household settings page MUST define loading, error, and success states for both the household info edit and member management. Member add/remove operations MUST show loading state on the action button and inline error on failure. The member list MUST show a loading skeleton during fetch and an `ErrorState` on failure.

#### Scenario: Settings loading state
- **WHEN** the settings page is opened
- **THEN** a loading skeleton is shown for the household info and member list until data arrives

#### Scenario: Settings save success
- **WHEN** an admin saves household name or currency changes
- **THEN** a success confirmation is shown without unmounting the form

#### Scenario: Settings save error
- **WHEN** a settings save request fails
- **THEN** an `ErrorState` is shown with a retry action and the form retains the user's input

#### Scenario: Member add loading
- **WHEN** an admin submits the add-member form
- **THEN** the add button is disabled and shows a spinner until the response arrives

#### Scenario: Member add error
- **WHEN** adding a member fails (user not found or already a member)
- **THEN** an inline i18n-keyed error is shown and the form remains visible

#### Scenario: Member removal loading and success
- **WHEN** an admin initiates a member removal
- **THEN** the remove action shows a loading state until the response arrives, and on success the member's badge updates to REMOVED in the list

#### Scenario: Member removal error
- **WHEN** a member removal fails (e.g. attempting to remove self or server error)
- **THEN** an inline i18n-keyed error is shown and the member's status is unchanged

#### Scenario: Member list load error
- **WHEN** the member list fails to load
- **THEN** an `ErrorState` with a retry action is shown in place of the list

### Requirement: Household i18n Keys

All household user-facing text MUST be rendered via i18n keys under the `households` namespace (and `common` where shared). Spanish is the default locale. Keys MUST cover: `households.onboarding.*`, `households.settings.*`, `households.members.*`, and `households.errors.*` (including `userNotFound`, `alreadyMember`, `cannotRemoveSelf`, `invalidCurrency`, `requiredName`, `currencyChangeWarning`).

#### Scenario: All household text via keys
- **WHEN** any household screen is rendered
- **THEN** every user-facing string is produced by a `t('households.*')` or `t('common.*')` call

#### Scenario: Error messages are i18n-keyed
- **WHEN** a household error is displayed
- **THEN** the message corresponds to a `households.errors.*` key

### Requirement: Household Audit Logging

The application MUST write an `AuditLog` entry for household mutations: household creation, settings update, member addition, and member removal. The `userId` column MUST be set to the actor's userId (per Spec 01 AuditLog model). The `householdId` MUST be set to the relevant household. The `entity` is `'Household'` or `'HouseholdMember'`, `entityId` is the relevant id, and `metadata` includes the changed fields.

#### Scenario: Household creation audited
- **WHEN** a household is created
- **THEN** an `AuditLog` row is created with `action = 'householdCreated'`, `entity = 'Household'`, and the household id

#### Scenario: Member removal audited
- **WHEN** a member is removed
- **THEN** an `AuditLog` row is created with `action = 'memberRemoved'`, `entity = 'HouseholdMember'`, the removed member's id, and metadata including the actor's userId
