# ADR-0016: Logging and audit via journald, forwarded to the OS-disk config/var partition

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — `config/var` now inside the OS-disk LUKS container (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 13; ADR-0001, ADR-0002,
  ADR-0007, ADR-0008, ADR-0011, ADR-0031

## Context

Feature 13 (logging + audit) had no hosting decision. Logs cannot live under the
read-only root (ADR-0001), and system config state does not live on a pool
(ADR-0007) — it lives on OS-disk partitions (ADR-0011). The audit trail needs
actor context that only exists at API admission (ADR-0002).

## Decision

System logs are captured by journald on tmpfs and forwarded to the `config/var`
partition on the OS disk (ADR-0011) with rotation and retention. The audit trail
is journald entries carrying actor, resource, and action, emitted at API admission
(desired-state writes) and by reconcile mutations; they are namespaced `audit`
and forwarded to the same target. Metrics covering log/audit health are exposed
per ADR-0008.

## Alternatives considered

- **Separate audit log file**: queryable independently, but a second pipeline to
  build and retain.
- **Audit derived from spec-store history**: couples audit to the rollback/versioning
  story and misses reconcile-side and non-spec events.
- **Forward to a `config` pool dataset**: gives ZFS snapshots of logs, but couples
  logging to pool availability and reintroduces the boot/OS-state coupling to
  pools that ADR-0011 rejects; the `config/var` partition replaces it.

## Consequences

- One logging pipeline with a tagged `audit` namespace; standard journald tooling.
- `journald` on tmpfs means logs exist only up to forward latency after a hard
  crash; acceptable for v1.
- Retention and rotation are configured on the `config/var` partition forward
  target; there is no snapshot story for logs (ADR-0011).
- With host encryption (ADR-0031), `config/var` sits inside the OS-disk LUKS2
  container: forwarded logs/audit and generated daemon config fragments are
  encrypted at rest once the container is unlocked. This does not contradict the
  rotation/regeneration guarantees — logs accumulate on an unlocked host exactly
  as before, and encryption does not change retention or rotation behavior. A
  locked host cannot boot the OS at all (ADR-0031), so there is no
  locked-but-rotating state to reconcile with.
- Audit and metrics together make admin actions observable (ADR-0008).
