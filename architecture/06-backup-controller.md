# Backup Controller — Off-site restic Archive of Opted-in Snapshots

> Discovery-phase design. Authored from the `offsite-backup-restic` openspec
> change. The decisions D-BK1–D-BK8 here fix how the `Backup` controller
> reconciles the singleton `Backup` resource (feature 15) so each newly created
> snapshot on an opted-in dataset is ingested into a remote restic repository,
> converging repository retention to the declared policy. ADR-0030 fixes the
> *decision* (restic archive of opted-in ZFS snapshots, event-driven
> per-snapshot ingestion, co-located password); this document fixes the
> *internal mechanics*, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031), the storage
> data-plane anchor in `docs/architecture/03-storage-controller.md` (D-S1–D-S13),
> the shares slice in `docs/architecture/04-shares-controller.md`
> (D-FS1–D-FS8), and the block-shares/zvols slice in
> `docs/architecture/05-block-shares-zvols.md` (D-B1–D-B6). The feature map
> (feature 2 snapshots, feature 15 off-site backup) is in
> `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

Storage, shares, and block shares are implemented and green. Off-site backup —
feature 15 — is the remaining data-protection gap: ZFS snapshots (feature 2,
ADR-0022) are the only protection primitive and are local-only, guarding
against user error and corruption but not data-pool loss, disk theft, or
site-level failure (ADR-0030). The contract surface is fully declared — the
`Backup` resource, the `datasets.spec.backup` opt-in already reflected onto the
`amberhold:backup` ZFS user property by the `Dataset` controller (D-S9), the
snapshot controller's external-hold pattern, and the `amberhold.backup.*` metric
catalog — but there is no controller and no restic/zfs-send primitive in the
host facades. This slice adds both: a `ResticHost` facade over the pinned restic
binary, snapshot mount/send primitives on the storage host facade, and the
`BackupController` owning the `Backup` kind.

## 2. Goals / Non-Goals

**Goals:**
- Ingest every snapshot on an opted-in dataset into the remote restic repo,
  convergently and restart-safely, without a new `Schedule` consumer or a
  framework change to cross-kind event delivery — D-BK1/D-BK2.
- Guarantee that no unbacked snapshot can be destroyed by local retention (the
  hold invariant) — D-BK3.
- Keep the host surface testable: restic and zfs-mount/stream behind narrow
  facades with in-memory fakes, matching the `ZFSHost`/`NvmetHost` discipline —
  D-BK4.
- Repository password write-only at the API and auto-loaded from the OS-disk
  spec-store partition (ADR-0011 keyfile pattern) — D-BK5.
- `backups:read`/`backups:write` RBAC gates and `/v1/backups` routes —
  D-BK6.
- Repository retention converges to the declared policy via `restic forget
  --prune` — D-BK7.
- Status follows the declared contract shape — D-BK8.

**Non-Goals:**
- **Restore flow** — the restore controller/action endpoint is a separate
  change (ADR-0030); this change backs up only.
- **User-managed / exportable repository password** — the co-located password
  keeps the D1 recovery boundary (OS-disk loss loses the password) unchanged.
- **Replication** (`zfs send/receive` to a remote ZFS host) — remains deferred
  per the feature map.
- **Multi-repository** — one repository per system in v1; the API rejects a
  second `Backup` with `409 Conflict`.
- **New `Schedule` consumer** — cadence stays governed by the existing snapshot
  schedules (ADR-0022).
- **App-images dataset** — excluded from the backup set by default as
  reproducible (ADR-0030).
- **Image work** — restic pinning and the `config/var` cache-dir baking are
  `infra` follow-ups; this change makes the paths configurable.

## 3. D-BK1: Observation is a resync-driven sweep, not cross-kind event subscription

On each reconcile pass of the `Backup` resource, the controller lists the
opted-in datasets (from the spec store: `Dataset.spec.backup == true`, minus the
`Backup` spec's `excludes`) and their snapshots, and ingests the not-yet-ingested
ones. The 30 s resync cadence bounds latency; snapshot cadence is typically far
slower, so a 30 s bound is ample.

**Rationale:** D2 delivers events only for the owned kind. A per-snapshot event
path would require either a cross-kind subscription (framework change) or the
backup controller owning `Snapshot` (violates D1). The sweep is idempotent,
restart-safe, and zero-framework-change. *Alternative rejected:* relax the
runtime to allow controllers to subscribe to other kinds' events — lower
latency but a framework change with no v1 need.

## 4. D-BK2: The ingest ledger is the restic snapshot tags; the sweep dedups from `restic snapshots`

Each ingested snapshot is tagged `<dataset>@<name>` (`restic backup --tag`).
Each sweep fetches `restic snapshots --json` once, builds the ingested-ref set
from tags, and skips matches.

**Rationale:** robust across repo re-attach — a re-attached/repopulated
repository dedups correctly, and an empty re-initialized repo cannot silently
skip snapshots (the correctness risk of a local-only ledger). One listing call
per sweep (network-local restic with cache), not per snapshot. Tags double as
restore metadata and as `forget` selectors. *Alternatives rejected:* a ZFS user
property `amberhold:backup=ingested` on the snapshot (cheaper, dies with the
snapshot, but stale-marks data as ingested after a repo wipe → silent data
loss); a spec-store ingest ledger resource (extra resource, contract change,
restart/rollback coupling).

## 5. D-BK3: Hold lifecycle is derived from the sweep

Before ingesting an un-ingested snapshot, apply the external hold
`amberhold:backup` (idempotent — `zfs hold` on an already-held ref is a no-op);
release it only after a successful ingest. The snapshot controller's prune
treats the external hold as protected and never releases it (snapshot.go:144),
so retention can never prune an unbacked snapshot. Hold state is re-derived
from `zfs holds` each sweep, so no extra ledger survives restart.

**Rationale:** the hold is the data-loss guard. Permanent repo failure therefore
accumulates held snapshots and the local dataset fills — surfaced loudly via
status/metrics, the accepted trade-off (ADR-0030: failure retries with backoff,
no data lost). **Consequence:** opting a dataset out (`spec.backup: false`) with
held, un-ingested snapshots leaves the holds in place until ingestion succeeds —
never a silent prune.

## 6. D-BK4: zfs mount/stream primitives extend the storage host facade; restic is a new `ResticHost` facade

- `ZFSHost` gains `MountSnapshotRO(dataset, name, dir)`, `UnmountSnapshot(dir)`,
  and `SnapshotSend(ctx, dataset, name, w io.Writer)` (streams `zfs send`).
  These are ZFS operations → they belong on the existing host, not a new one.
- `internal/backup/` defines `ResticHost` (shell-outs to the pinned binary:
  `init`, `backup`, `forget --prune`, `snapshots`), mirroring the
  smartctl/nerdctl external-tool pattern. Datasets back up from the read-only
  mount; zvols stream via `SnapshotSend` into `restic backup --stdin`.
- The exact read-only mount mechanism (`zfs mount -o ro` vs `.zfs/snapshot/<name>`
  with `snapdir=visible`) is a Linux-integration-surface variant, isolated
  behind `MountSnapshotRO` exactly as the nvmet port-binding symlink variant
  was (design D-B3, 05:282).

**Rationale:** one zfs host keeps all ZFS state access in one place; a separate
`ResticHost` isolates the external binary + password handling.

## 7. D-BK5: The repository password lives in the spec store, write-only at the API

`repository.password` is `writeOnly` (already declared); the API never returns
it. The spec store persists it — the spec-store partition IS the co-located
secret location (ADR-0030: "co-located on the OS-disk spec-store partition",
protected at rest inside the LUKS container, ADR-0011). The controller reads it
from the store and feeds restic via `RESTIC_PASSWORD` env, never argv. It is
never logged or surfaced in status.

**Rationale:** matches the ADR's co-located-secret convention with no new
secret-file plumbing; the API-boundary `writeOnly` (D7 admission/validation)
plus read-stripping (`writeOnlySpecKeys`) keeps it out of every response.
*Alternative rejected:* a separate secret file on the OS-disk partition written
by the API — violates D4 (API is a pure reader).

## 8. D-BK6: RBAC — new `backups:read`/`backups:write` capabilities; backup management is a data-plane role

The capability map (ADR-0018) gains `backups:read` and `backups:write`. Role
assignment: `admin` and `storage-admin` get both (backups are data-plane
ownership); `auditor` and `read-only` get `backups:read`. `share-admin` and
`app-admin` do not. `/v1/backups` routes gate reads on `backups:read` and writes
on `backups:write`, enforced at admission.

**Rationale:** follows the established data-plane gating (storage family uses
`pools:*`, shares use `shares:*`); avoids a new role in the fixed v1 set.
*Alternative rejected:* a new `backup-admin` role — cleaner separation but grows
the fixed role set.

## 9. D-BK7: Retention converges via `forget --prune` after ingest, using the shared `RetentionPolicy`

The `Backup` spec reuses the shared `RetentionPolicy` schema. The controller
runs `restic forget --prune` on the sweep (after any ingests), converging
repository retention to the declared policy. The repo is the durable store;
local snapshot retention stays independent and short.

**Rationale:** one cadence (the sweep) keeps the controller simple; per-ingest
`forget` would run the expensive prune on every snapshot. The sweep's 30 s
resync bounds over-retention.

## 10. D-BK8: Status follows the declared shape

Status reports `state` (`unconfigured`/`initializing`/`ready`/`syncing`/
`error`), `repositoryState` (`absent`/`initialized`), `lastRunAt`, `lastResult`
(`ok`/`error`), `lastError`, `lastSnapshotIngested` (`dataset@name`), and
`optedInDatasets` (count from the spec store). Idempotent no-op writes follow
the D9 pattern (skip the status write when nothing moved).

## 11. Ingestion flow

Each reconcile pass of the `Backup` resource runs the sweep:

```
reconcile(Backup):
  if deleted → release controller-applied holds on opted-in datasets; stop (finalizer)
  if !spec.enabled → gate: no init, no ingest; report state=gated; emit metrics; done
  repo := read repository spec (backend, uri, password from the store)
  if repository absent on host (restic snapshots fails with "no repo"):
      restic init  → repositoryState=initialized (state=initializing during)
  optedIn := Dataset.spec.backup==true  minus spec.excludes        (spec store)
  ingested := tags of `restic snapshots --json`                     (one listing)
  for dataset in optedIn:
      for snap in zfs list -t snapshot -r <dataset>:
          ref := "<dataset>@<snap.name>"
          if ref ∈ ingested:  continue                              (dedup)
          if "amberhold:backup" ∉ zfs holds <ref>:  zfs hold        (D-BK3)
          if dataset is a filesystem:
              dir := mountRoot/<dataset>-<snap>; MountSnapshotRO; restic backup <dir>
          else (zvol):
              SnapshotSend(dataset, snap, w)  →  restic backup --stdin
          on success: zfs release amberhold:backup; ingested += ref; record lastSnapshotIngested
          on failure: keep hold; state=error; lastError; backoff retry
  restic forget --prune <retention>                                  (D-BK7)
  report state=ready / syncing, lastRunAt, lastResult, optedInDatasets
```

The dataset-vs-zvol branch is resolved by the dataset type (filesystem vs
volume) on the host facade; the controller never mounts a zvol as a filesystem.
A snapshot created while backup is disabled is not ingested; a snapshot of a
dataset listed in `excludes` is never held or ingested.

## 12. Metrics (catalog `amberhold.backup.*`, ADR-0008)

The controller emits only the catalog-declared backup metrics, enforced by the
daemon registry:

| Metric | Kind | When |
|--------|------|------|
| `amberhold.backup.state{state}` | gauge | every pass (1 for the current state, 0 for the previous) |
| `amberhold.backup.runs{result}` | counter | every completed sweep (ok/error) |
| `amberhold.backup.run.duration{result}` | histogram | sweep duration (declared buckets) |
| `amberhold.backup.snapshots.ingested{dataset}` | counter | every successful snapshot ingest |
| `amberhold.backup.repository.size` | gauge | `restic stats` (tuning: sweep cadence in v1) |
| `amberhold.backup.errors{reason}` | counter | every ingest/repo failure |

No metric names are emitted outside the declared catalog (D10, ADR-0008).

## 13. Wiring into the daemon (D6)

In `app.New` (step 3, controller registration):

1. build the production storage host (default `ZFSHost`); it now implements the
   snapshot mount/send primitives (D-BK4);
2. construct the `ResticHost` over the configured binary (restic path,
   `--cache-dir` under the writable state partition) and the
   `BackupController(host, resticHost, cfg, registry)` — the store's events +
   resync drive its loop (D2/D3) with the shared 30 s reconcile timeout;
3. register the controller with the runtime;
4. declare the `amberhold.backup.*` metric names in `declareCatalog`;
5. register the `/v1/backups` admission routes (reads `backups:read`, writes
   `backups:write`) and the API handlers — API last (D6 step 6).

`Config.StorageHost`/`Config.ResticHost` remain the test/install seams; tests
inject the fakes and the controller reconciles a seeded `Backup` end-to-end.

## 14. Risks / Trade-offs

- **Permanent repo failure accumulates held snapshots** → dataset fills; the
  hold invariant prefers this over data loss. Surfaced loudly:
  `amberhold.backup.errors`, `state: error`, `lastError`; operator intervention
  (fix repo, or opt out) releases holds.
- **`restic snapshots` listing on the sweep path** → if the repo is unreachable
  the sweep fails closed (nothing ingested, holds retained) — same failure
  surface as ingest itself; no partial state.
- **Sweep cost grows with dataset count** → one `zfs list` per opted-in dataset
  + one `restic snapshots` per sweep; bounded by v1's single-pool scale. Fine.
- **Prune/ingest race** → prune skips held snapshots; the sweep holds before
  ingest, so retention can never destroy a mid-flight snapshot. Converges on the
  next resync.
- **Password in the spec store** → the store is on the encrypted OS-disk
  partition (ADR-0011); DB-local, never returned. Matches D1 posture (OS-disk
  loss loses the password).
- **Mount mechanism variant** → isolated behind `MountSnapshotRO`; fakes test
  the controller; the concrete variant is resolved on the Linux integration
  surface (like the nvmet symlink).

## 15. Migration Plan

No existing data or host state to migrate: this change adds a resource, a
controller, and routes; nothing running depends on backup in v1. Rollback =
archive the change and remove the controller/routes; held snapshots are
released by the deletion finalizer on the `Backup` resource removal. No
spec-store format change (the `Backup` resource is new).

## 16. Open Questions

- The exact read-only snapshot mount mechanism on the pinned ZFS userland
  (isolated behind `MountSnapshotRO`) — resolved on the Linux integration
  surface, like the nvmet symlink variant.
- Whether `restic stats` (for `amberhold.backup.repository.size`) runs on every
  sweep or on a coarser cadence — a tuning detail, no spec impact.