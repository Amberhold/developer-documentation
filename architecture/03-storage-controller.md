# Storage Controller Architecture — the Data-Plane Anchor

> Discovery-phase design. Authored from the `storage-anchor` openspec change,
> extended by `datasets-schedules`, and completed by `storage-slice-completion`
> (pool destruction finalizer, scrub cadence consumer, disk replacement).
> The decisions D-S1–D-S13 here fix how the `Disk`, `Pool`, `Dataset`,
> `Snapshot`, and `Schedule` controllers reconcile host state (zpool/smartctl)
> against the declarative desired-state resources. The ADRs fix the *decisions*
> (per-controller loops ADR-0017, observability ADR-0008, OS-disk exclusion
> ADR-0011, SMART/scrub ADR-0021, topology ADR-0024, shared schedule ADR-0022,
> off-site backup ADR-0030, RBAC ADR-0018); this document fixes the *internal
> mechanics* of the storage slice, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031).

## 1. Purpose

The `core` framework is validated by the `auth` slice alone — a store-only
subsystem that never touches host state. The storage controllers are the first
to reconcile desired state against actual `zpool`/`smartctl` reality,
exercising the idempotency/drift contract (D9) for the first time. This change
is the storage data-plane anchor: the host-state facade, `Disk` discovery +
SMART, the `Pool` controller with immutable topology, the imperative scrub
action, storage CRUD + admission routes, and the `amberhold.storage.*` metric
declarations. `datasets-schedules` (Change B) builds the content layer on top:
the `Dataset`, `Snapshot`, and `Schedule` controllers plus the shared schedule
engine (ADR-0022). Shares, apps, and backup build on it in later changes.

## 2. Goals / Non-Goals

**Goals:**
- A narrow, testable host-state facade (`ZFSHost`) so the storage controllers
  run and unit-test on any machine (macOS dev, no ZFS) — D-S1.
- `Disk` discovery + SMART reconciled against a small desired set, with the
  dedicated OS disk excluded (ADR-0011) — D-S2.
- `Pool` create/import/status with immutable topology enforced (ADR-0024) — D-S3.
- Storage CRUD API + admission routes (ADR-0018) — D-S4.
- `amberhold.storage.*` metrics declared in the catalog (ADR-0008) — D-S5.
- Scrub as an imperative action routed to the owning pool controller (D7,
  ADR-0021) — D-S6.
- `Dataset` lifecycle (create/properties/destroy) reconciled against ZFS, with
  `backup` opt-in reflected to the `amberhold:backup` user property (ADR-0030)
  — D-S8/D-S9.
- A single shared schedule engine (cadence + retention) consumed only by the
  snapshot and scrub controllers (ADR-0022) — D-S7.
- `Snapshot` create/prune on cadence with ZFS holds respected — D-S10.
- `Schedule` validation at the resource boundary via the engine — D-S11.
- `Pool` destruction on resource removal through the deletion finalizer
  (`host.PoolDestroy`, retried on failure) — D-1.
- Scrub *cadence* consumed: due scrub schedules fire `zpool scrub` on the target
  pool through the pool controller's trigger, recording the run in schedule
  status instead of `lastResult: skipped` — D-2.
- Disk replacement (`POST /v1/disks/{id}/replace`) with spare auto-substitution
  or an explicit replacement disk, converging the pool spec member reference so
  a legitimate replacement is never flagged as immutable-topology drift — D-3.

**Non-Goals:**
- Shares, apps, backup (later changes layered on storage).
- The backup controller / restic ingestion / repository `forget --prune` (later
  backup change, ADR-0030) — this change only reflects the opt-in property.
- No cascade-destroy of `Dataset`/`Snapshot` resources when a pool is removed
  (v1: the orphaned children report `pool_missing`/`dataset_missing` until
  cleaned up; a later change can wire child deletion).
- No vdev layout mutation beyond member substitution (raidz/expansion stays
  ADR-0007 migration work).
- Installer-driven pool creation / OS-image boot flows (infra track); the pool
  controller reconciles desired state seeded by the installer (ADR-0007).
- File/block shares, apps, replication, user/group quotas (feature-map
  deferrals).
- Any schedule consumer beyond snapshot and scrub (ADR-0022).

## 3. D-S1: The `ZFSHost` facade — narrow, faked in tests

A small Go interface over pool/device primitives, injected into the storage
controllers. Production implementation wraps `zpool`/`zfs`/`smartctl`/`lsblk`
shell-outs through the D9 single shell-out wrapper (arg slices, never shell
strings) plus udev `by-id` discovery; tests provide an in-memory fake. The
surface stays minimal so the fake is trivial and the interface grows only from
real needs:

```
PoolExists   PoolCreate   PoolImport   PoolStatus   PoolDestroy   PoolScrub
PoolReplace  ListDevices  DeviceHealth SmartInfo
DatasetExists  DatasetCreate  DatasetStatus  DatasetDestroy  DatasetSetProperties
DatasetUserProperty  ListSnapshots  SnapshotCreate  SnapshotDestroy
SnapshotHold  SnapshotRelease  SnapshotHolds
```

- **D9 wrapper**: every external command runs through a `Runner` (`Run(ctx,
  name, args...)`) that maps nonzero exits to a typed `ExitError` (code +
  stderr) so callers can distinguish "no such pool" from real failures. The
  caller's bounded reconcile/action context (runtime `reconcileTimeout`) kills
  a hung binary instead of wedging a controller queue.
- **Production host** (`NewZFSHost`): `zpool list -H` for existence,
  `zpool create/import/destroy/scrub`, `zpool status -j` for state/capacity/
  scan/topology, `smartctl -a -j` / `smartctl -H -j` for health, `lsblk -J` +
  the `/dev/disk/by-id` symlink directory for discovery. Dataset primitives
  wrap `zfs list/create/destroy/set/inherit` and `zfs snapshot/hold/release/
  holds`; property names and values are validated against the declared v1 set
  (D-S8) before any command is built.
- **Alternatives considered** (from `design.md`): faking at the shell-out
  boundary (couples tests to CLI parsing), and `go-zfs` directly with no facade
  (not testable on macOS, couples the slice to a library rather than an owned
  contract). Both rejected. The production implementation is a **thin
  shell-out wrapper** — no new Go dependency — settling the change's open
  question in favor of the wrapper over `go-zfs`.

## 4. D-S2: `Disk` is discovery-first, reconciled to a small desired set

`Disk` spec is nearly all discovery output (ADR-0019: unassigned/spare/failed
disks exist independent of pool membership). The controller inventories
devices via the facade, excludes the dedicated OS disk (ADR-0011), and
converges the discovered set toward the desired `enabled`/`retired` flags.
Read-mostly reconcile: the diff is "which discovered disks to represent"
rather than "what to create".

- **OS-disk exclusion (ADR-0011)**: the OS-disk device id is supplied by the
  installer's layout assignment (ADR-0007) and injected into the slice at
  wiring (`app.Config.OSDiskDevice`, empty in dev). A `Disk` spec referencing
  the OS disk is a hard `Error` (`os_disk_protected`) — never silently added
  to a pool. The OS disk itself is never represented as an assignable data
  disk.
- **Discovery**: `ListDevices` returns by-id identities (falling back to the
  kernel path when no by-id alias exists). A device the spec references but
  the inventory no longer reports is `Degraded`/`device_missing` — identity
  loss is a degraded disk, not a crash (design risk).
- **Membership state**: `state` (`unassigned`/`spare`/`in-use`/`retired`) is
  derived by scanning `Pool` specs for references to the disk id — disks exist
  independent of pool membership, but membership is observable.
- **`enabled: false`** → the disk is still represented but excluded from pool
  membership *and* health monitoring: no SMART poll, health `unknown`,
  reason `disk_disabled`.
- **`retired: true`** → state `retired`, excluded from pool membership, still
  monitored for SMART health.
- **SMART** (`ADR-0021`): the controller polls `SmartInfo` per enabled disk and
  applies spec-declared thresholds (`spec.smart.maxTemperature`,
  `spec.smart.maxErrors`); the worst of raw SMART health and threshold breaches
  is surfaced as `OK`/`degraded`/`failed`. A failed/over-threshold poll writes
  the corresponding `amberhold.storage.disk.*` metrics.
- **Pool-create guard**: the `Pool` controller refuses to create a pool whose
  topology references a missing, disabled, or retired member — reported
  `Pending` (members absent) rather than a partial pool.

## 5. D-S3: `Pool` — create on appearance, adopt on boot, error on topology change

- A `Pool` resource whose named pool doesn't exist → **create** it from the
  declared topology/spares (ADR-0024 v1 vdev types: single, mirror, raidz1,
  raidz2). Creation is deferred to `Pending`/`pool_members_absent` while the
  desired members are not all present in the inventory (startup with pools
  absent: report and retry on a later pass, never fail hard — D3).
- A `Pool` resource whose named pool already exists → **adopt**: import when
  the pool is unimported, report its health/capacity/resilvering.
- Spec topology/spares differ from the existing pool's actual layout →
  `Error`/`topology_immutable`, pool left untouched — no silent no-op, no
  destructive change (ADR-0024: layout immutable after creation).
- The controller resolves `DiskRef.id` → device by reading `Disk` resources
  from the store (no dependency ordering, D3: an unresolvable member is
  `Pending`, retried next loop).
- Status reports state/health/capacityUsed/capacityTotal/resilvering plus the
  observed topology; pool metrics are emitted on every pass.
- **Destruction finalizer (D-1)**: deleting a `Pool` resource destroys the
  underlying zpool via `host.PoolDestroy` (the resource name is the pool name,
  the same convention the content layer uses). A failed destroy returns `Err`
  so the runtime keeps the finalizer pending and retries on a later resync
  (ADR-0031 D9); a missing pool is already destroyed. **Orphan behavior**: the
  pool's `Dataset`/`Snapshot` resources are not cascade-destroyed in v1 — the
  datasets die with the pool and the orphaned resources surface
  `pool_missing`/`dataset_missing` until cleaned up (a later change can wire
  child deletion through the runtime finalizer).

## 6. D-S4: Storage CRUD handlers + admission routes

`pools`/`disks` CRUD follows the auth handler pattern (per-kind handlers over
the store): list/get/create/update/delete plus `PATCH` as RFC 7386 merge-patch.
Admission gates reads on `CapPoolsRead` and writes on `CapPoolsWrite` for
every verb, matching the existing role/capability map (ADR-0018, admin and
storage-admin hold `pools:write`; read-only and auditor hold `pools:read`).
Spec input is validated at the API boundary: topology vdev types, required
`poolName`/`device`, a zpool-safe `poolName` charset (no leading dash, so a
name can never be parsed as a zpool option), and an `id` on every disk/spare
reference (a ref without an id is rejected, never silently dropped into a
partial pool). Admission is the only RBAC enforcement point.

## 7. D-S5: `amberhold.storage.*` metrics declared up front

The metric catalog (`contracts/metrics/catalog.yaml`, ADR-0008) already
declares the storage pool/disk metrics (capacity, health, temperature, SMART,
scrub/resilver progress) under the `storage` subsystem, plus the dataset,
snapshot, and schedule metrics added by `datasets-schedules`; each change
declares the emitted subset in the daemon's `declareCatalog` so the catalog
enforces the storage controllers' emissions like every other subsystem.
Emitted metrics:

| Metric | Kind | When |
|--------|------|------|
| `amberhold.storage.pool.health` | gauge | every pool pass (1 for the current state) |
| `amberhold.storage.pool.capacity.total` | gauge | every pool pass |
| `amberhold.storage.pool.capacity.used` | gauge | every pool pass |
| `amberhold.storage.pool.scrub.progress` | gauge | scrub running (0 when idle) |
| `amberhold.storage.disk.health` | gauge | every disk pass (1 for the current state) |
| `amberhold.storage.disk.temperature` | gauge | every SMART poll |
| `amberhold.storage.disk.smart.errors` | counter | delta of SMART-attributable errors since the last poll |
| `amberhold.storage.dataset.used` | gauge | every dataset pass |
| `amberhold.storage.dataset.referenced` | gauge | every dataset pass |
| `amberhold.storage.snapshot.created` | counter | snapshot created (labels source: scheduled/manual) |
| `amberhold.storage.snapshot.pruned` | counter | snapshot pruned by retention |
| `amberhold.storage.snapshot.create.duration` | histogram | snapshot creation latency |
| `amberhold.storage.schedule.evaluations` | counter | every schedule evaluation |
| `amberhold.storage.schedule.misses` | counter | every evaluation that did not fire |
| `amberhold.storage.schedule.fires` | counter | every evaluation that fired |

`amberhold.storage.disk.resilver.progress` is declared (catalog surface is
complete for the pool/disk scope). The replacement flow (ADR-0024) tracks
resilver in pool status (`resilvering`) and disk/pool metrics; emitting the
resilver-progress gauge is a later observability pass once the facade parses
resilver progress percentages (the zpool scan JSON already carries them).

## 8. D-S6: Scrub is an imperative action (D7)

`POST /v1/pools/{id}/scrub` routes to the pool controller's action map
(`scrub`). The action shells out via the host facade, **never mutates spec**,
and reports the outcome in status (`status.actual.scrub.{state,progress}`) +
metrics (`amberhold.storage.pool.scrub.progress`) — ADR-0021. Actions are
audited exactly like spec writes (D7) and serialized through the controller's
queue, so the action's status write cannot race a concurrent reconcile.

## 9. D-S7: The schedule engine is an evaluator, not a scheduler

`core/internal/schedule` evaluates cadence/retention on each consumer's
reconcile pass rather than owning its own timer loop (ADR-0022, Change B). The
engine is a **pure function**: `due(now, schedule)` and `Prune(now, retention,
snapshots)`. The `Schedule` controller recomputes which resources are due and
hands the snapshot controller its due work through the event bus + resync
(D2/D3) — it never runs a second independent timer that would fight D3.

- **Cadence subset** (syntax settled by Change B): five fields (minute, hour,
  day-of-month, month, day-of-week) with `*`, ranges `a-b`, steps `*/n` /
  `a-b/n`, `,` lists, and the `@daily`-style aliases; both-restricted dom/dow
  uses standard OR semantics. Minute granularity bounds evaluation — the
  resync cadence is far finer than any schedule.
- **Retention** (`RetentionPolicy`): count bound (`keepLast`) and/or age bound
  (`keepFor`, durations `s/m/h/d/w`). `Prune` keeps the newest `keepLast`
  non-held resources and drops resources older than `keepFor`; held resources
  are always kept (D-S10).
- **The `Schedule` controller fires** a due snapshot schedule by creating the
  scheduled `Snapshot` resource (name `auto-<UTC timestamp>`, deterministic
  per minute window — D9 idempotent), which lands on the event bus and drives
  the snapshot controller (D2). The snapshot controller creates the ZFS
  snapshot itself (D-S10).
- **Metrics**: `amberhold.storage.schedule.{evaluations,misses,fires}` are
  emitted per evaluation (D-S5).
- **Alternative considered:** a dedicated scheduler goroutine emitting timer
  events — rejected: reintroduces the ordering/cadence complexity ADR-0017's
  per-controller loops avoid.

## 10. D-S8: Dataset reconcile maps spec properties 1:1 to ZFS properties

`Dataset.spec.properties` is a pass-through map applied by the host facade;
quota/refquota/encryption/recordsize are surfaced as observed values in
status. The immutable field is the dataset `name` (path): a path change on an
existing dataset reports `Error`/`name_immutable` (compared against
`status.wanted`, the last observed spec) rather than a destructive rename,
consistent with pool topology immutability (D-S3).

- **Declared property set**: the facade validates property names against
  quota/refquota/encryption/recordsize and values against a size/mode grammar
  (D9: an unknown or invalid property is never handed to `zfs`). Unknown spec
  properties surface as `Degraded`/`dataset_properties_invalid` with the keys
  listed — never silently dropped, never partially applied.
- **Pool prerequisite**: the named pool must be imported (D3: no dependency
  ordering — a missing pool is `Pending`/`pool_missing`, retried).
- **Delete finalizer**: the framework gained a deletion finalizer path
  (tombstone `ReconcileRequest.Deleted` driven by the delete event, which now
  carries the resource name); the Dataset controller destroys the underlying
  dataset (`zfs destroy -r`) on removal.

## 11. D-S9: Backup opt-in is the storage controller's reflection, not the backup controller's concern

`spec.backup` → `amberhold:backup` ZFS user property lives in the `Dataset`
controller (the resource is the source of truth, ADR-0030). The reflection is
idempotent: the last reflected value is tracked in `status.actual`, and the
first pass always converges (an opt-out removes a pre-existing property).
The future backup controller only reads the property — its selection logic is
independent of dataset reconcile mechanics. Opt-in sets `zfs set
amberhold:backup=true`, opt-out removes via `zfs inherit`.

## 12. D-S10: Snapshot holds are honored by prune

The snapshot controller computes the prune set as "created before the
retention window AND no ZFS hold" (ADR-0022/0030). Holds (e.g. backup
in-flight) protect snapshots from prune; the controller skips held snapshots
and reports them as `held` in status. A spec-declared `hold` tag is applied to
the snapshot after creation. Pruned snapshots are destroyed on the host and
their `Snapshot` resources archived (status `state: deleted` — the controller
never resurrects an archived resource). The resource-name convention is the
full `<dataset>@<name>` reference so the deletion finalizer resolves the host
snapshot.

## 13. D-S11: Schedule controller validates before consuming

The `Schedule` controller validates cadence syntax and policy type at
reconcile via the engine — the single syntax authority; invalid schedules
report `Error`/`schedule_invalid` with `lastError` and are never fed to the
engine. A snapshot schedule without a `target` reports
`Error`/`schedule_target_required` (it can never hand work to the snapshot
controller). Valid schedules report `Ready` with `nextRun`; a due scrub
schedule fires through the pool controller's injected trigger (D-2) and the
schedule status records the run (`lastRun`/`lastResult: ok`, or `lastResult:
error` on failure) — the `lastResult: skipped` consumer gap is retired.

## 14. D-S12: Disk replacement is the one sanctioned topology mutation

`POST /v1/disks/{id}/replace` (ADR-0002, ADR-0024) runs on the `Disk`
controller's action map. It substitutes a failed in-use member with a
replacement disk — the explicit `replacementDiskId` from the payload (validated
enabled/non-retired/non-OS-disk, ADR-0011) or the pool's first attached spare —
via the facade primitive `zpool replace <pool> <old> <new>` (`PoolReplace`),
and **converges the desired state in the same request** so the acknowledged
replacement is never *persistently* reported as immutable-topology drift
(a pool reconcile interleaving between the host replace and the spec write can
report `topology_immutable` transiently; it self-heals on the next pass, and a
spec-write failure after a successful host replace is a loud, retriable
error):

1. Resolve the pool(s) the disk is an in-use member of (a non-member or spare
   disk errors `disk_not_a_member`).
2. Resolve the replacement device (explicit disk or first attached spare,
   `replacement_unavailable` when none is valid).
3. `host.PoolReplace(pool, oldDevice, newDevice)` — the authoritative host act
   (`pool_replace_failed` on error, surfaced verbatim).
4. Converge: the pool spec topology `DiskRef.id` swaps to the replacement disk
   id (the substituted spare leaves the spares), and the replaced `Disk`
   resource is marked `retired: true`. The writes are
   resourceVersion-guarded; a failure after a successful `zpool replace` is a
   loud, retriable error — the action is re-issueable (a re-issue after the
   host replaced a mid-resilver member surfaces the host error verbatim).

The immutable-topology invariant is preserved in spirit (D-S3): **vdev shape,
redundancy, and member count never change**; the physical member backing a slot
is the one imperative substitution, performed through the audited action path.
The action's status reports `Ready`/`disk_replaced` with the old device, the
retired state, and the replacement identity. Resilver progress remains
observable via pool status `resilvering` and the declared
`amberhold.storage.disk.resilver.progress` metric.

## 15. D-S13: Scrub cadence is consumed by the pool controller

The `ScheduleController` stays the cadence authority and owns `Schedule`
status (D1); the pool controller owns `Pool` status and the scrub *execution*
(D-2, ADR-0021). On a due scrub schedule the schedule controller calls the
pool controller's injected `ScrubTrigger` (a narrow `func(ctx, pool) error`
wired at app composition): a no-op when the pool is already scrubbing
(`ScrubState` non-idle — scrubs are never stacked), an error when the pool does
not exist (so no successful run is recorded), otherwise `host.PoolScrub`. On
success the schedule status records `lastRun`/`lastResult: ok`; on failure
`lastResult: error` with the error surfaced; the persisted `lastRun` dedupes
same-window passes (D9). A schedule whose trigger is not wired still reports
`skipped` for a due run. This matches ADR-0021's "scrub triggers are issued via
the imperative action path": the pool controller's trigger IS that imperative
path, reached through the schedule controller rather than the API.

## 16. Wiring into the daemon (D6)

In `app.New` (step 3, controller registration):

1. build the production `ZFSHost` (default runner and binaries);
2. construct the five storage controllers — `DiskController(host,
   osDiskDevice, registry)`, `PoolController(host, registry)`,
   `DatasetController(host, registry)`, `SnapshotController(host, registry)`,
   and `ScheduleController(registry, WithScrubTrigger(pool.ScrubTrigger))` —
   the schedule controller never shells out (the engine is pure, firing goes
   through the store, and the scrub trigger is the pool controller's method,
   D-2) — controllers receive the metric sink (consumer-defined `MetricSink`
   interface, satisfied by the daemon registry) and never build providers
   themselves (D8);
3. register all five with the runtime manager; the store's events + resync
   drive their loops (D2/D3) with the shared 30 s reconcile timeout. The
   schedule controller's due-fire writes a `Snapshot` resource, landing on the
   event bus and driving the snapshot controller (D-S7);
4. declare the storage metric names in `declareCatalog` (D5), including the
   dataset/snapshot/schedule subset.

The API server and admission routes are registered last (D6 step 6). `app`
exposes `Config.StorageHost` and `Config.OSDiskDevice` as test/install seams:
empty `StorageHost` means the production host, and `OSDiskDevice` carries the
installer's layout assignment (ADR-0007/0011). Content-layer admission routes
(`/v1/datasets`, `/v1/snapshots`, `/v1/schedules`) gate reads on `pools:read`
and writes on `pools:write` like the pool/disk family (D-S4).

## 17. Status shapes (contract envelope)

The controllers write the generic contract envelope (D4) — `phase`, `reason`,
`wanted`, `actual`, `observedGeneration`, `conditions` — with the
contract-specific fields carried under `status.actual`:

- **Pool** `actual`: `state` (online/degraded/faulted/offline/unknown),
  `health` (ONLINE/…/REMOVED), `capacityUsed`, `capacityTotal`,
  `resilvering`, `topology`, `spares`, and `scrub` when a scrub is running.
- **Disk** `actual`: `device`, `state` (unassigned/spare/in-use/retired/
  unknown), `health` (OK/degraded/failed/unknown), `temperature`, `capacity`,
  `smartSummary`, `resilverProgress` (reserved). The replace action's status
  adds `pool`, `replacementDiskId`, and `replacementDevice` with `state:
  retired` (D-S12).
- **Dataset** `actual`: `name` (path), `pool`, `mounted`, `used`, `referenced`,
  `compression`, `properties` (observed declared set), `backup` +
  `backupReflected` (D-S9).
- **Snapshot** `actual`: `state` (created/deleted/error), `createdAt`,
  `referenced`, `held` (D-S10), `dataset`, `name`.
- **Schedule** `actual`: `policyType`, `nextRun`, `lastRun`, `lastResult`
  (ok/error/skipped), `lastError`.

Phases: `Ready` when converged, `Degraded` for health/membership/property
anomalies, `Pending` for prerequisites (members absent, pool/dataset missing),
`Error` for protected or invalid specs (OS disk, immutable-topology drift,
name-immutability, invalid cadence) and failed creates.

## 18. Risks / Trade-offs

- **Host facade becomes a leaky abstraction** → keep the interface minimal and
  driven by controller needs; let the fake grow with it; revisit only when a
  controller genuinely needs a new primitive.
- **macOS has no ZFS, so production paths are untested on dev** → the facade
  is exercised through the fake in unit tests; a later infra/CI step (a Linux
  runner with a real pool) is the integration test surface.
- **Disk discovery depends on host udev/by-id naming** → production discovery
  targets stable `by-id` device paths; a missing/inconsistent identity is a
  `Degraded` disk, not a crash.
- **OS-disk exclusion must be airtight** → keyed on the OS-disk device from
  ADR-0011, verified by the installer's layout assignment; a wrongly-included
  OS disk is surfaced as a hard `Error`, never silently added to a pool.
- **`zpool status -j` topology comparison is exact-string** → pools created
  from by-id paths report by-id paths; a pool created with kernel paths (not
  v1) would report drift until imported with the declared layout. Documented
  v1 constraint.
- **Persisted status write amplification** → reconciled status writes follow
  the runtime's coalescing (D5); unchanged resources emit metrics but skip the
  status write (idempotent no-op, D9). Dataset used/referenced and snapshot
  referenced move in metrics, not in status writes.
- **Engine-as-evaluator can fire late** (no dedicated timer) → bounded by the
  consumer's resync cadence; consistent with ADR-0017's "fails and retries"
  model, and cadence granularity makes sub-second accuracy irrelevant.
- **Snapshot prune race with holds** (a hold is set between "check holds" and
  "prune") → the controller re-checks holds immediately before prune and
  treats a conflict as a retry, never a force.
- **Property pass-through is open-ended** → the facade validates property
  names/values against the declared set; unknown properties are reported as
  `Degraded` with a reason, not silently dropped.
- **Schedule validation is duplicated** (controller + engine) → the engine is
  the single syntax authority; the controller surfaces the engine's error
  rather than re-implementing parsing.

## 19. Implementation notes (settled open questions)

- **`go-zfs` vs thin wrapper**: thin wrapper over `zpool`/`zfs`/`smartctl`/
  `lsblk` through the D9 `Runner`; no new dependency (open question in
  `design.md`, no spec impact).
- **Metric names/labels**: settled in §7; tunable without spec impact.
- **OS-disk identity**: injected `OSDiskDevice` (installer layout assignment);
  empty in dev disables the guard (the fake controls it in tests).
- **SMART thresholds**: `spec.smart.maxTemperature` / `spec.smart.maxErrors`
  spec-declared (ADR-0021), with the raw `smartctl` verdict as the baseline.
- **Schedule expression syntax**: the five-field subset plus `@` aliases,
  minute granularity, and `s/m/h/d/w` retention durations (ADR-0022 leaves the
  syntax to the storage change; settled here, tunable without spec impact).
- **`amberhold.storage.schedule.*` metric names**: evaluations/misses/fires
  counters per schedule (Change B open question; settled in §7/§9).
- **Deletion finalizers**: the framework hands a tombstone (`Deleted`) with
  the deleted resource name to the owning controller; Dataset, Snapshot, and
  Pool controllers destroy host state on removal (`zfs destroy -r`, `zfs
  destroy`, `zpool destroy -f`). A failed finalizer stays pending and is
  retried by the resync drain (D-1). The Snapshot finalizer releases its
  controller-applied hold before `zfs destroy` (a held snapshot refuses
  destruction) and never forces past an external hold — a delete of a snapshot
  under backup in-flight retries until the hold clears. Deletion ordering
  matters for datasets: a dataset whose snapshots carry holds should have those
  holds released (snapshot resources deleted/held cleared) before the Dataset
  resource is deleted, because `zfs destroy -r` refuses held snapshots and the
  holds map is per-controller. Removing a pool orphans its Dataset/Snapshot
  resources (they report `pool_missing`/`dataset_missing`); no cascade-destroy
  in v1.
- **Resync cadence / backoff / coalescing**: the runtime defaults
  (`docs/architecture/02-core-daemon.md` §14) apply unchanged.
