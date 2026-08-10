# ADR-0013: Spec store on a persistent OS-disk partition

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, §6; ADR-0001, ADR-0002,
  ADR-0011

## Context

The spec store is the source of truth (ADR-0002) and must be versioned/snapshotted
to support the revert half of composite rollback. ADR-0011 dedicates an OS disk to
the A/B slots and bootloader, but does not say where the spec store lives — it was
left open in the feature map (§6). Placement decides boot ordering (does `core`
need a pool imported before it can read its own desired state?) and the failure
story for both the OS disk and the pools.

## Decision

The spec store lives on a persistent config partition on the dedicated OS disk,
outside the A/B slots. Boot reads the spec before any pool import: `core` starts
reconciling from the spec with pools absent, then imports pools as they become
available. Updates swap slots and never touch the spec store.

The spec store is versioned: it records a schema version, and `core` migrates the
store forward on read. Rollback restores a prior versioned snapshot rather than
reverse-migrating, so an older `core` never reads a newer schema
(resolve-control-plane-gaps D7).

## Alternatives considered

- **Spec store on the configurable `config` dataset (a pool)**: fits ADR-0007 roles,
  but makes boot depend on pool import and couples the source of truth to pool
  availability.
- **Spec store on a user data pool**: couples config to user data and its failure modes.

## Consequences

- Boot ordering is independent of pools; a missing/unimportable pool degrades
  reconcile scope rather than blocking `core` startup.
- Update (ADR-0006) never touches the spec store — slot swap and desired state are
  cleanly separated.
- OS-disk failure means config loss; recovery is reinstall + `zpool import`
  (ADR-0011 recovery path). Accepted: data pools are unaffected.
- OS-disk sizing must reserve headroom for spec-store versioning/snapshots; sizing
  is an install-time check (ADR-0011 amendment).
- The store's schema version makes update/rollback (ADR-0006) explicit: a new
  `core` migrates forward; a rolled-back `core` restores a compatible prior
  snapshot. Downgrade across a schema change is never attempted.
