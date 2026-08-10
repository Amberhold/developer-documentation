# ADR-0005: Identity model (separate NAS DB with optional system/SMB link)

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.5, decision 5
- Amendments: NFS identity model recorded — host-based access with UID alignment
  to the NAS user (the SMB path uses the samba passdb link).
- Amendments: NFS identity mechanism — UID alignment is policy; NFSv4 `idmapd`
  performs domain mapping so mismatched client UIDs still resolve to the NAS UID.
- Amendments: app identity — the identity controller allocates a dedicated UID
  per app stack (alongside user and share UIDs), so app datasets have derived
  ownership (resolve-control-plane-gaps D9).

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
- UIDs are also allocated to app stacks (ADR-0004): the controller owns a single
  UID allocation space shared by users and apps, so POSIX ownership stays
  consistent across shares and app datasets.
- RBAC is enforced at API admission (ADR-0002); host accounts are derived from,
  never the source of, NAS identity.
- **NFS model**: NFS has no user authentication. Share access is granted to
  allowed hosts (host/IP-based, expressed via `sharenfs`, ADR-0009); the NAS
  user's UID is the dataset owner, and clients map their client-side UID to that
  NAS UID to see correct file ownership. The identity controller allocates UIDs
  so NFS and SMB agree on the same UID for a NAS user.
