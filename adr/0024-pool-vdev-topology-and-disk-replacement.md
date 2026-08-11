# ADR-0024: Pool/vdev topology — v1 vdev types, spares, and disk replacement semantics

- Status: accepted
- Date: 2026-08-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 11; ADR-0002,
  ADR-0007, ADR-0019, ADR-0021

## Context

Feature 11 (pool storage) covers disk inventory, pool/vdev creation, and disk
replacement, but no decision records which vdev topologies v1 supports, how
spares are handled, or the replacement flow. The installer assigns pool layout at
install (ADR-0007) and replacement is an imperative action (ADR-0002), so the
supported topology set must be decided up front: it bounds both the installer UI
and the storage controller.

## Decision

v1 supports a single pool whose vdev layout is chosen at install from a fixed,
minimal set:

- **Vdev types**: single-disk, mirror (2–n), and raidz1/raidz2. dRAID, raidz3,
  and striped vdevs are out of scope for v1. A pool may combine multiple vdevs
  of these types.
- **Spares**: warm/hot spares are supported; spare membership is declared in the
  spec store and the pool is assembled with the spare attached. Failed-disk
  replacement substitutes a spare automatically when one is attached; otherwise
  replacement is manual.
- **Replacement flow** (imperative action, ADR-0002): the admin issues
  `replace disk`, the storage controller runs `zpool replace` (or online-replaces
  a spare), resilvering proceeds, and the `disks` resource status (ADR-0019)
  plus metrics (ADR-0021) track progress. The spec store records the pool layout
  as desired state; replacement mutates actual state, not desired state.
- **Layout mutation post-install** is out of scope: a vdev or pool layout change
  after install is migration work (ADR-0007 consequence), not a routine action.

## Alternatives considered

- **Full topology freedom (any vdev type, dRAID, striped, expandable)**: maximal
  flexibility, but expands installer and controller surface well beyond v1 needs.
- **Single mirror-only pool**: smallest possible, but ignores single-disk and
  raidz installs that dominate small-NAS hardware.

## Consequences

- The installer offers the fixed topology set at the pool-creation step
  (feature 12) and records the chosen layout in the spec store (ADR-0007).
- The storage controller implements `zpool create/replace`, spare attach, and
  resilver progress reporting for exactly these types; raidz3/dRAID/expand are
  deferred features, not silent gaps.
- Disk health (ADR-0021) and replacement share the `disks` resource status; a
  failed disk with no spare surfaces as degraded pool status until manual
  replacement.
- The storage change decides only v1 default pool/vdev parameters (e.g. ashift,
  record size, quota defaults); the supported topology set is fixed here.
