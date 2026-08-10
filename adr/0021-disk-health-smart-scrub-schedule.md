# ADR-0021: Disk health — core polls SMART, metrics and status

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 14; ADR-0008,
  ADR-0017, ADR-0019, ADR-0022; `resolve-control-plane-gaps` D4, D5

## Context

Feature 14 (disk health) lists scrub scheduling and SMART health monitoring, but
no decision existed for how monitoring runs or where results surface. The
declarative model (ADR-0002) and per-controller loops (ADR-0017) constrain the
shape: monitoring is a controller concern, and every subsystem must be observable
(ADR-0008).

## Decision

`core` polls SMART data via `smartctl` on the cadence declared by a `Schedule`
resource (ADR-0022). Each disk exposes Prometheus gauges (ADR-0008) and a
per-disk health/status entry in the disk resource's status object (ADR-0019).
SMART thresholds and scrub cadence are spec-declared, not hardcoded.

## Alternatives considered

- **Scrub-only v1**: ships scheduling but silently omits the health data that
  motivates scrubs.
- **Pass-through scripts**: thin `smartctl` wrappers with no `core` integration,
  no status object, and no metrics — breaks ADR-0008.

## Consequences

- A disk-health controller reads `smartctl` on a `Schedule` (ADR-0022); scrub
  triggers are issued by the same controller via the imperative action path
  (ADR-0002).
- Disk status feeds the status objects (ADR-0019) and metrics (ADR-0008);
  alerting (deferred) can later consume the same gauges.
- Thresholds/cadence are spec resources, so the `storage`/`disk-health` change
  only decides default values, not the mechanism.
