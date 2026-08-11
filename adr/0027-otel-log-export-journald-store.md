# ADR-0027: Direct OTEL log export with journald as the durable local store

- Status: accepted
- Date: 2026-08-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` features 9, 13;
  ADR-0001, ADR-0008, ADR-0016, ADR-0025, ADR-0026

## Context

ADR-0016 commits local logging to journald with rotation and retention on the
OS-disk `config/var` partition. ADR-0008 (amended) makes OpenTelemetry the
instrumentation layer for all three signals. Core components need a single
logging API that keeps journald as the durable local store while also being able
to forward logs off-box when an OTLP target is configured.

## Decision

Every core component (controllers, API, reconcilers) emits its logs through the
OpenTelemetry logging pipeline, with a logger scoped to its subsystem. The
LoggerProvider is configured with:

- a **journald exporter** — the local durable store, keeping ADR-0016's rotation
  and retention intact; journald is never removed from the path, so local
  operation is independent of off-box availability;
- an **OTLP log exporter** per configured target (ADR-0025), purely additive.

Log level filtering is applied globally with per-subsystem overrides; records
below the effective level are not emitted or exported.

Audit events (admin actions; ADR-0016 tagged journald entries) are emitted
through the same pipeline carrying an audit attribute, so they remain
identifiable locally and can be forwarded to OTLP targets when configured.

Logs from third-party daemons that do not speak OpenTelemetry (samba,
nerdctl/containerd, kernel, sshd) remain journald-only in this change and are
not forwarded over OTLP.

## Alternatives considered

- **Replace journald with OTLP export as the store**: breaks ADR-0016's local
  durable store, rotation, and retention, and couples local logs to off-box
  availability.
- **A second parallel hand-rolled log path**: duplicates the pipeline and loses
  the single-API property.

## Consequences

- One logging API; journald and OTLP are parallel exporters from the same
  LoggerProvider (dual-write).
- ADR-0016 stands: journald on tmpfs forwarded to `config/var` with rotation and
  retention; audit entries remain tagged locally.
- The OTEL Go log SDK is beta — the log pipeline is the riskiest piece of the
  telemetry change and starts with a validation spike; the documented fallback is
  a thin logging facade that writes to journald and to the OTEL LoggerProvider
  from one call site, preserving the single-API property.
- Dual-write overhead is bounded by batch exporters with bounded buffers; the
  journald write is fire-and-forget.
