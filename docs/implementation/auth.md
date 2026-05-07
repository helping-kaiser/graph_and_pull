# Authentication

Server-side credential management for CoGra. Auth gates
**participation** — writing to the graph — but not **reading**, per
[graph-model.md §1](../primitive/graph-model.md). A frontend can
serve any actor's view of the graph to any reader; account
holders can additionally create edges, author nodes, vote in
governance instances, and join junctions.

This doc specifies what auth does. Concrete library choices and
endpoint shapes belong with the implementation when it's written.

---

## Scope

In scope:

- Account lifecycle (registration via invitation, email
  verification, deletion handoff).
- Credentials (password storage and reset).
- Session tokens (JWT access + Postgres-backed refresh).
- Session listing and revocation.
- Rate limiting on auth endpoints.

Out of scope:

- **Federated identity.** Reconciling identities across instances
  is [open-questions.md Q15](../open-questions.md). When that
  lands it may add a public-key field to User; auth as defined
  here does not store user-owned keys.
- **OIDC provider role.** CoGra is not an identity provider for
  third-party apps. The token model below is the OAuth2
  resource-server shape, which leaves room to add OIDC client
  support (e.g. "log in with Google") later but does not commit
  to issuing identity tokens for other apps.
- **End-to-end content encryption.** Chat E2EE keys are managed
  client-side per [chats.md](../instances/chats.md); the server
  never holds them.
- **MFA.** Not in v1. See "MFA" below.

---

## Server-stored credentials vs. user-owned keys

The server stores **password hashes** — credentials it can verify
but not reverse. These are not "user secrets" in the
cryptographic-key sense; they are server-managed access controls.
The principle that CoGra does not hold user-owned keys (relevant
for Q15 federation reconciliation, where users may eventually hold
identity key pairs client-side) is preserved: hashed credentials
and user-owned cryptographic material are distinct concerns.

---

## Account lifecycle

Three paths reach the same end state — a User node plus the
account's invitation edges in the graph and credentials in
Postgres.

### Invitation acceptance (the default path)

1. **Invite-link click.** The invitee opens the URL. The server
   validates the invitation token (not expired, within any
   usage limits the link carries — see
   [invitations.md](../primitive/invitations.md)) and renders
   the registration form. Invite links are time-gated and
   multi-use; many invitees can register through the same link.
2. **Registration submit.** The invitee submits username, email,
   password, and their outgoing-edge values toward the inviter.
   The server creates a **pending registration record** —
   *not* a User node — and sends a verification email to the
   submitted address.
3. **Email verification.** The invitee clicks the verification
   link. The server atomically:
   - Creates the User node with the registered username.
   - Writes the two invitation edges per
     [invitations.md](../primitive/invitations.md), using the
     pre-committed inviter values from the link and the invitee
     values from the form.
   - Issues the first session (access + refresh token).
   - Deletes the pending registration record.

If verification doesn't happen within 24 hours, the pending
record expires. No User node is created, no edges are written.
The invitation link itself is unaffected — its lifecycle is
independent of any one pending registration.

**Why no User node before verification:** the graph has no
"unverified" state and no concept of partial actorhood. Per the
no-half-state spirit of [layers.md](../primitive/layers.md), nodes
either exist with full standing or they don't. An unverified
holding state would add semantics no other primitive uses and the
ranking math is not designed for.

### First-user genesis bootstrap

A fresh instance has no accounts. The first user creates
themselves directly, without an invitation token. The server
detects "first user" by the User table being empty (and no
pending registrations holding it open). On verification, the
created User is marked as the genesis moderator per
[network.md](../primitive/network.md). Subsequent registrations
require an invitation token.

Email verification still applies — the pending-registration
mechanism is the same.

### Self-service deletion (handoff out)

User-initiated deletion is governed by
[account-deletion.md](../instances/account-deletion.md). Auth's
contribution:

- The deletion confirmation email goes to the verified address on
  file.
- The user can cancel from any authenticated session during the
  7-day grace window.
- When deletion completes, all of the account's refresh tokens
  are revoked. Any outstanding access tokens age out within their
  normal TTL.

---

## Credentials

### Password storage

Passwords are hashed with **Argon2id** using current
OWASP-recommended parameters (re-evaluated periodically as
recommendations evolve). Plaintext is never persisted, never
logged, and never returned by the API.

### Password requirements

- Minimum 12 characters; no maximum.
- No composition rules (forced uppercase / digit / symbol).
  Composition rules reduce entropy by predictable means without
  improving real strength.
- Checked against a known-breach corpus (haveibeenpwned-style
  hash-prefix lookup) at registration and password change.
  Breached passwords are rejected with a message indicating why.

### Password reset

1. User submits their email at the reset endpoint. The server
   responds success **regardless of whether the email exists** —
   no account enumeration via this endpoint.
2. If an account exists for that email, the server generates a
   single-use, short-lived (default 15 min) reset token and
   emails it as a link.
3. The user clicks the link, submits a new password (subject to
   the requirements above). The server validates the token,
   rotates the password hash, and revokes all existing refresh
   tokens for the account — password change is a security event.

---

## Tokens

Two token types per session: a stateless access token and a
stateful refresh token. The split is the standard 2024-era
pattern; rationale for this project below.

### Access token

- **Format.** JWT, signed by the server (Ed25519 recommended for
  size and verification cost).
- **Claims.** `sub` (User UUID), `iat`, `exp`, `jti` binding to
  the issuing refresh token. No role claims — authorization that
  depends on `network_role` (per
  [network.md](../primitive/network.md)) reads the live value
  from the graph at the action site.
- **Lifetime.** 15 minutes (default).
- **Transport.** `Authorization: Bearer <token>` HTTP header on
  every authenticated GraphQL request, validated in Axum
  middleware before reaching resolvers.
- **Revocation.** Not directly revocable within its TTL. Achieved
  through short lifetime + refresh-token revocation: a revoked
  session cannot mint a new access token once the current one
  expires.

### Refresh token

- **Format.** Opaque, cryptographically-random 256-bit value.
  *Not* a JWT.
- **Storage.** Postgres `auth_refresh_tokens` table. The raw
  token is never persisted — only its SHA-256 hash, so a
  database read does not yield usable tokens.
- **Row shape.** `id`, `user_id`, `token_hash`, `created_at`,
  `last_used_at`, `expires_at`, `device_label` (short
  user-readable string for the session list, e.g. derived from
  User-Agent), `revoked_at` (nullable).
- **Lifetime.** 30 days (default), sliding — each successful use
  extends `expires_at` by 30 days from the use time. Inactive
  sessions age out.
- **Rotation.** Every successful refresh consumes the current
  token (sets `revoked_at`) and issues a new one. The client
  must replace its stored refresh token on every refresh. This
  bounds the exposure of a stolen refresh token to a single
  use.
- **Reuse detection.** If a refresh token marked `revoked_at` is
  presented — i.e. someone tried to use a token that was already
  rotated — the server revokes **all** of that user's refresh
  tokens and surfaces a security event on next login. Standard
  refresh-rotation hygiene; signals likely token theft.

### Why split formats

JWT access tokens are stateless and cheap to validate per
request — no database round-trip for read-only authorization.
Opaque refresh tokens are stateful so they can be explicitly
revoked. Refresh requests are infrequent (every ~15 minutes per
active session), so the database round-trip cost is acceptable.

---

## Sessions

A "session" is a row in `auth_refresh_tokens`. The authenticated
user can:

- **List active sessions** — each row's `device_label`,
  `created_at`, `last_used_at`, plus a flag identifying the
  current session.
- **Revoke one session** — sets `revoked_at` on the chosen row.
  The associated access token cannot be invalidated mid-TTL but
  cannot be refreshed past expiry.
- **Revoke all other sessions** — convenience after suspected
  compromise.

Server-initiated revocations:

- Password change or reset → revoke all.
- Account-deletion completion → revoke all.
- Refresh-token reuse detected → revoke all.

---

## Rate limiting

Per-IP and per-account limits on auth endpoints to bound
credential stuffing, registration spam, and reset abuse. The
spec commits to *which* endpoints are limited; specific
thresholds are an implementation choice.

- Login attempts — limited per IP and per account, with
  exponential backoff on consecutive failures.
- Registration / invite-acceptance — limited per IP and per
  invitation token.
- Password-reset requests — limited per IP and per account.
- Verification-email resend — limited per pending-registration
  record.

These match the operational-concern framing in
[moderation.md](../instances/moderation.md): abuse mitigation
lives at the API edge, not in the graph primitives.

---

## MFA

Not in v1. The single-channel email-recovery model is the
standard floor for a community network and avoids the support
burden of TOTP / WebAuthn / recovery-code mechanics during early
operations.

When MFA is added, the natural shape is **TOTP as the second
factor with a WebAuthn upgrade path**, plus single-use recovery
codes (stored hashed) issued at enrollment. MFA becomes a
User-level setting; sessions issued post-MFA-success carry an
`mfa: true` claim that high-stakes mutations (e.g. the
[account-deletion.md](../instances/account-deletion.md)
confirmation, role changes per
[network.md](../primitive/network.md)) can require.

This is forward-looking only. Nothing in v1 should preclude this
upgrade.

---

## Cross-references

- [graph-model.md §1](../primitive/graph-model.md) —
  read-vs-participation distinction.
- [invitations.md](../primitive/invitations.md) — invitation
  primitive that registration consumes.
- [account-deletion.md](../instances/account-deletion.md) —
  consumes session listing and email verification.
- [network.md](../primitive/network.md) — first-user genesis
  bootstrap; `network_role` read at action time.
- [api-spec.md](api-spec.md) — outdated; the auth-stub line in
  §Authentication points here.
- [open-questions.md Q15](../open-questions.md) — federation
  reconciliation; may add user-owned keys later.
