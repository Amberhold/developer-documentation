# ADR-0020: Authentication — sessions for the UI, API tokens for CLI/automation, Argon2id

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — implementation-time decisions surfaced by the
  `core` skeleton (token hashing, session-store choice, revocation semantics)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` features 6, 7, 8; ADR-0005,
  ADR-0013, ADR-0018, ADR-0019; `resolve-control-plane-gaps` D1;
  `docs/architecture/02-core-daemon.md` D1–D10

## Context

Identity (ADR-0005) and RBAC roles (ADR-0018) are decided, but nothing specified
how a principal proves identity to the API. ADR-0018 enforces RBAC at API
admission; admission needs an authenticated principal first. The web-UI (ADR-0012)
and the thin CLI client (feature 6) are both API clients, and the audit trail
(ADR-0013) needs an actor context that only a confirmed identity provides.

## Decision

NAS users authenticate to the API with server-side sessions for the web-UI and
long-lived bearer tokens for the CLI and automation. Passwords are hashed with
Argon2id in the NAS user DB. Authentication is the admission gate that RBAC
(ADR-0018) and audit (ADR-0013) build on; every request reaches admission with a
confirmed principal.

## Alternatives considered

- **Sessions only**: browser-friendly but leaves no clean credential for
  CLI/automation.
- **API tokens only**: uniform for clients, but the UI would have to manage a
  token client-side instead of a cookie session.
- **HTTP Basic**: simplest, but transmits credentials on every request over the
  plain-HTTP management-LAN baseline (scope-missing-plane D6).

## Consequences

- The `auth` capability owns login/session/token endpoints; session store, token
  format, expiry, and revocation are designed there, not here.
- RBAC (ADR-0018) evaluates the authenticated principal; audit (ADR-0013) records
  it as the actor on desired-state writes and action calls.
- The CLI (feature 6) uses bearer tokens; the installer bootstrap (feature 12)
  seeds the first admin principal.
- Argon2id hashing lives with the user DB (ADR-0005); password storage and reset
  flow are part of the `auth` design.

## Implementation decisions (core skeleton, 2026-08-27)

The first `core` implementation (`core-skeleton-auth-slice`) pinned the open
items left by this ADR:

- **Token secrets are hashed with Argon2id at rest** (same PHC encoding as
  passwords). A `Token` resource's `spec` holds the hash, never the plaintext;
  `POST /v1/tokens` returns the secret exactly once in the response
  (`TokenCreateResult`), and admission validates bearer tokens by hash lookup.
  The secret embeds a random **lookup id** (`<lookupId>.<random>`); admission
  resolves the token record by that id in O(1) and runs a single Argon2id
  verification — never a linear scan (a scan would be a remote, unauthenticated
  CPU DoS). The lookup id is stripped from API responses; `status.tokenPrefix`
  carries the preview for identification (contract position). Tokens are
  **immutable credentials**: `PUT`/`PATCH /v1/tokens/{id}` are rejected in v1
  (contract 405); the post-issue lifecycle is revocation and expiry
  reconciliation by the owning `auth` controller.
- **Revocation is terminal and storage-enforced.** `DELETE /v1/tokens/{id}`
  (`revokeToken`) marks the token's status `revoked`; the spec store rejects
  any later status write that would overwrite a `revoked` status (the guard
  runs under the store's single serialized writer, closing the window where an
  in-flight reconcile could resurrect a revoked token). Admission rejects
  revoked tokens as unauthenticated. Password verification bounds Argon2id
  parameters (`m ≤ 1 GiB, t ≤ 10, p ≤ 8`) so a planted record cannot force
  unbounded derivation cost, and a client-supplied `passwordHash` is never
  accepted at the API boundary — plaintext `password` is hashed server-side.
- **The session store is in-memory-backed** (change design D1): sessions are
  short-lived and derived from a login event, so an in-memory store is honest.
  The durable artifacts are the NAS user DB and the `Token` records in the spec
  store. Session `status` is persisted with its expiry so restarts surface
  stale/expired sessions and clients re-login gracefully. The session cookie
  value is the persisted `Session` resource id, so `DELETE /v1/sessions/{id}`
  (sign-out) resolves what the client holds.
- **Revocation semantics**: `DELETE /v1/tokens/{id}` (`revokeToken`) marks the
  token's status `revoked`; the stored hash remains but admission rejects the
  secret as unauthenticated from that point (see terminal-revocation guard
  above). Sessions are revoked by `DELETE /v1/sessions/{id}` (sign-out), which
  destroys the in-memory session and removes the session resource; expired
  tokens and sessions are reconciled by the `auth` controller (expiry observed
  from the resource spec/status), matching ADR-0017 fail-and-retry.
- **Session resource ids are never the cookie value**: the persisted `Session`
  resource gets a store-assigned id; the cookie ↔ resource mapping lives only
  in the in-memory session store, and `GET /v1/sessions` lists **the caller's**
  sessions only (a session resource id must never be enumerable by other
  principals — presenting it as the cookie would hijack the session).
- **Login throttling and observability**: `POST /v1/sessions` is rate-limited
  per client address, unknown usernames pay the Argon2id cost against a dummy
  hash (no timing-based user enumeration), and every attempt emits
  `amberhold.auth.login.attempts{source,outcome}`.
- **Bootstrap principal**: the installer seeds the first admin user (username,
  Argon2id password hash, `admin` role) before controllers start; the auth
  controller converges it to `Ready` on its first pass (ADR-0031 D6: API-last
  means the first admin request sees a converged admission gate).
