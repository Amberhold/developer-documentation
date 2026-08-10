# ADR-0018: RBAC — fixed roles with a capability map

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 7, §6; ADR-0002,
  ADR-0005

## Context

Feature 7 (RBAC + users) scopes roles per capability but the taxonomy was open
(§6). RBAC is enforced at API admission (ADR-0002); identity is a NAS-local DB
(ADR-0005). The question is which roles and capabilities v1 ships.

## Decision

v1 ships fixed roles with a capability map: `admin`, `storage-admin`,
`share-admin`, `app-admin`, `auditor`, and `read-only`. Capabilities map onto the
feature map's features (pools, shares, apps, users, updates, metrics). RBAC
applies at API admission; share grants additionally materialize the system/SMB
link per ADR-0005.

## Alternatives considered

- **Per-user capabilities, no roles**: maximally flexible but harder to reason
  about and more UI/CLI surface.
- **Admin + read-only only**: smallest possible, but forces a migration once the
  split roles are wanted.

## Consequences

- The role/capability taxonomy is part of the `contracts` surface (ADR-0019).
- Admission is the only enforcement point; the UI never gates capability access
  itself.
- Role membership is stored in the NAS user DB (ADR-0005).
