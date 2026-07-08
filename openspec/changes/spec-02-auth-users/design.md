## Context

Spec 01 established the `User` model (id, name, email unique, passwordHash nullable, avatarUrl, timestamps), the `(auth)` route group (no app shell), the `{ data, error }` API envelope, reusable components (Button, Input, Card, Modal, Badge, Spinner, EmptyState, ErrorState, ThemeToggle), i18n (`auth.*` namespace), and a basic auth middleware stub. This spec fills in the actual authentication flows and profile management.

The app supports two auth methods: Google OAuth and email/password. Users pick one at registration. OAuth users have `passwordHash = null`; password users have `passwordHash` set. Both kinds of users can edit their profile and (password users) change their password. The billing-month domain logic is irrelevant here — auth is orthogonal to financial cycles.

## Goals / Non-Goals

**Goals:**
- Google OAuth sign-in/sign-up with redirect flow and CSRF state.
- Email + password registration and login with validation.
- HTTP-only session cookies with logout.
- Basic profile editing (name, email, avatar URL).
- Password change requiring current-password verification.
- Rate limiting on auth endpoints.
- All auth UI in the `(auth)` route group with i18n + theming + empty/loading/error states.

**Non-Goals:**
- Forgot-password / password reset email flow (out of MVP).
- Avatar file upload (URL only).
- 2FA, email verification, multi-device session UI (future).
- Household onboarding (Spec 03).

## Decisions

### D1: Auth library — NextAuth.js (Auth.js)
**Decision:** Use NextAuth.js (Auth.js) with the Google provider and a Credentials provider for password auth.
**Rationale:** Built for Next.js App Router, handles OAuth redirects, session cookies, and CSRF state out of the box. Reduces boilerplate and security pitfalls vs hand-rolling OAuth.
**Alternatives considered:**
- Hand-rolled OAuth + JWT: more control, more security surface, more code. Rejected for MVP speed.
- Lucia-auth: lightweight but less ecosystem support; would require manual Google OAuth implementation.

### D2: Password hashing — bcrypt (cost ≥ 12)
**Decision:** Use bcrypt with a work factor of 12 (or higher, configurable via env).
**Rationale:** Industry standard, well-audited, supported in Node via `bcrypt` or `bcryptjs`. Cost 12 balances security and latency on commodity hardware.
**Alternatives considered:**
- argon2id: newer, memory-hard, arguably better against GPU attacks. Slightly more complex to deploy. Acceptable alternative — config can swap later.
- scrypt: also strong but less common in Node ecosystems.

### D3: Session strategy — JWT in HTTP-only cookie via NextAuth
**Decision:** NextAuth's JWT session strategy (not database sessions), stored in an HTTP-only, Secure (production), SameSite=Lax cookie. `SESSION_SECRET` from env signs the JWT.
**Rationale:** Stateless, no extra DB table, works with server components and API routes via `getServerSession`. JWT contains userId, email, name — no sensitive data.
**Alternatives considered:**
- Database-backed sessions: more revocable, but adds a Session table and lookup per request. Overkill for MVP.
- Self-issued opaque tokens in a cookie: equivalent to NextAuth JWT but more hand-rolled code.

### D4: Rate limiting — in-memory sliding window (MVP) with env-configurable limits
**Decision:** Simple in-memory sliding-window rate limiter middleware on `/api/auth/login` and `/api/auth/register` (e.g. 10 attempts / 10 minutes per IP for login, 5 / 10 minutes for register). Configurable via env (`AUTH_RATE_LIMIT_LOGIN`, `AUTH_RATE_LIMIT_REGISTER`).
**Rationale:** MVP is single-instance; in-memory is sufficient. Interface stays swappable for Redis-backed limiter later.
**Alternatives considered:**
- Redis-backed limiter: needed for multi-instance deployments but premature for MVP.
- No rate limiting: rejected — brute-force protection is a baseline security requirement.

### D5: OAuth first-login user creation and returning-user update
**Decision:** On first successful Google OAuth login (no existing User with that email), create a `User` row with name + email from the Google profile, `passwordHash = null`, `avatarUrl` from Google. On subsequent logins, always refresh `avatarUrl` from Google (users rarely manually edit avatar URLs) but NEVER overwrite `name` after first creation (to preserve manual edits). If the Google email matches an existing password-registered User, reject with `auth.errors.emailExistsPassword` — no account linking in MVP without email verification.
**Rationale:** Avoids schema changes (no need to store "previous Google values") while keeping avatars fresh and preserving name edits. Rejecting email collisions prevents account takeover without email verification.
**Alternatives considered:**
- Store `oauthName`/`oauthAvatarUrl` fields to detect manual edits: adds schema complexity for marginal value. Rejected for MVP simplicity.
- Always overwrite name + avatar from Google: clobbers manual name edits. Rejected.
- Link Google account to existing password user: security risk without email verification. Rejected for MVP.

### D6: Redirect after auth
**Decision:** After successful registration or first OAuth login → redirect to `/onboarding` (Spec 03's route). After successful login of a user who already belongs to at least one household → redirect to `/dashboard`. After successful login of a user with NO household → redirect to `/onboarding`.
**Rationale:** Avoids stranding authenticated users without a household context.
**Alternatives considered:**
- Always redirect to `/dashboard` and let the dashboard detect "no household": extra logic in the dashboard. Rejected — explicit redirect is clearer.

### D7: Profile avatar — URL only
**Decision:** Profile avatar is a URL string (`avatarUrl` on User). No file upload endpoint in MVP.
**Rationale:** OAuth provides a URL already; password users can paste a URL. File upload adds storage + processing complexity out of MVP scope.
**Alternatives considered:**
- S3/Supabase storage upload: future enhancement.

### D8: Email change uniqueness
**Decision:** Email change validates uniqueness against existing Users (excluding the current user). On collision → 400 with `auth.errors.emailInUse`.
**Rationale:** Email is the unique login identifier; collisions break login.

## Risks / Trade-offs

- [OAuth provider outage] → Mitigation: password auth remains a fallback for password users. OAuth-only users are temporarily blocked; surface a clear error.
- [bcrypt cost too high → slow login] → Mitigation: cost is env-configurable; default 12 is tested to stay under 500ms on the target hardware.
- [In-memory rate limiter resets on restart] → Mitigation: acceptable for MVP; document the limitation and plan Redis-backed limiter for production scale.
- [No password reset flow] → Mitigation: users with Google linked can still log in via Google. Document clearly in the spec that password reset is out of MVP.
- [NextAuth lock-in] → Mitigation: session logic is behind our `getSession()` wrapper; swapping providers later requires changing only the auth route handlers.
- [Email change without verification] → Mitigation: MVP accepts the risk; future spec adds email verification. Uniqueness check prevents the worst case (duplicate accounts).
