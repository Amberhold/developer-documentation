# ADR-0016: Logging and audit via journald, forwarded to the config dataset

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 13; ADR-0001, ADR-0002,
  ADR-0007, ADR-0008

## Context

Feature 13 (logging + audit) had no hosting decision. Logs cannot live under the
read-only root (ADR-0001), so they must land on a writable dataset (ADR-0007), and
the audit trail needs actor context that only exists at API admission (ADR-0002).

## Decision

System logs are captured by journald on tmpfs and forwarded to the `config`
dataset with rotation and retention. The audit trail is journald entries carrying
actor, resource, and action, emitted at API admission (desired-state writes) and
by reconcile mutations; they are namespaced `audit` and forwarded to the same
target. Metrics covering log/audit health are exposed per ADR-0008.

## Alternatives considered

- **Separate audit log file**: queryable independently, but a second pipeline to
  build and retain.
- **Audit derived from spec-store history**: couples audit to the rollback/versioning
  story and misses reconcile-side and non-spec events.

## Consequences

- One logging pipeline with a tagged `audit` namespace; standard journald tooling.
- `journald` on tmpfs means logs exist only up to forward latency after a hard
  crash; acceptable for v1.
- Retention and rotation are configured on the `config` dataset forward target.
- Audit and metrics together make admin actions observable (ADR-0008).
- The `config` dataset is snapshotted on the shared `Schedule` resource
  (ADR-0022), protecting forwarded logs/audit alongside encryption keyfiles
  (ADR-0015, resolve-control-plane-gaps D13).
