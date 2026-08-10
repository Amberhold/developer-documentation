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
containerd). Container volumes are ZFS-backed — a dataset (or zvol) per volume —
so app data participates in snapshots and replication.

## Alternatives considered

- **Reduced/own compose schema**: fragments ecosystem compatibility.
- **Plain-directory volumes**: no snapshots or replication for app data.

## Consequences

- Images and compose state live on a configured app dataset (see decision 7,
  configurable storage layout).
- `nerdctl`/compose translation is a moving target; containerd/nerdctl versions
  are pinned in the image (ADR-0001) to bound that risk.
- The Apps controller must map compose volumes to ZFS datasets/zvols.
