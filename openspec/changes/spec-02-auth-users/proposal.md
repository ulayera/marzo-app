## Why

Users need to authenticate before accessing any household financial data. Without authentication there is no way to scope transactions, settlements, or bills to a specific user or household. This is the first user-facing capability built on the Spec 01 foundation and the entry point to the onboarding flow (Spec 03).

## What Changes

- Add Google OAuth sign-in / sign-up (redirect-based flow).
- Add email + password registration and login.
- Add HTTP-only session management (cookie-based) with logout.
- Add a basic profile editor (name, email, avatar URL) and password change.
- Add registration → household onboarding redirect (the onboarding UI itself is Spec 03).
- Add auth API routes under `/api/auth/*` and `/api/users/me*`.
- Add rate limiting on auth endpoints and standard auth security (CSRF state for OAuth, password hashing, no passwordHash in responses).
- All auth UI rendered in the `(auth)` route group (no app shell) using Spec 01 components and i18n keys.

## Capabilities

### New Capabilities
- `auth-users`: Google OAuth + password authentication, session management, and basic profile editing for Marzo users.

### Modified Capabilities
- `system-design`: Consumes the `(auth)` route group, User model, API envelope, and components defined in Spec 01. No requirement changes to `system-design` itself.

## Impact

- **Code**: `app/(auth)/login`, `app/(auth)/register`, `app/(settings)/profile` route pages; `/api/auth/*` and `/api/users/me*` API routes; session middleware extended from Spec 01 auth middleware; Google OAuth provider config.
- **Dependencies**: NextAuth or a lightweight OAuth library (e.g. `@auth/core`), bcrypt/argon2, rate-limit middleware.
- **Environment variables**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET` added to `.env.example`.
- **Database**: No new models — uses the `User` model from Spec 01. Clarifies that `AuditLog.userId` is nullable (for failed-login audits with non-existent emails), consistent with the Spec 01 AuditLog definition.
- **Future specs**: Spec 03 (households-onboarding) consumes the authenticated session + redirect target from this spec.

## Non-goals

- Household creation / onboarding UI (Spec 03).
- Password reset / forgot-password email flow (out of MVP scope; users can re-auth via Google if they have it, or contact support).
- Avatar file upload (MVP uses URL only).
- Two-factor authentication (future).
- Multi-device session management UI (future).
- Email verification on registration (MVP skips; duplicate-email guard is the only email check).
