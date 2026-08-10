# ADR-0008: Prometheus metrics for every subsystem

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.8, decision 8

## Context

The NAS manages many subsystems (ZFS, shares, apps, networking, updates). With a
declarative reconciler at the center, operational health is a first-class concern:
every subsystem should be observable and drift between wanted and actual state
surfaced.

## Decision

Every subsystem exposes Prometheus metrics. Each controller publishes counters
and gauges for its slice of the spec store: pool and dataset state, share state,
app health, API request volume and latency, update progress, network state, log
and audit health. No subsystem is unobservable.

## Alternatives considered

- **Event log only**: tells you what happened, not the current state or rates.
- **Per-subsystem dashboards without a standard**: not composable.

## Consequences

- A metrics endpoint ships in `core`; every controller registers its own
  collectors (feature 9, Observability).
- Alerting (deferred, feature-map D6) can later consume the same metrics.
- Metrics names and labels are part of the `contracts` surface.
