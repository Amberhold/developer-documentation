# ADR-0005: Identity model (separate NAS DB with optional system/SMB link)

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.5, decision 5

## Context

RBAC identity for the NAS must be cleanly separated from the host's system
accounts, but shares (SMB/NFS) and POSIX ownership need a concrete account to act
on the host.

## Decision

RBAC identity is a NAS-local user database used by the API, UI, and CLI. A
separate, optional link materializes host-level identity only where POSIX/SMB
ownership is required, and is auto-created when a share grants access.

## Alternatives considered

- **Pure system accounts**: couples RBAC to host accounts and makes lifecycle
  management awkward.
- **Fully separate identity**: share auth becomes disconnected from the users
  the admin manages.

## Consequences

- "Create a share user" implies: NAS user + optional system UID + optional samba
  passdb entry.
- The Identity controller owns the user DB ↔ UID ↔ samba mapping and keeps the
  materialized system/SMB link consistent with the NAS user record.
- RBAC is enforced at API admission (ADR-0002); host accounts are derived from,
  never the source of, NAS identity.
