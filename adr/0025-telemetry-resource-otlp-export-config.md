# ADR-0025: Telemetry resource and OTLP export configuration

- Status: accepted
- Date: 2026-08-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 9, §3.8;
  ADR-0001, ADR-0002, ADR-0008, ADR-0017, ADR-0018, ADR-0019, ADR-0026,
  ADR-0027

## Context

ADR-0008 (amended) makes OpenTelemetry the instrumentation layer with Prometheus
as the always-on default exporter. Amberhold needs an off-box observability story
as well: metrics, traces, and logs should be able to reach an external backend.
That requires configuration — which targets to export to, with which protocol and
TLS, and per-subsystem control of sampling and log levels — expressed through the
declarative API model (ADR-0019), reconciled without daemon restarts (ADR-0002).

## Decision

A singleton `Telemetry` resource in the API resource model carries desired-state
spec and status, reconciled by a telemetry controller. The controller owns the
lifecycle of the running MeterProvider, TracerProvider, and LoggerProvider;
controllers and meters/tracers/loggers reference the providers and never build
their own. Writing desired state converges the running providers without a
daemon restart.

The resource has:

- A **global default block** and an **overrides map keyed by subsystem id**
  (pools, shares, apps, api, updates, identity, disk-health, network, logs).
  Overrides carry only the fields they change (traces sampling rate, log level,
  metric overrides) and take precedence over the global default; removing an
  override falls back to the default.
- An **OTLP target list**. Each target declares an endpoint, a protocol, and a
  TLS mode of `none`, `tls`, or `mtls` with optional CA/certificate/key
  references. TLS is optional per target; v1 defaults to `none` for the plain-HTTP
  LAN posture, with the configuration surface available from day one.

Only in-process exporters exist: Prometheus (always on), OTLP (one per configured
target, wired for whichever signals the target enables), and journald (log export
only, ADR-0027). No OTEL Collector runs on the appliance (ADR-0008 hosting). OTLP
is purely additive and never replaces the Prometheus endpoint.

RBAC is enforced at API admission under ADR-0018: the `admin` role reads and
writes telemetry configuration; `auditor` and `read-only` roles view it.

## Alternatives considered

- **Global singleton config read at startup**: violates declarative reconcile —
  telemetry changes would require a restart.
- **A config file on the `config/var` partition**: bypasses the API resource
  model, the status object, RBAC, and the audit trail.
- **An on-device OTEL Collector**: rejected by the scrape-only, no-embedded-daemon
  hosting decision (ADR-0008); exporters are always in-process.

## Consequences

- `Telemetry` joins the API resource model (ADR-0019) and the `contracts` surface.
- A telemetry controller reconciles the spec into provider lifecycles; providers
  are constructed at daemon start from the desired state and rebuilt/reconfigured
  on spec change (ADR-0017).
- OTEL modules are version-pinned and baked into the RO image (ADR-0001); no
  runtime `go get`.
- First off-box push in a no-TLS LAN (v1): per-target TLS modes default to `none`
  for convenience, with a docs warning; telemetry config is admin-only.
- The telemetry capability is part of the RBAC capability map (ADR-0018) and is
  observable via its own metrics (ADR-0008).
