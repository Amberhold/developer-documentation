# ADR-0020: Authentication — sessions for the UI, API tokens for CLI/automation, Argon2id

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` features 6, 7, 8; ADR-0005,
  ADR-0016, ADR-0018, ADR-0019; `resolve-control-plane-gaps` D1

## Context

Identity (ADR-0005) and RBAC roles (ADR-0018) are decided, but nothing specified
how a principal proves identity to the API. ADR-0018 enforces RBAC at API
admission; admission needs an authenticated principal first. The web-UI (ADR-0012)
and the thin CLI client (feature 6) are both API clients, and the audit trail
(ADR-0016) needs an actor context that only a confirmed identity provides.

## Decision

NAS users authenticate to the API with server-side sessions for the web-UI and
long-lived bearer tokens for the CLI and automation. Passwords are hashed with
Argon2id in the NAS user DB. Authentication is the admission gate that RBAC
(ADR-0018) and audit (ADR-0016) build on; every request reaches admission with a
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
- RBAC (ADR-0018) evaluates the authenticated principal; audit (ADR-0016) records
  it as the actor on desired-state writes and action calls.
- The CLI (feature 6) uses bearer tokens; the installer bootstrap (feature 12)
  seeds the first admin principal.
- Argon2id hashing lives with the user DB (ADR-0005); password storage and reset
  flow are part of the `auth` design.
