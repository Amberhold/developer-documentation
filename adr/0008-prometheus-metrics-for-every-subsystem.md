# ADR-0008: Prometheus metrics for every subsystem

- Status: accepted (amended 2026-08-11: Prometheus is the default *exporter*, not
  the instrumentation model)
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.8, decision 8;
  ADR-0019, ADR-0025, ADR-0026, ADR-0027, feature 9

## Context

The NAS manages many subsystems (ZFS, shares, apps, networking, updates). With a
declarative reconciler at the center, operational health is a first-class concern:
every subsystem should be observable and drift between wanted and actual state
surfaced.

Originally this ADR tied observability to Prometheus as the instrumentation model
(hand-rolled per-controller collectors). That blocks a unified off-box
observability story: metrics, traces, and logs cannot flow to a single backend,
and every future signal pipeline requires re-instrumentation.

## Decision

Every subsystem exposes metrics. **Instrumentation is the OpenTelemetry SDK**:
each controller publishes counters, gauges, and histograms through an OTEL meter
scoped to its subsystem. **Prometheus is the always-on default exporter** — the
OTEL MeterProvider's Prometheus exporter serves the existing `/metrics` endpoint
in scrape-only hosting. No subsystem is unobservable.

Metrics instrumentation changes from hand-rolled Prometheus collectors to OTEL
meters; metric names move to the `amberhold.*` catalog with a documented
Prometheus translation (the `contracts` surface, enforced by OTEL Views). This is
a breaking change to the instrumentation layer, additive to the `/metrics` wire
format during migration (each old collector metric appears in the catalog before
its removal).

## Alternatives considered

- **Event log only**: tells you what happened, not the current state or rates.
- **Per-subsystem dashboards without a standard**: not composable.
- **Prometheus as the instrumentation model**: the original decision; rejects a
  unified multi-signal backend story (see Context).

## Consequences

- A metrics endpoint ships in `core`; the OTEL MeterProvider with its Prometheus
  exporter backs `/metrics` (feature 9, Observability).
- OTEL metric export to OTLP targets is purely additive and never replaces the
  always-on Prometheus endpoint (ADR-0025).
- Alerting (deferred, scope-missing-plane D6) can later consume the same metrics.
- Metric names, labels, and histogram buckets are part of the `contracts` surface,
  declared in the metric catalog and enforced by OTEL Views.
- Hosting is scrape-only (resolve-architecture-gaps D9): `core` exposes
  `/metrics` and an external Prometheus scrapes it. There is no embedded
  Prometheus server on the appliance (no stateful storage on the RO-root device);
  on-device dashboards, if any, are a `web-ui`/`observability` question, not a
  hosting one.
- Traces and logs flow through the same OTEL SDK (ADR-0026, ADR-0027); OTEL
  dependencies are version-pinned into the image per ADR-0001.
