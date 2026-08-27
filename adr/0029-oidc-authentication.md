# ADR-0029: OIDC authentication for the Web-UI

- Status: accepted
- Date: 2026-08-16
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` features 6, 7, 10, §6;
  ADR-0005, ADR-0018, ADR-0020, ADR-0028

## Context

ADR-0020 commits authentication to NAS-local users: Argon2id passwords, sessions
for the Web-UI, bearer tokens for CLI/automation. OIDC is a common
enterprise/NAS expectation, and its authorization-code flow cannot work over a
plain-HTTP management LAN — the OIDC `redirect_uri` must be HTTPS. TLS is
therefore a prerequisite and is addressed by ADR-0028.

## Decision

The Web-UI supports authentication against exactly one external OIDC identity
provider at a time. Sign-in uses the OIDC **authorization-code flow with PKCE**;
the provider configuration carries discovery metadata, client credentials, and a
claim-to-role mapping.

Scope is **UI-only**: CLI and automation keep long-lived bearer tokens (ADR-0020);
there is no OIDC device-flow support in v1.

Federated principals get their RBAC roles from the IdP `groups` claims via the
claim-to-role mapping, **not** from role membership in the NAS user DB. This is an
explicit override of ADR-0018 for the federated principal class only; NAS-local
principals (including the installer-seeded first admin, feature 12) keep
DB-stored role membership. A federated principal whose claims map to no v1 role is
admitted with the read-only capability only.

On first successful authentication, the system **JIT-creates a NAS user record**
linked to the IdP subject, allocates the NAS UID, and materializes the POSIX/SMB
host link per the identity model (ADR-0005). Subsequent logins reuse the existing
record.

Both OIDC and password sign-in establish the same server-side session; admission
(ADR-0018) and audit (ADR-0016) consume a confirmed principal regardless of
source. When the IdP is unreachable, NAS-local principals continue to
authenticate with passwords and retain their DB-stored roles — the break-glass
path.

## Alternatives considered

- **Multi-IdP support**: richer, but more contract and configuration surface than
  v1 needs.
- **OIDC for CLI/automation (device flow)**: convenient for scripting, but bearer
  tokens already cover the use case and keep the CLI path uniform.
- **DB role membership for everyone**: consistent with ADR-0018 but forces double
  bookkeeping and makes the IdP a non-authority for access.
- **Pure claims-based roles for everyone**: breaks the break-glass guarantee on
  IdP outage.
- **Admin pre-provisioning of federated users**: more control, but worse UX than
  JIT account creation.
- **TLS only on auth endpoints**: rejected in ADR-0028; the UI serves the API from
  the same origin and IdP redirect URIs are per-origin.

## Consequences

- The `auth` capability (feature 6) gains an OIDC sign-in option for the Web-UI;
  the existing password path and bearer-token path are unchanged.
- ADR-0018 carries an override note scoped to federated principals; ADR-0005
  gains a second, JIT-created OIDC subject-link flavor.
- TLS on the management plane is required (ADR-0028); the IdP's registered
  `redirect_uri` is HTTPS.
- The OIDC provider resource shape and the claim-to-role mapping syntax are
  part of the `contracts` surface (ADR-0019) and are deferred to a later change.
- The RBAC capability map (ADR-0018) is unchanged; roles still come from the same
  v1 role set.
- IdP outage affects only federated sign-in; NAS-local sign-in and the seeded
  first admin are unaffected.
