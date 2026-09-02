# Telemetry Controller — OTEL Substrate, Provider Lifecycle, and OTLP Export

> Discovery-phase design. Authored from the `telemetry-controller` openspec
> change. The decisions D-T1–D-T9 here fix how the `TelemetryController`
> reconciles the singleton `telemetry` resource (ADR-0008) into running
> Meter/Tracer/Logger providers, how controllers resolve providers through the
> D8 handle, and how metrics, traces, and logs are exported (Prometheus
> always-on + OTLP per target + journald durable store). ADR-0008 fixes the
> *decision* (OpenTelemetry substrate for all three signals, Prometheus
> always-on, journald durable + OTLP additive); this document fixes the
> *internal mechanics*, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031). The feature map
> (feature 9 observability) is in `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

The `telemetry` singleton, its OpenAPI contract (`Telemetry`/`TelemetryDefault`/
`TelemetryOverride`/`OtlpTarget` in `contracts/openapi/v1.yaml`), the RBAC
capabilities (`telemetry:read`/`telemetry:write`), and the metrics
(`amberhold.telemetry.provider.ready`, `amberhold.logs.*` in
`contracts/metrics/catalog.yaml`) are all declared, but the daemon ships the
skeleton's hand-rolled substrate: a catalog-enforcing metrics `Registry`
rendering `/v1/metrics`, per-subsystem `slog` loggers to stderr, and a no-op
tracer. D8's "resolve per use" is nominal only — sinks are wired once at
construction. This slice lands the telemetry capability for real: the
`TelemetryController` owning the provider lifecycle, the OTEL substrate behind
the existing emission API, reconcile-loop tracing, the journald + OTLP log
pipeline, and per-target OTLP export.

## 2. Goals / Non-Goals

**Goals:**
- Land the `Telemetry` resource end-to-end: controller, `/v1/telemetry` API,
  provider lifecycle with restart-free reconvergence (ADR-0002).
- Move the emission substrate to the OpenTelemetry SDK for all three signals
  without rewriting every controller's call sites (the `MetricSink` emission
  API survives unchanged).
- Keep `/v1/metrics` behaviorally identical through the migration: Prometheus
  always-on, catalog enforced by OTEL Views compiled from
  `contracts/metrics/catalog.yaml`.
- Make tracing and OTLP export real, off by default (zero data until a target
  and a non-zero sampling rate are configured).
- Route all core-component logs through one LoggerProvider with journald as the
  durable local store and OTLP additive (ADR-0008/0013).

**Non-Goals:**
- No on-device OTEL Collector (ADR-0008: in-process exporters only).
- No on-device dashboards or Prometheus server (scrape-only hosting).
- No changes to journald rotation/retention or the audit path (ADR-0013).
- No OTEL migration of third-party daemons (samba, containerd, kernel); they
  remain journald-only.
- No alerting (feature-map deferred area).

## 3. Decisions

### D-T1: Provider lifecycle — rebuild-and-swap, never mutate in place

The `TelemetryController` owns the three providers. On spec change it builds a
**new immutable provider set** from the desired state and atomically swaps it
into the `Provider` handle, then shuts down the previous set after a short grace
period.
- *Why:* Incremental reconfiguration of live OTEL SDK state (adding/removing
  exporters, changing samplers, TLS) is fragile and leaks per-exporter state.
  Immutable sets make the swap race-free: in-flight emissions finish against
  the old set, new resolutions see the new set.
- *Alternative considered:* mutate providers in place on spec change — rejected
  for exporter-lifecycle leak risk and no atomicity across the three providers.

### D-T2: The handle is an atomic snapshot resolver

`telemetry.Provider` keeps its name and role but its internals become an
`atomic.Pointer[Providers]` holding `{MeterProvider, TracerProvider,
LoggerProvider}`. `Resolve()` returns the current set; controllers resolve per
loop (not per emit) and cache their subsystem meter/tracer/logger for the loop.
The `Registry` renderer moves behind the MeterProvider's Prometheus exporter,
which serves `/v1/metrics` (task 1.4).
- *Why:* D8's contract is "resolve the current provider per use"; a pointer
  snapshot is the minimal race-free indirection and makes the swap in D-T1
  trivial. An epoch counter on the handle invalidates cached OTEL instruments
  on swap, so an emission is never stranded on a stale provider.
- *Alternative considered:* per-controller callbacks re-injected at swap —
  rejected, requires controller registry bookkeeping the runtime does not have.

### D-T3: Metric emission migrates through the existing MetricSink seam

The consumer-defined `MetricSink` interfaces (`Add`/`Set`/`Observe` with catalog
names) **survive unchanged**. Their implementation (`telemetry.Registry`) is
rewritten as a thin adapter over per-subsystem OTEL meters obtained from the
handle: each catalog name maps to a registered counter/gauge/histogram
instrument, with histogram buckets taken from the catalog. Catalog enforcement
moves from the registry's `unsupported` map to OTEL Views compiled from
`contracts/metrics/catalog.yaml` at startup (unregistered names are rejected by
the MeterProvider). The registry retains a synchronous mirror so `Snapshot`/
`Render` stay deterministic for tests and the legacy `/v1/metrics` path during
migration. Sinks are re-resolved through the handle per emission (the epoch
check), strictly more often than the per-loop resolution the contract requires.
- *Why:* the emission API is already consumer-defined and stable; migrating the
  substrate behind it lands OTEL everywhere without a sweep of every controller
  in one change, and keeps the per-subsystem meter scoping ADR-0008 requires.
- *Alternative considered:* rewrite every controller to call OTEL instruments
  directly — more faithful to the letter of ADR-0008 but high churn and no
  behavioral gain; the adapter keeps the same per-subsystem meters underneath.

### D-T4: Tracing is runtime-owned via a span adapter

The runtime (D10) creates the root `reconcile` span per pass and a child span
per controller step from the handle's TracerProvider. The existing
`Tracer.StartSpan/SetAttributes/End` call shape is preserved as a thin adapter
over OTEL spans so `internal/runtime` and the `Controller` SDK do not churn.
Sampling uses a parent-based sampler whose local sampler reads the span's
`subsystem` attribute and applies the per-subsystem override rate (falling back
to the global default); children follow their parent (parent-based contract).
With a zero global rate and no overrides the provider uses `NeverSample`, so no
trace data is produced until a non-zero rate and a traces-enabled target are
configured.
- *Why:* D10 centralizes the span convention; the attribute-driven sampler maps
  the spec's `overrides.tracesSamplingRate` map directly onto per-subsystem
  root spans.

### D-T5: Log pipeline — slog API, OTEL LoggerProvider, custom journald exporter

`slog` stays the in-process logging API (pervasive, structured). A bridge
handler routes records into the OTEL LoggerProvider, which dual-writes to (a) a
small **custom journald exporter** (writes to the journald socket `/dev/log` —
the durable store per ADR-0013/0008) and (b) one OTLP log exporter per
configured target. Level filtering applies globally with per-subsystem
overrides via the bridge's `Enabled` check. The single call site
(slog → LoggerProvider) preserves the ADR-0008 fallback seam: if the log SDK
misbehaves, the bridge can dual-write to journald directly without touching any
controller.
- *Why:* no upstream journald exporter exists; a ~50-line socket writer is
  smaller than adopting a systemd dependency. The bridge keeps controllers'
  existing `Logger(subsystem)` usage untouched.
- *Alternative considered:* journald-only + OTLP side path (two APIs) — rejected,
  breaks ADR-0008's single-API property.

### D-T6: Controller shape — one kind, singleton status

`TelemetryController` owns kind `Telemetry` (D1). Reconcile: load spec →
compute desired provider set (defaults + overrides + targets) → build →
swap handle → persist status (`state` converged/reconciling/error +
`providers` per-subsystem readiness) through the store's serialized path (D4).
Empty spec = ADR-0008 defaults (metrics enabled, sampling 0, level info, no
targets). Construction failure keeps the previous set in service and reports
`error` with the failing target identified (D-T1 consequence). An unchanged
spec (canonical JSON compare) is an idempotent no-op (D9).

### D-T7: OTLP target TLS references are config/var file paths

`ca`/`cert`/`key` in `OtlpTargetTls` are **absolute file paths on the OS-disk
`config/var` partition** (PEM files, admin-provided, per the writable-config
convention ADR-0001). Provider construction validates existence and PEM
parseability; `mtls` additionally validates key/cert match. They do **not**
reference `certificates` resources.
- *Why:* the cert controller manages the management-plane **serving** identity;
  outbound OTLP client identity is a different concern, and coupling them would
  drag renewal/rotation semantics into export config. `config/var` paths match
  how cert files are already stored.
- *Alternative considered:* cert-resource references — rejected above.

### D-T8: API and RBAC reuse the singleton pattern

`GET/PUT/PATCH /v1/telemetry` routes mirror the `certificates`/`network`
singletons (handler → store → admission → audit). `telemetry:read` /
`telemetry:write` capabilities are enforced by admission; writes are audited
like any spec write. Spec validation at the boundary rejects malformed targets
(endpoint required, signals non-empty, TLS modes valid) before they reach the
controller.

### D-T9: Failure observability

The log pipeline and OTLP exporters emit `amberhold.logs.forwarded`,
`amberhold.logs.journald.failures`, and `amberhold.logs.otlp.failures`
counters; the controller sets `amberhold.telemetry.provider.ready` gauges per
subsystem/signal. Failures never block local production: batch exporters with
bounded buffers and timeouts, fire-and-forget journald writes, and the OTLP
log exporter is wrapped with a failure counter that reports the error to the
batch processor for retry.

## 4. Reconciliation

The `TelemetryController` owns the singleton `telemetry` kind (D1) and converges
the desired spec against the running provider set (task 2.2). Each pass:

- **Parse** the spec (defaults + per-subsystem overrides + OTLP targets); an
  empty spec is the ADR-0008 default set (D-T6).
- **Build** a new immutable `Providers` set: the MeterProvider with the
  always-on Prometheus exporter (catalog Views) plus one periodic OTLP metric
  reader per metrics-enabled target; the TracerProvider with the
  subsystem-aware parent-based sampler plus one OTLP span processor per
  traces-enabled target; the LoggerProvider with the journald exporter plus one
  OTLP log processor per logs-enabled target (D-T1/D-T5).
- **Swap** the set into the handle atomically and update the effective log
  levels; the previous set is shut down after a grace period.
- **Status**: write `status.state` (converged/reconciling/error) and
  `status.providers` (per-subsystem meter/tracer/logger readiness) through the
  store's serialized path (D4). Construction failure keeps the previous set in
  service and reports `error` with the failing target (D-T6).
- **Metrics**: publish the `amberhold.telemetry.provider.ready` gauges
  (labels `subsystem`, `signal`) from the catalog (D-T9).

## 5. The provider handle for controllers

`telemetry.Provider` (D8) is the atomic snapshot resolver (D-T2). The runtime's
reconcile loop resolves the current TracerProvider per pass and creates the
root `reconcile` span + child `reconcile.step` span per controller step carrying
the resource id and wanted/actual versions (D10, task 3.2). The `Registry`
metric sink resolves the current MeterProvider per emission (D-T3). The
`Logger(subsystem)` surface returns the slog bridge resolving the current
LoggerProvider per record with the effective levels (D-T5, task 4.1).

## 6. Status and observability

The controller writes the declared `TelemetryStatus` shape (state,
providers[keyed by subsystem]) through the store (D4). Metrics
(`amberhold.telemetry.provider.ready`, `amberhold.logs.forwarded`,
`amberhold.logs.journald.failures`, `amberhold.logs.otlp.failures`) are
published from the catalog (`contracts/metrics/catalog.yaml`) through the
daemon registry (D8/D10); no unregistered telemetry metric name is emitted
(ADR-0008, enforced by OTEL Views).

## 7. Risk / trade-offs

- **OTEL log SDK maturity (beta, waived spike)** → pin the exact SDK version in
  `go.mod` (and the image, ADR-0001); keep the D-T5 bridge as the single
  dual-write call site so the documented facade fallback is a small change if
  the SDK regresses.
- **Wide-touching metric migration (every controller)** → the MetricSink
  adapter (D-T3) keeps call sites and behavior stable; the existing test suite
  (registry snapshot + exposition assertions) was migrated to assert both the
  synchronous mirror and the OTEL pipeline, guarding the sweep.
- **Provider swap races with in-flight emission** → atomic snapshot swap (D-T1/
  D-T2); old set shut down after a grace period; OTEL shutdown is idempotent.
- **OTLP failure feedback loops** → failures only count metrics; local emission
  never blocks; bounded buffers bound memory; the OTLP/HTTP exporters are wired
  with explicit signal paths so a path-less endpoint URL still exports to
  `/v1/{metrics,traces,logs}`.
- **journald exporter on non-Linux (tests)** → exporter behind an interface
  seam; fake journald for unit tests, real socket writer on Linux (absent
  `/dev/log` degrades to stderr with a logged warning).
- **Label/bucket drift between catalog and OTEL instruments** → Views compiled
  from the catalog at startup; a startup validation test cross-checks every
  declared catalog metric against its instrument and the histogram bucket
  boundaries.

## 8. Migration plan

1. Land the OTEL substrate behind the existing emission API: meters + Views
   from the catalog + Prometheus exporter serving `/v1/metrics`. Behavior
   identical; full suite green before anything else moves.
2. Telemetry controller + handle swap + `/v1/telemetry` API + status + RBAC.
3. Tracing wiring in the runtime (root/child spans, samplers).
4. Log pipeline: slog bridge + journald exporter + level filtering.
5. OTLP exporters for all three signals + per-target TLS modes + failure
   metrics.

Rollback: reverting the `Telemetry` spec converges providers back to defaults;
a daemon restart rebuilds from the persisted spec. Binary rollback is the A/B
slot path (future update controller), out of scope here.

## 9. Open questions (implementation-time)

- Exact OTEL Go SDK version pin (resolved at implementation from the current
  stable release; the pin is recorded in the ADR-0008 amendment).
- journald socket writer details (native `/dev/log` unixgram vs
  `go-systemd/journal` helper) — implementation-time, behavior identical
  (resolved: native unixgram datagrams).
- Histogram bucket translation edge cases for catalog-declared buckets vs OTEL
  explicit-bucket histograms — covered by the startup validation test.