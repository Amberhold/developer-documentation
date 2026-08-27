# ADR-0008: Observability — OpenTelemetry instrumentation with a single `Telemetry` resource

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-11 — Prometheus is the default *exporter*, not the
  instrumentation model
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 9, §3.8;
  ADR-0001, ADR-0002, ADR-0013, ADR-0017, ADR-0018, ADR-0019

> Consolidates the former ADR-0008 (Prometheus metrics), ADR-0025 (telemetry
> resource / OTLP export), ADR-0026 (reconcile-loop tracing), and ADR-0027
> (OTEL log export with journald). One observability substrate decision; the
> alternatives and consequences of each are retained below.

## Context

The NAS manages many subsystems (ZFS, shares, apps, networking, updates). With a
declarative reconciler at the center, operational health is a first-class concern:
every subsystem should be observable and drift between wanted and actual state
surfaced. Originally observability was tied to Prometheus as the instrumentation
model (hand-rolled per-controller collectors). That blocks a unified off-box
observability story: metrics, traces, and logs cannot flow to a single backend,
and every future signal pipeline requires re-instrumentation. Amberhold needs an
off-box story — metrics, traces, and logs reaching an external backend — which
requires configuration, expressed through the declarative API model and reconciled
without daemon restarts (ADR-0002).

## Decision

**OpenTelemetry is the instrumentation substrate for all three signals.** Each
controller publishes counters, gauges, and histograms through an OTEL meter scoped
to its subsystem; a logger scoped to its subsystem; and reconcile passes are
traced. **Prometheus is the always-on default exporter** — the OTEL
MeterProvider's Prometheus exporter serves the existing `/metrics` endpoint in
scrape-only hosting. No subsystem is unobservable.

### The `Telemetry` resource

A singleton `Telemetry` resource in the API resource model (ADR-0019) carries
desired-state spec and status, reconciled by a telemetry controller (ADR-0017).
The controller owns the lifecycle of the running MeterProvider, TracerProvider,
and LoggerProvider; controllers reference the providers and never build their own.
Writing desired state converges the running providers without a daemon restart.

- A **global default block** and an **overrides map keyed by subsystem id**.
  Overrides carry only the fields they change (traces sampling rate, log level,
  metric overrides) and take precedence over the global default; removing an
  override falls back to the default.
- An **OTLP target list**. Each target declares an endpoint, a protocol, and a
  TLS mode of `none`, `tls`, or `mtls` with optional CA/certificate/key
  references. TLS is optional per target; v1 defaults to `none` for the plain-HTTP
  LAN posture, with the configuration surface available from day one.

Only in-process exporters exist: Prometheus (always on), OTLP (one per configured
target, wired for whichever signals the target enables), and journald (log export
only). No OTEL Collector runs on the appliance. OTLP is purely additive and never
replaces the Prometheus endpoint.

### Metrics

Metrics instrumentation changes from hand-rolled Prometheus collectors to OTEL
meters; metric names move to the `amberhold.*` catalog with a documented Prometheus
translation (the `contracts` surface, enforced by OTEL Views). This is a breaking
change to the instrumentation layer, additive to the `/metrics` wire format during
migration (each old collector metric appears in the catalog before its removal).

### Reconcile-loop tracing

One root `reconcile` span is created per reconcile pass over the event source
(ADR-0017), carrying the reconcile iteration identity. Each controller's reconcile
step creates a child span under the root, attributed to its subsystem, carrying
the resource identifier, wanted version, and actual version. Sampling uses a
parent-based sampler with a per-subsystem sampling rate and a global default of
zero — traces are only emitted when an OTLP target is configured (i.e. off by
default, producing no data on the always-on appliance).

### Logs with journald as the durable store

Every core component emits its logs through the OTEL logging pipeline with a
logger scoped to its subsystem. The LoggerProvider is configured with a **journald
exporter** (the local durable store, keeping ADR-0013's rotation/retention intact
— journald is never removed from the path) and an **OTLP log exporter** per
configured target (purely additive). Log level filtering is global with
per-subsystem overrides. Audit events (ADR-0013 tagged journald entries) flow
through the same pipeline carrying an audit attribute, so they remain identifiable
locally and can be forwarded when configured. Third-party daemons that do not speak
OpenTelemetry (samba, nerdctl/containerd, kernel, sshd) remain journald-only.

## Alternatives considered

- **Event log only**: tells you what happened, not the current state or rates.
- **Per-subsystem dashboards without a standard**: not composable.
- **Prometheus as the instrumentation model**: the original decision; rejects a
  unified multi-signal backend story (see Context).
- **Global singleton telemetry config read at startup**: violates declarative
  reconcile — telemetry changes would require a restart.
- **A config file on the `config/var` partition**: bypasses the API resource model,
  the status object, RBAC, and the audit trail.
- **An on-device OTEL Collector**: rejected by the scrape-only, no-embedded-daemon
  hosting decision; exporters are always in-process.
- **No tracing (metrics only)**: metrics tell you a controller is failing, not the
  per-resource causal path through a reconcile pass.
- **Explicit logging of reconcile steps**: unbounded volume, not structured, cannot
  carry span parentage.
- **Replace journald with OTLP export as the store**: breaks ADR-0013's local
  durable store, rotation, and retention, and couples local logs to off-box
  availability.
- **A second parallel hand-rolled log path**: duplicates the pipeline and loses the
  single-API property.

## Consequences

- A metrics endpoint ships in `core`; the OTEL MeterProvider with its Prometheus
  exporter backs `/metrics` (feature 9). Metric names, labels, and histogram
  buckets are part of the `contracts` surface, declared in the metric catalog and
  enforced by OTEL Views.
- Hosting is scrape-only: `core` exposes `/metrics` and an external Prometheus
  scrapes it. There is no embedded Prometheus server (no stateful storage on the
  RO-root device); on-device dashboards, if any, are a `web-ui`/`observability`
  question, not a hosting one.
- A telemetry controller reconciles the spec into provider lifecycles; providers
  are constructed at daemon start from the desired state and rebuilt on spec change
  (ADR-0017).
- OTEL modules are version-pinned and baked into the RO image (ADR-0001); no
  runtime `go get`. Traces/logs flow through the same OTEL SDK.
- Reconcile passes are traced with a root span per pass and child spans per
  controller (ADR-0017); tracing is off by default and produces no data until a
  target and a non-zero sampling rate are configured. Trace data is exported only
  to configured OTLP targets; there is no on-device trace store.
- One logging API; journald and OTLP are parallel exporters from the same
  LoggerProvider (dual-write). ADR-0013 stands: journald on tmpfs forwarded to
  `config/var` with rotation and retention; audit entries remain tagged locally.
- The OTEL Go log SDK is beta — the log pipeline is the riskiest piece of the
  telemetry change and starts with a validation spike; the documented fallback is
  a thin logging facade that writes to journald and to the OTEL LoggerProvider
  from one call site, preserving the single-API property. Dual-write overhead is
  bounded by batch exporters with bounded buffers; the journald write is
  fire-and-forget.
- The telemetry capability is part of the RBAC capability map (ADR-0018): the
  `admin` role reads and writes telemetry configuration; `auditor` and `read-only`
  roles view it.
- Alerting (deferred, scope-missing-plane D6) can later consume the same metrics.
- First off-box push in a no-TLS LAN (v1): per-target TLS modes default to `none`
  for convenience, with a docs warning; telemetry config is admin-only.
