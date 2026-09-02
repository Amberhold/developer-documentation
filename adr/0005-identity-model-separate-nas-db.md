# ADR-0005: Identity model (separate NAS DB with optional system/SMB link)

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.5, decision 5;
  ADR-0029

## Context

RBAC identity for the NAS must be cleanly separated from the host's system
accounts, but shares (SMB/NFS) and POSIX ownership need a concrete account to act
on the host.

## Decision

RBAC identity is a NAS-local user database used by the API, UI, and CLI. A
separate, optional link materializes host-level identity only where POSIX/SMB
ownership is required, and is auto-created when a share grants access. The link
has two flavors: the POSIX/SMB ownership link, and a JIT-created OIDC subject
link materialized on first successful OIDC login (ADR-0029).

## Alternatives considered

- **Pure system accounts**: couples RBAC to host accounts and makes lifecycle
  management awkward.
- **Fully separate identity**: share auth becomes disconnected from the users
  the admin manages.

## Consequences

- "Create a share user" implies: NAS user + optional system UID + optional samba
  passdb entry.
- The identity **service** owns the UID allocation ledger — the user DB ↔ UID ↔
  samba mapping — and keeps the materialized system/SMB link consistent with the
  NAS user record. It is a service, not a D1 controller: `status.uid` lives on
  the `User` resource, whose owner is the `User` controller (single status
  writer, D4), so the ledger is injected into the `User` controller (allocate
  on create, reclaim on delete) and into consumers that resolve a linked user's
  UID (the `FileShare` controller). It also owns the JIT-created OIDC subject
  link (ADR-0029), allocating the UID and materializing the host link on first
  federated login. A federated username that collides with an existing
  NAS-local username is rejected at sign-in (no silent re-use, no suffix
  mangling; the shared username namespace stays unambiguous).
- UIDs are also allocated to app stacks (ADR-0004): the identity service owns a
  single UID allocation space shared by users and apps, so POSIX ownership stays
  consistent across shares and app datasets (resolve-control-plane-gaps D9).
- RBAC is enforced at API admission (ADR-0002); host accounts are derived from,
  never the source of, NAS identity.
- **NFS model** (resolve-architecture-gaps D3): NFS has no user authentication.
  Share access is granted to allowed hosts (host/IP-based, expressed via
  `sharenfs`, ADR-0009); the NAS user's UID is the dataset owner, and clients map
  their client-side UID to that NAS UID to see correct file ownership. UID
  alignment is policy: NFSv4 `idmapd` performs domain mapping so mismatched
  client UIDs still resolve to the NAS UID. The identity service allocates
  UIDs so NFS and SMB agree on the same UID for a NAS user.
