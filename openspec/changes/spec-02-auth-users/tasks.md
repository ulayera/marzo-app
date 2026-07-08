## 1. Auth Infrastructure Setup

- [ ] 1.1 Install NextAuth.js (`@auth/core` + `next-auth`) and the Google provider, configure the NextAuth route handler at `app/api/auth/[...nextauth]/route.ts`, and verify the OAuth redirect initiates
- [ ] 1.2 Install bcrypt (`bcrypt` or `bcryptjs`) and add a `hashPassword(plaintext)` + `verifyPassword(plaintext, hash)` helper (cost from env, default 12), server-side only
- [ ] 1.3 Add `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET`, `AUTH_RATE_LIMIT_LOGIN`, `AUTH_RATE_LIMIT_REGISTER`, `AUTH_RATE_LIMIT_PASSWORD_CHANGE` to `.env.example` with placeholder values
- [ ] 1.4 Extend the Spec 01 startup env guard to validate the new auth env vars and emit clear errors on missing values

## 2. Rate Limiting

- [ ] 2.1 Implement an in-memory sliding-window rate limiter with an interface compatible with a future Redis backend (e.g. `RateLimiter.check(key, limit, windowMs)`)
- [ ] 2.2 Wire the limiter into `POST /api/auth/login` (default 10/10min per IP), `POST /api/auth/register` (default 5/10min per IP), and `POST /api/users/me/password` (default 5/10min per user) reading limits from env
- [ ] 2.3 Verify 429 responses include `error.code = 'rateLimited'` and a `Retry-After` header when the limit is exceeded

## 3. Auth API Routes

- [ ] 3.1 Implement `POST /api/auth/register` (validate body, hash password, create User, set session via NextAuth, return `{ data: { userId, email, name } }`, return 400 on duplicate email)
- [ ] 3.2 Implement `POST /api/auth/login` (look up user, verify bcrypt, set session, return user data; 401 generic `invalidCredentials` on any failure)
- [ ] 3.3 Implement `POST /api/auth/logout` (clear session cookie, return `{ data: null }`)
- [ ] 3.4 Implement `GET /api/auth/session` (return current user or 401)
- [ ] 3.5 Implement `GET /api/users/me` (return profile without `passwordHash`)
- [ ] 3.6 Implement `PUT /api/users/me` (update name/email/avatarUrl; 400 on email collision with `emailInUse`; 400 `invalidAvatarUrl` on non-URL avatarUrl; refresh session cookie on email change)
- [ ] 3.7 Implement `POST /api/users/me/password` (verify current password, validate new password length, hash and update; 400 `wrongCurrentPassword` on mismatch)
- [ ] 3.8 Verify `passwordHash` never appears in any auth API response

## 4. Google OAuth Wiring

- [ ] 4.1 Configure the Google provider in NextAuth with the OAuth `state` parameter enabled and the callback URL set
- [ ] 4.2 Implement the first-login user creation: on Google sign-in with no matching `User`, create one with name/email/avatarUrl from the Google profile and `passwordHash = null`
- [ ] 4.3 Implement the returning-user update: always refresh `avatarUrl` from Google; NEVER overwrite `name` after first creation
- [ ] 4.4 Implement Google login with existing password-user email: reject with `auth.errors.emailExistsPassword` and no session created
- [ ] 4.5 Verify OAuth failure (denied consent or provider error) redirects back to `/login` with an `ErrorState` and i18n-keyed message

## 5. Session & Redirect Logic

- [ ] 5.1 Verify the session cookie is `httpOnly`, `secure` in production, `sameSite=Lax`, with a max age of 30 days (configurable)
- [ ] 5.2 Implement `getSession()` (or use NextAuth's `getServerSession`) returning `{ userId, email, name }` or null, available to server components and API routes
- [ ] 5.3 Implement post-auth redirect: after register or first OAuth login → `/onboarding`; after login of a user with a household → `/dashboard`; after login of a user with no household → `/onboarding`
- [ ] 5.4 Implement the authenticated-redirect guard for `/login` and `/register` (redirect to dashboard/onboarding if already logged in)
- [ ] 5.5 Verify logout clears the cookie and redirects to `/login`

## 6. i18n Keys

- [ ] 6.1 Add the `auth` namespace JSON under `i18n/locales/es/auth.json` with keys for `login.*`, `register.*`, `profile.*`, `passwordChange.*`, and `errors.*` (including `invalidCredentials`, `emailInUse`, `emailExistsPassword`, `wrongCurrentPassword`, `oauthFailed`, `rateLimited`, `required`, `invalidEmail`, `passwordTooShort`, `passwordMismatch`, `invalidAvatarUrl`)
- [ ] 6.2 Audit all auth screens to confirm no hard-coded user-facing text exists outside the locale file

## 7. Login UI

- [ ] 7.1 Build the login page under `app/(auth)/login` using Spec 01 components (Card, Input, Button, Spinner, ErrorState) with NO app shell
- [ ] 7.2 Add email + password fields with labels, inline validation, and a disabled+spinner submit state during loading
- [ ] 7.3 Add the "Sign in with Google" button (real `<button>`, keyboard focusable) that initiates the OAuth redirect
- [ ] 7.4 Add a link to `/register` and a top-level error area for `invalidCredentials` / `oauthFailed` / `emailExistsPassword` (i18n-keyed)
- [ ] 7.5 Verify all login text uses `auth.login.*` and `auth.errors.*` i18n keys (Spanish default)

## 8. Register UI

- [ ] 8.1 Build the register page under `app/(auth)/register` with name, email, password, confirm-password fields
- [ ] 8.2 Add inline validation (email format, password ≥ 8 chars, confirmation match) with i18n-keyed messages
- [ ] 8.3 Add the Google sign-up button (same OAuth flow as login) and a link back to `/login`
- [ ] 8.4 Add loading state (disabled submit + spinner), error state (duplicate email, validation), and success redirect to `/onboarding`
- [ ] 8.5 Verify all register text uses `auth.register.*` and `auth.errors.*` keys

## 9. Profile UI

- [ ] 9.1 Build the profile page under `app/(settings)/profile` showing an editable form for name, email, avatarUrl (URL input with URL validation)
- [ ] 9.2 Implement save with loading + success + error states (ErrorState with retry on failure; success confirmation without unmounting the form)
- [ ] 9.3 Implement email-change uniqueness error display (`auth.errors.emailInUse`) and session refresh on email change
- [ ] 9.4 Build the password-change section (currentPassword, newPassword, confirmNewPassword) with inline validation and the OAuth-only disabled state with an i18n message
- [ ] 9.5 Verify all profile text uses `auth.profile.*`, `auth.passwordChange.*`, and `auth.errors.*` keys

## 10. Accessibility

- [ ] 10.1 Verify every auth form is keyboard navigable (Tab through fields and buttons; submit via Enter)
- [ ] 10.2 Verify each input label is associated with its input (`for`/`id` or wrapping label)
- [ ] 10.3 Verify inline error messages are announced via `aria-describedby` or `aria-live`
- [ ] 10.4 Verify the Google OAuth button is a real `<button>` and keyboard focusable

## 11. Audit Logging

- [ ] 11.1 Write `AuditLog` entries for: successful login, failed login (userId null if email doesn't exist, entityId = attempted email), registration, password change, profile update, logout (entity `User`, metadata includes auth method + IP)
- [ ] 11.2 Verify audit entries are created for each security-relevant auth event

## 12. Verification & Validation

- [ ] 12.1 Verify the full password registration → session → redirect-to-onboarding flow end to end
- [ ] 12.2 Verify the Google OAuth first-login → user creation → redirect-to-onboarding flow
- [ ] 12.3 Verify returning Google login refreshes avatar and preserves manually-edited name
- [ ] 12.4 Verify Google login with an existing password-user email is rejected with `emailExistsPassword`
- [ ] 12.5 Verify invalid-credentials returns 401 with the same message for wrong-password and non-existent-email
- [ ] 12.6 Verify profile email change to an in-use email returns 400 `emailInUse`
- [ ] 12.7 Verify profile avatar URL validation rejects non-URL strings with `invalidAvatarUrl`
- [ ] 12.8 Verify password change with wrong current password returns 400 `wrongCurrentPassword`
- [ ] 12.9 Verify rate limits return 429 with `Retry-After` after exceeding the configured window (login, register, password change)
- [ ] 12.10 Verify `passwordHash` never appears in any API response
- [ ] 12.11 Verify the `.env.example` lists all new auth env vars with placeholders
- [ ] 12.12 Verify session refresh on email change (`getSession()` returns new email)
- [ ] 12.13 Verify expired session redirects to `/login`
- [ ] 12.14 Run `openspec validate spec-02-auth-users` and resolve any reported issues
