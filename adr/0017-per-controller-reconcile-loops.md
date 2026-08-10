# ADR-0017: Per-controller reconcile loops over a shared event source

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, §4, §6; ADR-0002

## Context

The feature map (§6) left reconcile-loop granularity open: a single global pass
over all wanted vs actual state, or one loop per controller. The skeleton already
lists a controller per subsystem (ZFS, SMB, NFS, NVMe-oF, Apps, Identity, Update).

## Decision

Each controller runs its own reconcile loop over its slice of the spec store,
sharing a single event source (spec-store writes). A controller observes only the
resources it owns; failures are contained to that controller. There is no declared
dependency ordering: when a controller needs a prerequisite owned elsewhere (e.g.
an identity link before a share bind, or a network address before a share binds),
it fails and retries on its next loop; reconciliation is idempotent
(resolve-control-plane-gaps D12).

## Alternatives considered

- **Single global pass**: simpler sequencing, but any slow or failing controller
  blocks all others and couples subsystem cadences.

## Consequences

- Controllers are independent units, matching the skeleton's controller list.
- Spec-store writes fan out to interested controllers; `core` aggregates status
  from all controllers for the API.
- Each controller registers its own Prometheus collectors (ADR-0008).
