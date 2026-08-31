# ADR-0004: ZFS-backed container volumes with full docker-compose

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.4, decision 4

## Context

The NAS runs app workloads on containerd. Container volumes hold app data that
should participate in the ZFS snapshot/replication story, and users expect their
existing docker-compose files to work.

## Decision

Support full docker-compose semantics on containerd (via `nerdctl compose` →
containerd). Container volumes are ZFS-backed — a dataset per volume by default
(scope-missing-plane D3) — so app data participates in snapshots and
replication. A zvol is an explicit escape hatch where an app requires block
storage semantics (e.g. databases).

## Alternatives considered

- **Reduced/own compose schema**: fragments ecosystem compatibility.
- **Plain-directory volumes**: no snapshots or replication for app data.
- **zvol-per-volume default**: block semantics for everything, but needs a
  filesystem on top and alignment care for most workloads.

## Consequences

- Images and compose state live on a configured app dataset (see decision 7,
  configurable storage layout); the app role is opt-in — present only when the
  apps feature is used. App data, images, and compose state all stay on ZFS and
  are unaffected by OS-disk loss (ADR-0011).
- `nerdctl`/compose translation is a moving target; containerd/nerdctl versions
  are pinned in the image (ADR-0001) to bound that risk.
- The Apps controller must map compose volumes to ZFS datasets by default, with
  the zvol escape hatch documented per app (ADR-0010 governs concurrent block
  exposure).
- Each app stack runs as a dedicated UID allocated by the identity service
  (ADR-0005; resolve-control-plane-gaps D9); that UID owns the stack's
  dataset(s), so app data ownership is derived identity, never root by default.
