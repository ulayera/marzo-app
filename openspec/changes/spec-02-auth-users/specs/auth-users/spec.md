## ADDED Requirements

### Requirement: Google OAuth Sign-In / Sign-Up

The application SHALL support Google OAuth via a redirect-based flow. A "Sign in with Google" button SHALL appear on both the login and register pages. The OAuth flow MUST include a `state` parameter to prevent CSRF. On first successful Google login (no existing `User` with that email), the system MUST create a `User` row with `name` and `email` from the Google profile, `passwordHash = null`, and `avatarUrl` from the Google profile picture. On subsequent Google logins, the system MUST refresh `avatarUrl` from the current Google profile on every login (users rarely manually edit avatar URLs), but MUST NOT overwrite `name` after first creation (to preserve any manual edits). If the Google email matches an existing `User` that has a `passwordHash` set (password-registered user), the system MUST reject the OAuth login with a clear i18n-keyed error directing the user to log in with their password (no account linking in MVP without email verification).

#### Scenario: First Google login creates a user
- **WHEN** a new user completes the Google OAuth flow successfully and no `User` with that email exists
- **THEN** a `User` row is created with their Google name, email, avatar URL, and `passwordHash = null`, a session is created, and they are redirected to `/onboarding`

#### Scenario: Returning Google user refreshes avatar
- **WHEN** a returning Google user logs in
- **THEN** their `avatarUrl` is refreshed from the current Google profile, `name` is NOT overwritten, and they are redirected to `/dashboard` (if they have a household) or `/onboarding` (if they do not)

#### Scenario: Google login with existing password-user email
- **WHEN** a user completes Google OAuth with an email that matches an existing `User` whose `passwordHash` is not null
- **THEN** the OAuth login is rejected with an i18n-keyed error (`auth.errors.emailExistsPassword`) directing the user to log in with their password, and no session is created

#### Scenario: OAuth state parameter present
- **WHEN** the Google OAuth redirect is initiated
- **THEN** the redirect URL includes a `state` parameter and the callback rejects any response with a missing or mismatched `state`

#### Scenario: OAuth failure surfaces error
- **WHEN** the Google OAuth flow fails (user denies consent or provider error)
- **THEN** the user is returned to the login page with an i18n-keyed error message displayed in the `ErrorState`

### Requirement: Password Registration

The application SHALL allow new users to register with `name`, `email`, `password`, and `password confirmation`. The system MUST validate: email format, password minimum length (8 characters), password confirmation matches, and all required fields are present. The password MUST be hashed with bcrypt (cost ≥ 12, configurable via env) before storage. On success, a `User` row is created, a session is created, and the user is redirected to `/onboarding`. Duplicate email registration MUST be rejected with a 400 error and an `auth.errors.emailInUse` message.

#### Scenario: Successful password registration
- **WHEN** a user submits valid name, email, password, and matching confirmation
- **THEN** a `User` row is created with a bcrypt-hashed `passwordHash`, a session cookie is set, and the user is redirected to `/onboarding`

#### Scenario: Duplicate email rejected
- **WHEN** a user registers with an email that already exists on a `User` row
- **THEN** the API returns 400 with `error.code = 'emailInUse'` and no user is created

#### Scenario: Weak password rejected
- **WHEN** a user submits a password shorter than 8 characters
- **THEN** the form shows an inline i18n-keyed validation error and the submit does not call the API

#### Scenario: Password confirmation mismatch
- **WHEN** the password and confirmation fields differ
- **THEN** the form shows an inline i18n-keyed error and submission is blocked

#### Scenario: Password never stored in plaintext
- **WHEN** a `User` row is inspected after registration
- **THEN** `passwordHash` contains a bcrypt hash and the plaintext password is not present anywhere in the database or logs

### Requirement: Password Login

The application SHALL allow users to log in with `email` and `password`. The system MUST look up the user by email, verify the password against the stored `passwordHash` using bcrypt's compare function, and on success create a session and redirect to `/dashboard` (if the user has at least one household) or `/onboarding` (if not). Invalid credentials MUST return 401 with a generic `auth.errors.invalidCredentials` message (not revealing whether the email exists).

#### Scenario: Successful password login
- **WHEN** a user submits a valid email and correct password
- **THEN** a session is created and they are redirected to `/dashboard` or `/onboarding` based on household membership

#### Scenario: Invalid credentials
- **WHEN** a user submits an email and an incorrect password (or a non-existent email)
- **THEN** the API returns 401 with `error.code = 'invalidCredentials'` and the same generic message regardless of whether the email exists

#### Scenario: OAuth-only user attempts password login
- **WHEN** a user whose `passwordHash` is null attempts password login
- **THEN** the API returns 401 with `invalidCredentials` (no hint that the account exists with OAuth only)

### Requirement: Session Management

The application SHALL manage sessions via HTTP-only cookies containing a signed JWT (via NextAuth). Cookies MUST be `httpOnly`, `secure` in production, and `sameSite=Lax`. A `getSession()` utility MUST be available for server components and API routes to retrieve the current session (userId, email, name) or null. Sessions MUST expire after a configurable maximum age (default 30 days). Logout MUST destroy the session cookie and redirect to `/login`.

#### Scenario: Cookie flags set
- **WHEN** a session cookie is set in a production environment
- **THEN** it has `httpOnly`, `secure`, and `sameSite=Lax` attributes

#### Scenario: getSession returns the current user
- **WHEN** `getSession()` is called in a server component or API route for an authenticated request
- **THEN** it returns an object with `userId`, `email`, and `name`

#### Scenario: getSession returns null when unauthenticated
- **WHEN** `getSession()` is called for a request with no valid session cookie
- **THEN** it returns null

#### Scenario: Logout destroys session
- **WHEN** a user clicks logout
- **THEN** the session cookie is cleared and the user is redirected to `/login`

#### Scenario: Expired session redirects to login
- **WHEN** a user with an expired session cookie navigates to a protected route
- **THEN** the middleware redirects them to `/login`

### Requirement: Authenticated Redirect Behavior

An already-authenticated user visiting `/login` or `/register` MUST be redirected to `/dashboard` (if they have a household) or `/onboarding` (if they do not). An unauthenticated user visiting a protected route under `(app)` MUST be redirected to `/login` (per Spec 01 routing).

#### Scenario: Authenticated user visits login
- **WHEN** an authenticated user navigates to `/login`
- **THEN** they are redirected to `/dashboard` or `/onboarding`

#### Scenario: Unauthenticated user visits protected route
- **WHEN** an unauthenticated user navigates to a route under `(app)`
- **THEN** they are redirected to `/login`

### Requirement: Profile Viewing and Editing

An authenticated user SHALL be able to view and edit their profile (`name`, `email`, `avatarUrl`). Email changes MUST be validated for uniqueness against existing `User` rows (excluding the current user); on collision, the API returns 400 with `auth.errors.emailInUse`. The `avatarUrl` MUST be a valid URL (rejected with `auth.errors.invalidAvatarUrl` if not). When the email is changed, the session cookie MUST be refreshed so `getSession()` returns the updated email. The profile page MUST show loading, error, and success states. Avatar is a URL string only — no file upload in MVP.

#### Scenario: View own profile
- **WHEN** an authenticated user opens the profile page
- **THEN** their current `name`, `email`, and `avatarUrl` are displayed in an editable form

#### Scenario: Update name and avatar
- **WHEN** a user edits their name and avatarUrl and saves
- **THEN** the `User` row is updated and a success state is shown

#### Scenario: Email change to an in-use email
- **WHEN** a user changes their email to one already used by another `User`
- **THEN** the API returns 400 with `error.code = 'emailInUse'` and the profile is not updated

#### Scenario: Invalid avatar URL rejected
- **WHEN** a user submits an `avatarUrl` that is not a valid URL
- **THEN** the API returns 400 with `error.code = 'invalidAvatarUrl'` and the profile is not updated

#### Scenario: Email change refreshes session
- **WHEN** a user successfully changes their email
- **THEN** the session cookie is refreshed and `getSession()` returns the new email

#### Scenario: Profile save failure
- **WHEN** a profile save request fails (network or server error)
- **THEN** an `ErrorState` is shown with a retry affordance and the form retains the user's input

### Requirement: Password Change

A password-authenticated user SHALL be able to change their password by providing `currentPassword` and `newPassword` (with confirmation). The system MUST verify `currentPassword` against the stored hash; on mismatch return 400 with `auth.errors.wrongCurrentPassword`. The new password MUST meet the same minimum-length validation as registration. On success, the `passwordHash` is updated to a fresh bcrypt hash of the new password. OAuth-only users (`passwordHash = null`) MUST NOT be able to set a password via this flow (a separate "set password" flow is out of MVP scope).

#### Scenario: Successful password change
- **WHEN** a user provides the correct current password and a valid new password
- **THEN** `passwordHash` is updated to a hash of the new password and a success state is shown

#### Scenario: Wrong current password
- **WHEN** a user provides an incorrect current password
- **THEN** the API returns 400 with `error.code = 'wrongCurrentPassword'` and the hash is not changed

#### Scenario: New password too short
- **WHEN** a user provides a new password shorter than 8 characters
- **THEN** an inline validation error is shown and the API is not called

#### Scenario: OAuth-only user cannot change password
- **WHEN** a user with `passwordHash = null` opens the password-change form
- **THEN** the form is disabled or hidden with an i18n-keyed message explaining their account uses Google sign-in

### Requirement: Auth API Routes

The application SHALL expose REST-like auth API routes under `/api/auth/*` and `/api/users/me*`, all returning the Spec 01 `{ data, error }` envelope:

- `POST /api/auth/register` — body: `{ name, email, password }` → creates user, sets session cookie, returns `{ data: { userId, email, name } }`
- `POST /api/auth/login` — body: `{ email, password }` → verifies credentials, sets session cookie, returns `{ data: { userId, email, name } }`
- `POST /api/auth/logout` — clears session cookie, returns `{ data: null }`
- `GET /api/auth/session` — returns `{ data: { userId, email, name } }` or 401 if unauthenticated
- `GET /api/users/me` — returns the current user's profile (never `passwordHash`)
- `PUT /api/users/me` — body: `{ name?, email?, avatarUrl? }` → updates profile, returns updated user
- `POST /api/users/me/password` — body: `{ currentPassword, newPassword }` → changes password
- `GET /api/auth/google` + `GET /api/auth/google/callback` — Google OAuth redirect + callback (handled by NextAuth)

All routes MUST use the Spec 01 response envelope and status codes (400 validation, 401 unauth, 403 forbidden, 500 server). `passwordHash` MUST NEVER appear in any API response.

#### Scenario: Register endpoint shape
- **WHEN** `POST /api/auth/register` is called with valid body
- **THEN** the response body is `{ data: { userId, email, name }, error: null }` with a 200 status and a Set-Cookie header

#### Scenario: Login endpoint error shape
- **WHEN** `POST /api/auth/login` is called with invalid credentials
- **THEN** the response body is `{ data: null, error: { code: 'invalidCredentials', message: '...' } }` with a 401 status

#### Scenario: Session endpoint returns current user
- **WHEN** `GET /api/auth/session` is called with a valid session cookie
- **THEN** the response is `{ data: { userId, email, name }, error: null }` with a 200 status

#### Scenario: Session endpoint rejects unauthenticated
- **WHEN** `GET /api/auth/session` is called with no session cookie
- **THEN** the response is `{ data: null, error: { code: 'unauthorized', message: '...' } }` with a 401 status

#### Scenario: PasswordHash never in response
- **WHEN** any `/api/users/me` or `/api/auth/*` response body is inspected
- **THEN** no `passwordHash` field is present

### Requirement: Rate Limiting on Auth Endpoints

The application MUST apply rate limiting to `POST /api/auth/login`, `POST /api/auth/register`, and `POST /api/users/me/password`. Limits MUST be configurable via environment variables (`AUTH_RATE_LIMIT_LOGIN`, `AUTH_RATE_LIMIT_REGISTER`, `AUTH_RATE_LIMIT_PASSWORD_CHANGE`) with defaults of 10 attempts per 10 minutes (login), 5 per 10 minutes (register), and 5 per 10 minutes (password change, per user) respectively. When the limit is exceeded, the API MUST return 429 with `error.code = 'rateLimited'` and a `Retry-After` header. The rate limiter MUST be interface-compatible with a future Redis-backed implementation.

#### Scenario: Login rate limit enforced
- **WHEN** an IP exceeds the configured login attempt limit within the window
- **THEN** subsequent login attempts from that IP return 429 with `error.code = 'rateLimited'` and a `Retry-After` header

#### Scenario: Register rate limit enforced
- **WHEN** an IP exceeds the configured registration limit within the window
- **THEN** subsequent registration attempts return 429 with `error.code = 'rateLimited'`

#### Scenario: Password change rate limit enforced
- **WHEN** a user exceeds the configured password-change attempt limit within the window
- **THEN** subsequent password-change attempts return 429 with `error.code = 'rateLimited'` and a `Retry-After` header

#### Scenario: Rate limit env-configurable
- **WHEN** the `AUTH_RATE_LIMIT_LOGIN` env variable is set to a different value
- **THEN** the login rate limit reflects the new value without code changes

### Requirement: Auth UI States

Every auth screen (login, register, profile) MUST define empty/initial, loading, error, and success states. Form submit buttons MUST be disabled and show a spinner during loading. Inline validation errors MUST appear next to the relevant field. Top-level errors (e.g. invalid credentials, OAuth failure) MUST render in an `ErrorState` or inline error banner with i18n-keyed text. Profile save success MUST show a success confirmation.

#### Scenario: Login loading state
- **WHEN** a user submits the login form
- **THEN** the submit button is disabled and shows a spinner until the response arrives

#### Scenario: Login error state
- **WHEN** login fails with invalid credentials
- **THEN** an i18n-keyed error message is displayed and the form remains populated with the email (password cleared)

#### Scenario: Register loading state
- **WHEN** a user submits the registration form
- **THEN** the submit button is disabled and shows a spinner until the response arrives

#### Scenario: Register validation error state
- **WHEN** registration fails validation
- **THEN** inline errors appear next to the invalid fields and the form is not submitted

#### Scenario: Profile success state
- **WHEN** a profile save succeeds
- **THEN** a success confirmation is shown without unmounting the form

### Requirement: Auth i18n Keys

All auth user-facing text MUST be rendered via i18n keys under the `auth` namespace. Spanish is the default locale. Keys MUST cover: `auth.login.title`, `auth.login.email`, `auth.login.password`, `auth.login.submit`, `auth.login.googleButton`, `auth.login.registerLink`, `auth.register.*`, `auth.profile.*`, `auth.passwordChange.*`, and `auth.errors.*` (including `invalidCredentials`, `emailInUse`, `wrongCurrentPassword`, `oauthFailed`, `rateLimited`, `required`, `invalidEmail`, `passwordTooShort`, `passwordMismatch`). Interpolation MUST use simple `{{name}}` only.

#### Scenario: All auth text via keys
- **WHEN** any auth screen is rendered
- **THEN** every user-facing string is produced by a `t('auth.*')` call and no hard-coded text appears

#### Scenario: Error messages are i18n-keyed
- **WHEN** an auth error is displayed
- **THEN** the message corresponds to an `auth.errors.*` key, not a hard-coded string

### Requirement: Auth Accessibility

All auth forms MUST be keyboard navigable: every input and button reachable via Tab and operable via Enter/Space. Labels MUST be associated with their inputs (via `for`/`id` or wrapping). Error messages MUST be announced to assistive technology (via `aria-describedby` or `aria-live`). The Google OAuth button MUST be a real button (not a div) and keyboard focusable.

#### Scenario: Keyboard-only login
- **WHEN** a user navigates the login form using only the keyboard
- **THEN** they can focus the email field, password field, submit button, and Google button, and submit the form via Enter

#### Scenario: Form errors announced
- **WHEN** an inline validation error appears on a field
- **THEN** it is associated with the input via `aria-describedby` and announced to assistive technology

### Requirement: Environment Variables for Auth

The `.env.example` file MUST list `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET`, `AUTH_RATE_LIMIT_LOGIN`, and `AUTH_RATE_LIMIT_REGISTER` with placeholder values and no real secrets. The application MUST emit a clear startup error if any required auth env var is missing (consistent with Spec 01's env guard behavior).

#### Scenario: Env example lists auth keys
- **WHEN** the `.env.example` file is inspected
- **THEN** it includes `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET`, `AUTH_RATE_LIMIT_LOGIN`, `AUTH_RATE_LIMIT_REGISTER` with placeholder values

#### Scenario: Missing auth env var surfaced
- **WHEN** the application starts without `SESSION_SECRET` defined
- **THEN** a clear startup error names the missing variable

### Requirement: Auth Audit Logging

The application MUST write an `AuditLog` entry (per Spec 01 AuditLog model) for security-relevant auth events: successful login, failed login, registration, password change, profile update, and logout. The `entity` is `'User'`, `entityId` is the user id (or the attempted email for failed login), and `metadata` includes the auth method (`oauth` | `password`) and IP address.

#### Scenario: Successful login audited
- **WHEN** a user logs in successfully
- **THEN** an `AuditLog` row is created with `action = 'login'`, the user id, and metadata including auth method and IP

#### Scenario: Failed login audited
- **WHEN** a login attempt fails
- **THEN** an `AuditLog` row is created with `action = 'loginFailed'`, `userId` is null (if the email does not match an existing user) or set to the user's id (if the password was wrong), `entityId` set to the attempted email, and metadata including IP

#### Scenario: Password change audited
- **WHEN** a user changes their password
- **THEN** an `AuditLog` row is created with `action = 'passwordChange'` and the user id
