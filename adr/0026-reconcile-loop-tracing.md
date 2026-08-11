# ADR-0026: Reconcile-loop tracing

- Status: accepted
- Date: 2026-08-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, §3.8, feature 9;
  ADR-0008, ADR-0017, ADR-0025

## Context

Per-controller reconcile loops over a shared event source (ADR-0017) are the
core's central activity. When the desired state and the system drift, the first
question is always *which* controller failed to converge *which* resource.
ADR-0008 (amended) makes OpenTelemetry the instrumentation layer; tracing the
reconcile loop turns the event source into an observable unit of work without a
separate trace infrastructure.

## Decision

One root `reconcile` span is created per reconcile pass over the event source,
carrying the reconcile iteration identity. Each controller's reconcile step
creates a child span under the root span, attributed to its subsystem. Controller
child spans carry attributes identifying the resource being reconciled: its
resource identifier, its wanted version, and its actual version.

Sampling uses a parent-based sampler with a per-subsystem sampling rate and a
global default rate of zero. Traces are only emitted and exported when an OTLP
target is configured (ADR-0025); with sampling at zero or no target, no trace
data is produced at all.

## Alternatives considered

- **No tracing (metrics only)**: metrics tell you a controller is failing, not
  the per-resource causal path through a reconcile pass.
- **Explicit logging of reconcile steps**: unbounded volume, not structured,
  cannot carry span parentage.

## Consequences

- The reconciler creates a root span per pass and child spans per controller
  (ADR-0017); span attributes follow the declared conventions.
- Tracing is off by default and produces no data until a target and a non-zero
  sampling rate are configured — safe on the always-on appliance.
- Trace data is exported only to configured OTLP targets; there is no on-device
  trace store (ADR-0008 hosting).
- Sampling rates are configurable globally and per subsystem via the `Telemetry`
  resource (ADR-0025).
