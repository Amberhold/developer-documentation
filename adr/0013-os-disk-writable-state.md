# ADR-0013: OS disk writable state — spec store and `config/var`

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — `config/var` and the spec store now inside the OS-disk
  LUKS container (host-encryption change); serialized write path with optimistic
  concurrency, coalesced status writes, and an in-memory read index
  (`core-daemon-design`, D5)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, §3.7, features 2,
  13, 14; `docs/architecture/02-core-daemon.md`; ADR-0001, ADR-0002, ADR-0007,
  ADR-0008, ADR-0011

> Consolidates the former ADR-0013 (spec store on an OS-disk partition) and
> ADR-0016 (logging + audit via journald forwarded to `config/var`). Both concern
> the *logical writable state* that lives on the dedicated OS disk; the physical
> layout and encryption of that disk is ADR-0011.

## Context

The spec store is the source of truth (ADR-0002) and must be versioned/snapshotted
to support the revert half of composite rollback. ADR-0011 dedicates an OS disk to
the A/B slots and bootloader but does not say where the spec store lives — its
placement decides boot ordering (does `core` need a pool imported before it can read
its own desired state?) and the failure story for both the OS disk and the pools.
Similarly, feature 13 (logging + audit) had no hosting decision: logs cannot live
under the read-only root (ADR-0001), system config state does not live on a pool
(ADR-0007), and the audit trail needs actor context that only exists at API
admission (ADR-0002).

## Decision

The spec store and the `config/var` partition both live on the dedicated OS disk
(ADR-0011), outside the A/B slots, and are read/written only after the OS-disk
container is unlocked.

### Spec store

The spec store lives on a persistent partition on the dedicated OS disk, alongside
the encryption keyfiles (ADR-0011). Boot reads the spec before any pool import:
`core` starts reconciling from the spec with pools absent, then imports pools as
they become available. Updates swap slots and never touch the spec store. This
partition is plain (no fs snapshots); its only recovery mechanism is its own
internal versioning.

The spec store is versioned: it records a schema version, and `core` migrates the
store forward on read. Rollback restores a prior versioned snapshot rather than
reverse-migrating, so an older `core` never reads a newer schema.

The write path is serialized through the store's single writer (a
mutex-protected append), per `core-daemon-design` D5:

- **Spec writes**: API admission → RBAC → optimistic concurrency check on
  `resourceVersion` → append (bump `metadata.generation`, new `resourceVersion`)
  → publish event → audit.
- **Status writes**: the owning controller (ADR-0017 D4), serialized, coalesced
  on a short minimum interval to avoid write amplification on busy resources.
- An **in-memory index** serves API reads, keeping them off the write path.
- The on-disk concrete format (versioned JSON snapshot directories vs an embedded
  KV) is reconsiderable at implementation without changing this approach.

### Logging + audit via journald

System logs are captured by journald on tmpfs and forwarded to the `config/var`
partition on the OS disk with rotation and retention. The audit trail is journald
entries carrying actor, resource, and action, emitted at API admission
(desired-state writes) and by reconcile mutations; they are namespaced `audit` and
forwarded to the same target. Metrics covering log/audit health are exposed per
ADR-0008. The core log path is the OTEL pipeline with journald as the durable
parallel exporter (ADR-0008); third-party daemons that do not speak OpenTelemetry
(samba, nerdctl/containerd, kernel, sshd) remain journald-only.

With host encryption (ADR-0011), the spec store (and with it the keyfiles) and
`config/var` sit inside the OS-disk LUKS2 container and are only readable once the
container is unlocked.

## Alternatives considered

- **Spec store on a `config` pool dataset**: fits the earlier ADR-0007 role model
  (the `config` role has since been removed), but makes boot depend on pool import
  and couples the source of truth to pool availability.
- **Spec store on a user data pool**: couples config to user data and its failure
  modes.
- **Separate audit log file**: queryable independently, but a second pipeline to
  build and retain.
- **Audit derived from spec-store history**: couples audit to the
  rollback/versioning story and misses reconcile-side and non-spec events.
- **Forward logs to a `config` pool dataset**: gives ZFS snapshots of logs, but
  couples logging to pool availability and reintroduces the boot/OS-state coupling
  to pools that ADR-0011 rejects; the `config/var` partition replaces it.

## Consequences

- Boot ordering is independent of pools; a missing/unimportable pool degrades
  reconcile scope rather than blocking `core` startup.
- Update (ADR-0006) never touches the spec store — slot swap and desired state are
  cleanly separated.
- The store's schema version makes update/rollback (ADR-0006) explicit: a new
  `core` migrates forward; a rolled-back `core` restores a compatible prior
  snapshot. Downgrade across a schema change is never attempted.
- OS-disk failure means config loss; recovery is reinstall + `zpool import`
  (ADR-0011 recovery path). This applies to unencrypted data only: the encryption
  keyfiles live on the OS disk (ADR-0011), so encrypted pools and datasets are lost
  with it.
- OS-disk sizing must reserve headroom for spec-store versioning/snapshots; sizing
  is an install-time check (ADR-0011), with a 128–256 GB floor.
- A single serialized writer (D5) avoids torn spec/status state and keeps
  optimistic concurrency on `resourceVersion` coherent; low-rate admin spec
  writes and coalesced status writes bound write amplification, and the
  in-memory index keeps API reads off the write path.
- One logging pipeline with a tagged `audit` namespace; standard journald tooling.
  `journald` on tmpfs means logs exist only up to forward latency after a hard
  crash; acceptable for v1. Retention and rotation are configured on the
  `config/var` partition forward target; there is no snapshot story for logs.
- With host encryption, forwarded logs/audit and generated daemon config fragments
  are encrypted at rest once the container is unlocked; this does not contradict
  rotation/regeneration guarantees — logs accumulate on an unlocked host exactly as
  before, and encryption does not change retention or rotation behavior. A locked
  host cannot boot the OS at all (ADR-0011), so there is no locked-but-rotating
  state to reconcile with.
- Audit and metrics together make admin actions observable (ADR-0008).
