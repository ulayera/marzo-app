## 1. i18n Keys

- [ ] 1.1 Add the `households` namespace JSON under `i18n/locales/es/households.json` with keys for `onboarding.*`, `settings.*`, `members.*`, and `errors.*` (including `userNotFound`, `alreadyMember`, `cannotRemoveSelf`, `invalidCurrency`, `requiredName`, `currencyChangeWarning`)
- [ ] 1.2 Audit all household screens to confirm no hard-coded user-facing text exists outside the locale file

## 2. Household API Routes

- [ ] 2.1 Implement `POST /api/households` (validate name + mainCurrency against ISO 4217 list, reject if caller already has ACTIVE HouseholdMember with 400 `alreadyHasHousehold`, create Household with createdByUserId, create ADMIN HouseholdMember, seed predefined expense categories, return `{ data: { householdId, name, mainCurrency } }` with 201)
- [ ] 2.2 Implement `GET /api/households/[id]` (return household info; verify ACTIVE membership; 403 for non-members)
- [ ] 2.3 Implement `PUT /api/households/[id]` (update name/mainCurrency; ADMIN only; 403 for non-admin; show currency-change warning in UI but no retroactive conversion)
- [ ] 2.4 Implement `GET /api/households/[id]/members` (list all members including REMOVED; any ACTIVE member)
- [ ] 2.5 Implement `POST /api/households/[id]/members` (look up User by email; ADMIN only; create MEMBER or reactivate REMOVED with 200 for reactivation / 201 for new; create Notification(MEMBER_ADDED) with userId=added member, householdId, emailStatus=PENDING, payload={householdId, householdName, addedByUserId, addedByName}; 404 userNotFound, 400 alreadyMember)
- [ ] 2.6 Implement `DELETE /api/households/[id]/members/[userId]` (soft-delete: status REMOVED, removedAt set; ADMIN only; 400 cannotRemoveSelf; 403 for non-admin)

## 3. Membership & Role Middleware

- [ ] 3.1 Extend the Spec 01 membership middleware with an admin-role check helper (e.g. `requireAdmin(householdId, session)`) that short-circuits with 403 when the caller is not an ADMIN
- [ ] 3.2 Wire admin-role checks into the household mutation routes (PUT household, POST/DELETE members)

## 4. Onboarding UI

- [ ] 4.1 Build the onboarding page at `app/(onboarding)/onboarding` (dedicated route group, NO app shell/nav, NO household guard) with a single-step form: household name input + main currency select (curated list + free-text ISO 4217 input validated against the lib/ code list)
- [ ] 4.2 Add inline validation (non-empty name, valid ISO 4217 currency from the code list) with i18n-keyed errors
- [ ] 4.3 Add loading state (disabled submit + spinner), error state (ErrorState with retry on server failure), and success redirect to `/dashboard`
- [ ] 4.4 Implement the onboarding enforcement guard in the `(app)` layout ONLY: redirect users with no ACTIVE household to `/onboarding`; the `(onboarding)` group is exempt from this guard. Redirect users with a household who visit `/onboarding` to `/dashboard`

## 5. Household Settings UI

- [ ] 5.1 Build the household settings page at `app/(settings)/household` showing the household name, main currency, and member list
- [ ] 5.2 Implement admin-only edit controls for name and main currency (disabled/hidden for non-admins) with the currency-change warning
- [ ] 5.3 Implement settings save with loading + success + error states (ErrorState with retry on failure; success confirmation without unmounting the form)

## 6. Member Management UI

- [ ] 6.1 Build the member list using the Spec 01 DataTable (dense rows, status Badge for ACTIVE/REMOVED, loading skeleton, EmptyState when sole admin)
- [ ] 6.2 Implement the add-member form (email input, add button with loading + error states) for admins
- [ ] 6.3 Implement the remove-member action (admin only; cannot-remove-self guard; loading state on the action)
- [ ] 6.4 Verify removed members appear in the list with a REMOVED badge and cannot access household routes

## 7. Audit Logging

- [ ] 7.1 Write `AuditLog` entries for: household creation, settings update, member addition, member removal (userId = actor, householdId = household, entity `Household` or `HouseholdMember`, metadata includes changed fields)
- [ ] 7.2 Verify audit entries are created for each household mutation

## 8. Verification & Validation

- [ ] 8.1 Verify the full onboarding flow: authenticated user with no household → /onboarding (no nav shell, no loop) → create household → redirect to /dashboard with categories seeded
- [ ] 8.2 Verify an existing-household user visiting /onboarding is redirected to /dashboard
- [ ] 8.3 Verify a user with an existing household cannot create a second via `POST /api/households` (400 alreadyHasHousehold)
- [ ] 8.4 Verify admin can add a member by email and a Notification(MEMBER_ADDED) row is created with the correct payload and emailStatus=PENDING
- [ ] 8.5 Verify adding an already-active member returns 400 alreadyMember
- [ ] 8.6 Verify reactivating a removed member updates the existing row (no duplicate, 200 response, fresh notification created)
- [ ] 8.7 Verify adding a non-existent email returns 404 userNotFound
- [ ] 8.8 Verify admin can remove a member (soft delete, data retained, REMOVED badge shown)
- [ ] 8.9 Verify admin cannot remove themselves (400 cannotRemoveSelf)
- [ ] 8.10 Verify non-admin members get 403 on mutation routes
- [ ] 8.11 Verify currency change does not re-convert existing transactions
- [ ] 8.12 Verify all household text uses i18n keys (Spanish default)
- [ ] 8.13 Run `openspec validate spec-03-households-onboarding` and resolve any reported issues
