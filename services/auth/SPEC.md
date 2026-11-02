# Opsweave Auth — SPEC

The token model, cookies, password rules and service tokens are defined in `docs/ARCHITECTURE.md` §3. This file covers the endpoints and data.

- **Owns:** login credentials, refresh tokens, signing keys, set-password tokens.
- **Consumes:** `roster.employees`.
- **Publishes:** nothing.
- **API:** REST under `/api/v1/auth`.
- **UI:** none of its own. Its pages live in the Portal.

## Data (`auth` DB)
- `credentials`: `user_id` PK, `email` unique, `status`, `password_hash` (nullable until set), `failed_attempts`, `locked_until`, `password_changed_at`, `apps` (jsonb, copied from the event and used to build the `apps` claim)
- `refresh_tokens`: `id`, `user_id`, `token_hash`, `family_id`, `expires_at`, `used_at`, `revoked_at`, `user_agent`, `ip`
- `password_tokens`: `id`, `user_id`, `token_hash`, `purpose` (`INVITE` | `RESET`), `expires_at`, `used_at`
- `auth_signing_keys`: `kid`, `public_key`, `private_key_encrypted`, `active`, `created_at`
- `service_clients`: `client_id`, `secret_hash`, `scopes` (seeded from env)
- `processed_events`

## Consumer rules (`roster.employees`)
- **CREATED:** insert credentials with `password_hash = null`, then create an INVITE token and email it. If `DEMO_SEED=true` and the employee is in the seed list, set the demo password instead.
- **UPDATED:** update `email`, `status` and `apps`. A changed email takes effect at the next login.
- **DEACTIVATED:** set the status, revoke every refresh token, invalidate open password tokens.
- **REACTIVATED:** set the status. If no password was ever set, send a fresh invite.

## Endpoints

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /login` | public | `{email, password}` → `{accessToken, expiresIn, user: {userId, email, apps}}` and sets the refresh cookie |
| `POST /refresh` | refresh cookie + `X-Requested-With` | rotates the refresh token and returns a new access token |
| `POST /logout` | refresh cookie | revokes the token family and clears the cookie |
| `POST /password/set` | public + token | `{token, newPassword}` for both invite and reset |
| `POST /password/forgot` | public | `{email}`. Always returns 202, so it never reveals whether an account exists. |
| `POST /password/change` | user | `{currentPassword, newPassword}`. Revokes the user's other sessions. |
| `GET /password/token/{token}` | public | checks a token is valid before the form is shown |
| `POST /invites/{userId}` | service token (`invites:send`) | resend an invite, called by Roster |
| `POST /service-token` | client credentials | issues a service token |
| `GET /.well-known/jwks.json` | public | public keys |

## Rules
- Login is refused for these cases (all return a generic `401 invalid-credentials`, except locked accounts):
  - unknown email
  - `INACTIVE` status
  - no password set
  - locked account → `423 account-locked`, with `lockedUntil`
- Login rate guard: the 5-strike lockout per account, plus a simple in-memory limit of 20 attempts per minute per IP.
- Every login, logout, lockout and password change writes an `audit_log` row.

## Tests
- Integration tests:
  - invite email contains a working link
  - token expiry
  - refresh rotation, including reuse detection revoking the whole family
  - lockout and unlock after 15 minutes (with a controllable clock)
  - deactivation revokes sessions
- JWKS-verified token round trip.
