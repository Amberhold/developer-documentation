# ADR-0030: Off-site backup of ZFS snapshots with restic

- Status: accepted
- Date: 2026-08-16
- Amended: 2026-08-27 — repository password now protected at rest inside the OS-disk LUKS container (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 2 (feature 15);
  ADR-0001, ADR-0006, ADR-0013, ADR-0015, ADR-0017, ADR-0019, ADR-0022, ADR-0031;
  `backup-restic-snapshots` D1–D6

## Context

ZFS snapshots (feature 2, ADR-0022) are the only protection primitive and are
local-only: they guard against user error and corruption, not against data-pool
loss, disk theft, or site-level failure. Backup was deferred in the feature map
with snapshots named the primitive a future replication feature builds on.
Off-site backup closes the data-pool-loss gap while reusing the snapshot
primitive and the existing external-tool and co-located-secret conventions.

The read-only squashfs root (ADR-0001) means external binaries must be baked
into the image and invoked by `core` by shelling out (the smartctl/nerdctl
pattern); systemd timers are unavailable, so cadence must come from the
reconciler. The OS-disk spec-store partition already holds encryption keyfiles
auto-loaded at start (ADR-0013, ADR-0015), establishing the co-located-secret
pattern this decision reuses.

## Decision

Backups back up **snapshots of opted-in datasets** to a remote **restic**
repository. restic is a Go static binary with no runtime dependencies, so it is
pinned (version + verified hash) in the read-only image and invoked by `core`
by shelling out, matching the external-tool pattern of ADR-0001/ADR-0006.

Backup is **event-driven ingestion per snapshot**: the backup controller
observes the shared event source (ADR-0017) and ingests each newly created
snapshot on an opted-in dataset into the repository. There is **no new
`Schedule` consumer** — restic's chunk-level dedup makes per-snapshot runs
cheap, and the repository becomes the durable retention store while local
snapshot retention stays short. Cadence stays governed by the existing snapshot
schedule (ADR-0022); ADR-0022's shared-schedule scope (snapshots + scrubs) is
untouched.

Datasets are backed up **per-dataset opt-in**: the `datasets` resource spec
gains a `backup` field (the source of truth, ADR-0019), which the storage
controller reflects onto an `amberhold:backup` ZFS user property so the pool
itself carries the record. The app-images dataset is excluded by default as
reproducible.

A backup run mounts the target snapshot **read-only** (`zfs mount -o ro`) and
points restic at the mountpoint, giving a crash-consistent, point-in-time view
with file-level restore. **zvols** cannot be mounted as filesystems; they are
covered by streaming `zfs send <vol>@<snap>` into `restic backup --stdin`.

The **repository password is co-located** on the OS-disk spec-store partition
and auto-loaded by `core` at start, exactly the ADR-0015 keyfile pattern — no
new installer or password-recovery UX. With host encryption (ADR-0031), the
spec-store partition is inside the OS-disk LUKS2 container, so the repository
password is protected at rest and auto-loaded only after the container unlocks.
The recovery boundary is therefore **data-pool loss only**: the repository does
not recover a lost OS disk or the repository password itself, and the D1
posture (OS-disk loss loses keys, password, and data) is unchanged.

A `Backup` resource (ADR-0019) carries the remote URI + backend type
(S3-compatible, SFTP, rest-server), the retention policy, and an implicit
selector over all opted-in datasets (one repository per system in v1). The
controller reconciles it: `restic init` when the repository is absent, `backup`
per snapshot event, `forget --prune` when the retention policy demands.

**Out of v1 scope**, recorded here so the design is preserved:

- **Restore flow** — the mechanism is restic `restore` into a new dataset, but
  the restore controller/action endpoint is a separate change.
- **User-managed/exportable repository password** — the co-located password
  keeps D1 unchanged; a future path that flips D1 (installer + recovery UX) is
  deliberately not in v1.

## Alternatives considered

- **kopia**: Go and deduplicating, but less mature with a thinner zvol story.
- **borg**: deduplicating, but needs a Python runtime in the image — heavy for
  the RO root.
- **`zfs send/receive` replication**: native and efficient, but needs a ZFS
  sink at the far end and no object-store support; replication stays deferred.
- **A dedicated backup `Schedule` consumer**: an extra knob decoupling cadence
  from snapshots — rejected for v1 simplicity; per-snapshot ingestion with
  dedup keeps runs cheap.
- **Back up every dataset**: rejected — backs up disposable/reproducible data
  (e.g. app images); per-dataset opt-in keeps the backup set meaningful.
- **ZFS-property-only opt-in**: rejected — the declarative spec (ADR-0002) is
  the source of truth; the property is a reflection for pool-side tooling.
- **User-managed repository password**: flips D1 and adds installer + recovery
  UX — recorded as the future path, not v1.

## Consequences

- A new backup controller in `core` (restic shell-out) joins the per-controller
  reconcile loops (ADR-0017) and is listed in the feature-map skeleton.
- `Backup` joins the API resource model (ADR-0019); the resource shape and the
  `amberhold:backup` user property are part of the `contracts` surface, authored
  by a later change.
- The feature map promotes backup to feature 15; replication
  (`zfs send/receive` to a remote ZFS host) remains deferred.
- The repository protects against data-pool loss only; OS-disk loss keeps its
  D1 unrecoverable posture (ADR-0015) — the repository password is lost with
  the OS disk. At rest it is now protected inside the OS-disk LUKS container
  (ADR-0031) and auto-loaded only after unlock, matching the encryption keyfiles;
  its D1 recovery-boundary text is unchanged.
- Failure of a backup run retries with backoff and is surfaced via the
  `Backup` resource status and Prometheus metrics; the repository retains prior
  state, so no data is lost while the remote is unreachable.
- restic cache must not write under the RO root; `--cache-dir` points at the
  writable state partition (ADR-0011 `config/var`).
