# OIDC Controller — Identity Provider Singleton, PKCE Redirect Flow, and JIT Federated Identity

> Discovery-phase design. Authored from the `oidc-authentication` openspec
> change. The decisions D-O1–D-O6 here fix how the `OIDCProviderController`
> reconciles the singleton `oidc` resource (ADR-0029) into a validated config
> snapshot the Web-UI sign-in flow resolves per use, how the authorization-code
> + PKCE redirect flow completes, and how federated principals get JIT NAS
> identity and claim-derived roles. ADR-0029 fixes the *decision* (exactly one
> IdP, authorization-code + PKCE, claims-as-roles, JIT NAS account, UI-only);
> this document fixes the *internal mechanics*, built on the framework-first
> runtime in `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031). The
> feature map (features 6 and 7, identity + RBAC) is in
> `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

The contract already declares the OIDC surface (`source: [password, oidc]`,
`principalSource: [local, oidc]`, the reserved `oidc` login source in
`internal/auth/actions.go`), but the auth slice serves only NAS-local Argon2id
password login. This slice lands OIDC sign-in for the Web-UI: the
`OIDCProvider` singleton + controller, the public redirect flow
(`/v1/oidc/login` + `/v1/oidc/callback`), JIT federated user creation through
the shared identity allocator (ADR-0005), claim-to-role mapping (ADR-0029
override of ADR-0018), and `principalSource` emission in `User`/`Session`
status.

## 2. Goals / Non-Goals

**Goals:**
- Land the `OIDCProvider` singleton resource end-to-end: controller,
  `/v1/oidc` API (GET/PUT/PATCH), RBAC (`oidc:read`/`oidc:write`).
- Implement the authorization-code + PKCE browser redirect flow with a
  one-time, short-lived in-memory state store.
- JIT-create federated NAS users linked by IdP subject, with UIDs from the
  shared allocator (ADR-0005), reused on later logins.
- Derive federated RBAC roles from IdP `groups` claims (ADR-0029 override of
  ADR-0018); no match admits read-only.
- Wire `principalSource` (`local`/`oidc`) into `User` and `Session` status.
- Keep NAS-local password sign-in and the seeded first admin unchanged
  (break-glass on IdP outage).

**Non-Goals:**
- No OIDC device flow / CLI OIDC (bearer tokens remain the CLI path,
  ADR-0020).
- No multi-IdP support (exactly one provider, ADR-0029).
- No directory sync / AD-LDAP join (feature-map deferred area).
- No DB role membership for federated principals (ADR-0029 override).
- No admin pre-provisioning of federated users (JIT only).

## 3. Decisions

### D-O1: `OIDCProvider` singleton modeled on telemetry/certificates

Kind `OIDCProvider`, fixed singleton id `default`, paths `GET/PUT/PATCH
/v1/oidc`. Follows `internal/telemetry/controller.go` (singleton id `default`,
empty-spec defaults) and the `/v1/certificates`/`/v1/telemetry` route +
capability gating in `internal/app/app.go`. **Empty spec = disabled;
non-empty spec = enabled.** The controller validates the issuer (discovery
metadata), parses the claim-to-role map, and publishes an immutable provider
config snapshot to the redirect handler — the same resolve-per-use intent as
the telemetry provider handle (D8), but here the "host" is a validated config
struct rather than live exporters. A failed discovery keeps the previous
validated snapshot in service and reports `error`.

### D-O2: clientSecret in the spec store, write-only (backup pattern)

The client secret is stored in the spec store and stripped on read/status,
following `internal/backup/controller.go` + `internal/certificates/
controller.go`: `sanitizeWanted` removes the write-only field from the status
`Wanted` echo, and the API's write-only filter (`writeOnlySpecKeys` in
`internal/api/server.go`) never returns it. The secret is consumed by the flow
only at token-exchange time, read from the spec store — it never enters the
published `Config` snapshot. Rationale over an OS-disk keyfile: the secret is a
per-provider, rotate-able credential managed through the spec like the backup
restic password, not a boot-time OS secret like LUKS keys. No host-file writes
on the read-only root.

### D-O3: Redirect flow as public routes, distinct from the JSON login handler

The authorization-code flow cannot reuse `POST /v1/sessions` (JSON body,
rate-limited, no browser redirect). New public routes:

```
GET  /v1/oidc/login     -> 302 Location: <IdP auth>?...code_challenge=...
GET  /v1/oidc/callback  -> verify state -> exchange code -> validate id_token
                           -> JIT user -> session -> Set-Cookie -> 302 UI
```

An in-memory PKCE store holds one-time, short-lived (TTL ~5 min) `state` +
`code_verifier` entries keyed by a random state, cleaned on use/expiry — the
same in-memory short-lived pattern as `internal/auth/session.go`'s
`SessionStore`. The callback consumes the state **before** any token exchange:
a replayed or stale state is rejected. The post-login redirect target is a
fixed/validated UI origin (default `/`), never a client-supplied URL — no open
redirect. The routes are registered public in the admission route map (exact
match wins over the `/v1/oidc` capability prefix), and the login route is
rate-limited per client address like `POST /v1/sessions` (M7).

### D-O4: JIT federated identity reuses the identity allocator

On first login, the flow creates a `User` record (via the `UserController`
path) linked by IdP subject: the spec carries `oidcSubject` (the IdP `sub`) and
`principalSource: oidc`, and the `UserController.Reconcile` pass allocates the
UID through `identity.Allocator.Allocate` and reports
`status.principalSource = oidc` — the same convergence path as a local user.
Reuse on later logins via `Service.UserByUsername`; the existing record is
returned without a new UID allocation. The `source` on the created `Session` is
`oidc`.

### D-O5: Reject on username collision

If the IdP `preferred_username` collides with an existing NAS-local username
(a record with a password hash and no `oidcSubject` link), the federated login
is rejected with a clear error (`ErrUsernameCollision`) — no silent re-use, no
suffix mangling. A federated username linked to a *different* subject is also
rejected (`ErrSubjectMismatch`): identity confusion is never silently
re-linked. This keeps the shared username namespace unambiguous and surfaces
the conflict to the operator.

### D-O6: principalSource wired into User/Session status

`User.status.principalSource` and `Session.status.principalSource` are emitted
(`local` for password login, `oidc` for federated), closing the contract/core
gap. The User controller derives the field from the controller-owned spec
marker (default `local`); the Session controller derives it from the login
`source` (`password -> local`, `oidc -> oidc`). Federated roles come from IdP
`groups` claims via the declarative `claimRoles` map on the provider resource,
**not** from DB role membership (ADR-0029 override of ADR-0018): the flow puts
the claim-derived roles into the Session at creation, and admission consumes
them unchanged. No mapped role admits the `read-only` role only.

## 4. Reconciliation

The `OIDCProviderController` owns the singleton `oidc` kind (D1) and converges
the desired spec against the redirect flow's `Provider` handle (task 2.2).
Each pass:

- **Parse** the spec (issuer, clientId, write-only clientSecret, redirectUri,
  scopes, claimRoles); an empty spec is a disabled provider (D-O1).
- **Validate** at the boundary: issuer and HTTPS redirect URI well-formedness,
  claim-to-role values naming fixed v1 roles (ADR-0018); the controller
  additionally validates issuer **discovery** via the IdP host facade
  (D-O1) — the network boundary is the controller's, not the API's.
- **Swap** an immutable `Config` snapshot (validated issuer, clientId,
  redirectUri, scopes, claimRoles) into the handle atomically; a disabled spec
  swaps in `nil` so the flow refuses sign-in; a failed discovery swaps nothing
  (the previous snapshot stays in service).
- **Status**: write `status.state` (`disabled`/`ready`/`error`) and the
  validated `issuer` through the store's serialized path (D4), with the
  client secret stripped from `wanted` (D-O2). An unchanged spec (canonical
  JSON compare) is an idempotent no-op (D9).

## 5. The redirect flow

`oidc.Flow` resolves the current `Config` snapshot per use (D8) and reads the
client secret from the spec store only at exchange time (D-O2). `GET
/v1/oidc/login` builds the IdP authorization URL with an S256 PKCE challenge
and a one-time state (D-O3). `GET /v1/oidc/callback` consumes the state,
exchanges the code with the code verifier, validates the ID token (issuer,
audience, expiry, signature) via the go-oidc-backed host facade, JIT-creates or
reuses the federated user (D-O4/D-O5), derives roles from the `groups` claims
(D-O6), establishes a server-side session with `source=oidc`, sets the session
cookie, and 302-redirects to the fixed UI origin. Sign-in attempts flow into
`amberhold.auth.login.attempts` with `source=oidc` (the catalog label already
generalizes).

## 6. Status and observability

The controller writes the declared `OIDCProviderStatus` shape (`state`
disabled/ready/error, validated `issuer`, `error` reason) through the store
(D4). Sign-in outcomes (success/failure, `source=oidc`) are emitted to the
declared `amberhold.auth.login.attempts` counter via the login hook; no
unregistered metric name is emitted (ADR-0008). The write-only client secret
never appears in status or any API response (D-O2, the API's write-only
filter + the controller's `sanitizeWanted`).

## 7. Risk / trade-offs

- **IdP outage** → Only federated sign-in is affected; NAS-local password
  sign-in and the seeded first admin remain the break-glass path (ADR-0029).
  Mitigation: no coupling between the OIDC flow and the local path; admission
  accepts both.
- **TLS required** → The OIDC redirect URI must be HTTPS (ADR-0028).
  Mitigation: the flow is only reachable behind the certificates-managed TLS
  terminator; the boundary validation rejects non-HTTPS redirect URIs.
- **PKCE state replay/CSRF** → Cryptographically random state, one-time
  consumption, short TTL, verified against the store before any token
  exchange (D-O3).
- **Open redirect via callback** → The post-login redirect target is a
  fixed/validated UI origin, never a client-supplied arbitrary URL (D-O3).
- **JWT validation surface** → Issuer, audience, expiry, and signature are
  validated against IdP discovery keys via the pinned go-oidc client library
  (v3.14.1, ADR-0001); unknown issuers are rejected by the verifier; an empty
  subject is rejected. Caveat: the `groups` claims must be emitted in the **ID
  token** by the IdP (many IdPs put them only in the access token), otherwise
  every federated principal silently lands at read-only — the mapping never
  matches an absent claim.
- **Federated username collisions / identity confusion** → Rejected with clear
  errors (D-O5), never silently resolved.
- **`principalSource` drift between contract and core** → Single change lands
  both the contract and the emission together (D-O6).

## 8. Migration plan

No data migration: OIDC is additive to the auth slice. The `OIDCProvider`
default is an empty (disabled) spec; existing NAS-local users, sessions, and
tokens are unaffected. Rollback is by reverting the `OIDCProvider` spec to
empty, which disables federated sign-in and returns to password-only. Binary
rollback is the A/B slot path (future update controller), out of scope here.

## 9. Open questions (implementation-time)

- Exact OIDC client library version pin (resolved at implementation:
  `github.com/coreos/go-oidc/v3` v3.14.1 + `golang.org/x/oauth2` v0.36.0, the
  current stable releases; recorded in `core/go.mod` per ADR-0001).
- IdP `preferred_username` fallback when absent (resolved: the IdP `sub` is
  used as the NAS username — unique per subject, collision-checked like any
  federated username).