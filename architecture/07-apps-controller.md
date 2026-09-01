# Apps Controller — containerd Workloads with ZFS-backed Volumes

> Discovery-phase design. Authored from the `apps-containerd-compose` openspec
> change. The decisions D-A1–D-A9 here fix how the `App` controller reconciles
> the `App` resource (feature 5) — docker-compose and single-image workloads on
> containerd via nerdctl, with named compose volumes backed by per-volume ZFS
> datasets and a dedicated per-stack UID. ADR-0004 fixes the *decision* (full
> docker-compose on containerd, dataset-per-volume by default, dedicated UID per
> stack); this document fixes the *internal mechanics*, built on the
> framework-first runtime in `docs/architecture/02-core-daemon.md` (D1–D10,
> ADR-0031), the storage data-plane anchor in
> `docs/architecture/03-storage-controller.md` (D-S1–D-S13), the shares slice in
> `docs/architecture/04-shares-controller.md` (D-FS1–D-FS8, whose identity
> service reserves the app UID space), the block-shares/zvols slice in
> `docs/architecture/05-block-shares-zvols.md` (D-B1–D-B6), and the backup slice
> in `docs/architecture/06-backup-controller.md` (D-BK1–D-BK8). The feature map
> (feature 5 app workloads, feature 9 observability) is in
> `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

Storage, shares, block shares, and backup are implemented and green. Feature 5 —
app workloads on containerd (ADR-0004) — is the last major data-plane slice: run
full docker-compose stacks on the appliance with app data on ZFS (a dataset per
volume by default), each stack owned by a dedicated UID from the shared identity
space, so app data participates in the snapshot/backup story (features 2 and 15)
and survives OS-disk loss (ADR-0011). The contract surface is fully declared —
the `App` resource, the `amberhold.apps.*` metric catalog, and `apps` in the
`api-contract` resource set — but there is no controller and no nerdctl/containerd
primitive in the host facades. This slice adds both: a `NerdctlHost` facade over
the pinned nerdctl binary, the compose translation pipeline, and the
`AppController` owning the `App` kind.

## 2. Goals / Non-Goals

**Goals:**
- A `App` controller (D1) reconciling compose and single-image workloads on
  containerd through a `NerdctlHost` facade — D-A1.
- Named compose volumes backed by ZFS datasets under `volumeDataset/volumes/<name>`,
  with explicit rules for anonymous volumes and user bind mounts — D-A2/D-A3.
- A dedicated stack UID per app from the shared allocation space keyed
  `app:<name>`, owning the app's volume mountpoints — D-A4.
- An ownership-based deletion model: the controller destroys what it derived and
  preserves what it only referenced — D-A5.
- `/v1/apps` API + admission gated on `apps:*`, and the declared
  `amberhold.apps.*` metrics emitted — D-A7/D-A8.

**Non-Goals:**
- zvol-per-volume escape hatch (ADR-0004 mentions it; the contract does not
  express it — recorded as a deferred decision, not silently dropped).
- Updates / A-B (feature 1) and networking (feature 10) — separate slices.
- Web-UI and CLI clients for apps.
- Registry auth / pull policy beyond what the compose spec itself expresses.
- Image storage management — images live on the `app` role dataset (ADR-0007)
  with nerdctl managing the store; no image GC controller in v1.
- Any compose feature nerdctl does not implement — nerdctl is the compatibility
  boundary (ADR-0004), not core.

## 3. D-A1: One `App` controller, a `NerdctlHost` facade, two deployment modes

The `App` controller owns the `App` kind (D1). All host interaction goes through
a `NerdctlHost` facade (nerdctl `compose config`/`up`/`down`, `ps`, `restart`,
`run`) over the D9 `Runner` — the external-tool pattern of ADR-0001/0006, nerdctl
pinned in the image. The controller branches on the spec: `compose` → the compose
pipeline (D-A2/D-A3); `image` → a thin single-container deployment (`nerdctl run`
with the app's volume dataset mounted). Exactly one mode must be set — validated
at the API boundary, `Error`/`invalid_spec` otherwise.

**Rationale:** ADR-0004 mandates "full docker-compose semantics"; a second
mechanism backend is not warranted because single-image is expressible as a
degenerate compose, but the contract exposes it as a distinct field. Alternatives
rejected: `image` as an implicit one-service compose (hides the mode distinction
the contract makes); a direct containerd client (violates the external-tool
pattern and the pinned-tool posture of ADR-0001).

## 4. D-A2: Compose translation — normalize, walk, materialize, override

The controller never runs the user's compose verbatim. Pipeline per pass:

```
user compose ──► nerdctl compose config      (normalize: env, defaults, extends)
        │
        ├─► walk named volumes (top-level volumes map)
        │        └─► ensure dataset volumeDataset/volumes/<name> (D-A3), chown (D-A4)
        ├─► generate override compose (derived state, regenerable):
        │        volumes.<name>.driver_opts = {type: none, device: <mountpoint>, o: bind}
        │        + default user: <stackUID> per service (D-A4)
        └─► nerdctl compose up -f <generated base> -f <override>   (recreate on change)
```

Rules:
- **Named volumes → datasets.** Each named volume gets a dataset under
  `volumeDataset/volumes/<name>`.
- **Anonymous volumes → nerdctl default store.** Anonymous volumes receive hashed
  names only *after* `up`, so they cannot be pre-materialized as datasets. This is
  an explicit, documented rule, not a silent gap; their data still lives on the
  `app` dataset at coarse granularity.
- **User bind mounts pass through.** They already point where the user chose; the
  controller does not intercept them.
- **Change policy is recreate**: any `compose` change → `down` + `up`. Data lives
  in the datasets, so recreation is cheap and safe. No partial diffs in v1.
- The generated base + override live under `volumeDataset/` (relative paths
  resolve there) and are fully regenerable — derived state per ADR-0002.

**Rationale:** `nerdctl compose config` offloads compose semantics to the tool
that owns them, keeping `core` free of a compose-parser dependency; the override
keeps the user's service definitions intact while intercepting only the volume
map. Alternatives rejected: full transpile to a rewritten compose (more control
but more drift and debugging surface); passthrough with coarse ZFS backing (fails
ADR-0004's per-volume dataset consequence).

## 5. D-A3: Volume layout and ownership of derived datasets

`volumeDataset` is a **reference to an existing dataset** (declared as a
`/v1/datasets` resource or present on host), not created by the `App` controller.
Derived children are `volumeDataset/volumes/<name>`:

```
volumeDataset (referenced, NOT App-owned)
├── volumes/data    (derived dataset, mountpoint volumeDataset/volumes/data)
├── volumes/cache   (derived dataset)
├── compose.yaml    (generated, derived)
└── override.yaml   (generated, derived)
```

Mountpoints nest under `volumeDataset` (ZFS children inherit the parent
mountpoint — no legacy-mount fiddling). App reconcile Pends (`dataset_missing`,
FileShare pattern, D-FS1) until `volumeDataset` exists on host.

**Rationale:** avoids dual-owner ambiguity (D1) — a dataset path created by both
the `Dataset` controller and the `App` controller would split ownership and status
writing. Alternatives rejected: the App controller implicitly creating
`volumeDataset` (ownership split, breaks the single-status-writer contract); flat
`<volumeDataset>/<vol>` children (reserved-name collisions).

## 6. D-A4: Stack UID — `app:<name>` from the shared space

Each app draws a UID from the shared allocation space keyed `app:<name>` — the
reserve in `04-shares-controller.md` D-FS2, now consumed. The `app:` prefix keeps
app principals distinct from users in the same 1000–65534 space; numeric collision
is impossible while both are held (`nextFree` skips in-use numbers). The
controller chowns each derived volume mountpoint to the stack UID on create and
**re-converges ownership on resync** (mountpoint ownership is host state, not a
ZFS property — it drifts). The generated override sets `user: <stackUID>` per
service *by default*, overridable per service (`user: 0` for privileged apps).
"Never root by default" (ADR-0004) is satisfied at the identity/ownership layer
regardless of a given app's runtime user.

**Rationale:** reuses the existing allocator and keeps the identity model
consistent with shares. Alternatives rejected: a separate app UID range
(contradicts the D-FS2 shared-space decision); lazy allocation inside the App
controller (fragments the space the identity service already owns).

## 7. D-A5: Deletion ownership model

The controller destroys what it derived and preserves what it only referenced:

```
DELETE App:
  1. nerdctl compose down                     (no -v → data untouched on disk)
  2. remove generated compose + nerdctl project state
  3. destroy volumeDataset/volumes/* datasets (App-owned; idempotent if gone)
  4. identity.Release("app:<name>")
  → volumeDataset root + its data PRESERVED
```

The idempotent "nothing to destroy" path mirrors the storage deletion finalizers
(`pool.go`/`dataset.go`). Because derived datasets are children of `volumeDataset`,
deleting the Dataset resource first destroys them recursively (ZFS) — the App's
finalizer then no-ops, so the ordering hazard self-resolves. Destroy-on-delete is
consistent with the Dataset/Pool finalizer precedent; app volume data is
recoverable from snapshots of the backing dataset (features 2 + 15). The admin
who wants data preserved deletes the Dataset resource instead of the App, or
relies on the snapshot `Schedule`.

**Rationale:** ownership, not a preserve/destroy preference, settles this — every
storage controller already destroys what it owns on delete. Alternatives
rejected: preserve + a new imperative `destroy` action (new contract surface,
diverges from precedent); destroying the `volumeDataset` root too (destroys a
resource the App does not own).

## 8. D-A6: `enabled: false` — stop, keep everything

`enabled: false` → `nerdctl compose down` only; datasets, UID, compose, and
project state are retained. `state` reports `stopped`. `enabled: true` resumes
with `up` (cheap recreate). Deletion semantics are independent of the `enabled`
value.

## 9. D-A7: API + admission follow the slice pattern

`/v1/apps` CRUD follows the per-kind handler pattern; reads gate on
`CapAppsRead`, writes on `CapAppsWrite` (new `RoleAppAdmin`, ADR-0018). Boundary
validation: exactly one of `compose`/`image`, non-empty `volumeDataset`
referencing an existing dataset, `enabled` boolean. Invalid specs report
`Error`/`invalid_spec`, never a partial deployment.

## 10. D-A8: Metrics follow the catalog

The controller emits the already-declared `amberhold.apps.state` (gauge, labels
app+state), `amberhold.apps.restarts` (counter, app), and
`amberhold.apps.reconcile.duration` (histogram, app) on each pass; no unregistered
names (D10, ADR-0008).

## 11. D-A9: App-dataset prerequisites

Two prerequisites gate the reconcile: the `app` role dataset (images + compose
state, ADR-0007) and the per-app `volumeDataset`. Both are probed through the host
facade; absence → `Pending`/`dataset_missing` with retry (D3: never a hard
failure). The `app` dataset is installer-configured and stubbed in tests via the
fake facade.

## 12. Reconcile flow

```
App reconcile, one pass:
  validate mode + preflight volumeDataset        (D-A1/D-A9)
  allocate/verify uid app:<name>                 (D-A4)
  nerdctl compose config → walk named volumes    (D-A2)
  ensure volumeDataset/volumes/<vol> datasets,   (D-A3)
    chown mountpoints to stack uid               (D-A4)
  generate override (bind driver_opts + user:)   (D-A2/D-A4)
  nerdctl compose up -f base -f override         (D-A2)
  write status {state, containers, imageDigest}  (D4)
  emit amberhold.apps.* metrics                  (D-A8)

App delete (finalizer):
  nerdctl compose down (no -v)                   (D-A6/D-A5)
  remove generated + project state               (D-A5)
  destroy derived volume datasets                (D-A5)
  identity.Release("app:<name>")                 (D-A5/D-A4)
```

## 13. Wiring into the daemon (D6)

`AppController` + `NerdctlHost` are constructed in `app.New` step 3 (after storage,
shares, block shares, and backup), sharing the framework's 30s reconcile timeout
and the identity `Allocator`. `/v1/apps` routes are added to the API server with
the generic action route already covering any future imperative actions.

### Implementation-pinned mechanics

The code pins the mechanics the decisions above leave open; the design does not
change, only the detail:

- **Resource-name convention (D-A5):** the App resource name is the
  `volumeDataset` path (spec-derived at the API boundary, mirroring
  `FileShare` name = dataset). The deletion tombstone carries only kind/id/name,
  so the name must locate every derived child — `volumeDataset/volumes/<name>`,
  the generated files on the mountpoint, and the `app:<name>` UID key. One App
  per `volumeDataset` is enforced at the boundary (the design's
  `volumeDataset/compose.yaml` + `volumes/<name>` layout collides otherwise),
  and `volumeDataset` is immutable on update.
- **Compose project / container identity:** the nerdctl `-p` project name and
  the single-image container name are the store-assigned resource id — stable
  across reconciles, valid for nerdctl's charset, unique, and carried by the
  tombstone. The app name (the `volumeDataset` path) keys the UID allocation
  and the derived datasets instead.
- **Generated files:** the base (`compose.yaml`) and the override
  (`override.yaml`) live at the `volumeDataset` mountpoint root; derived
  datasets nest under `volumeDataset/volumes/<name>`.
- **Facade surface (D-A1):** `ComposeConfig/Up/Down/Ps`, `Restart`, and the
  single-image `Run`/`Stop`/`Start`/`Rm`/`Ps` primitives — the container-level
  set is what `enabled: false` (D-A6: stop/start) and deletion (D-A5: rm) need
  for the first-class image mode.
- **Image-mode mount target:** `spec.image` has no mount field (the thin-mode
  open question), so the `volumeDataset` mounts at `/data` in the container and
  the stack UID is passed as `--user` (D-A4: never root by default at the
  runtime layer too).
- **Derived-dataset destruction on delete (D-A5):** the finalizer destroys the
  volume set listed by the last generated override (the current derived set).
  Datasets orphaned by removing a volume from compose before deletion are
  admin-visible and recoverable via the Dataset resource; a documented v1 edge.

## 14. Risks / Trade-offs

- **[nerdctl compose divergence] `driver_opts` bind mounts for named volumes are
  honored inconsistently across compose implementations** → primary mechanism is
  the bind `driver_opts` override; fallback is rewriting per-service volume
  references to explicit `type: bind` mounts. Verified against the pinned nerdctl
  version on the integration surface.
- **[compose merge semantics] Override merging (list concatenation vs
  replacement) varies across versions** → v1 regenerates the base+override pair
  from normalized output each pass and validates the merged result (`nerdctl
  compose config` round-trip) before `up`; a failed validation retries, never
  deploys a half-merged project.
- **[permission mismatch] Apps that expect host-root writes hit UID-owned
  mountpoints** → the override's default `user: <stackUID>` is per-service
  overridable; image-level UID expectations are a documented v1 constraint (same
  posture as the shares slice's `getpwnam` limitation).
- **[destroy-on-delete data loss] Spec delete destroys volume data** →
  recoverable from snapshots (features 2 + 15); consistent with the storage
  finalizer precedent; the admin preserves data by deleting the Dataset resource
  instead of the App.
- **[anonymous volumes unbacked per-volume] Their data lives in nerdctl's default
  store** → on the `app` dataset at coarse granularity; documented rule so it
  reads as a decision, not a gap.
- **[containerd/nerdctl moving target] Compose translation breaks on version
  changes** → nerdctl pinned in the image (ADR-0001); the facade isolates the
  version surface.

## 15. Migration Plan

Greenfield discovery-phase feature: no existing app state to migrate. Deployment
is a normal `core` change — new controllers registered in `app.New`, routes added
to the API server. Rollback is spec-store revert (ADR-0013); derived state
(datasets, generated compose, nerdctl project) is regenerable or destroyed by the
finalizer, with no host-state residue beyond the preserved `volumeDataset` root.

## 16. Open Questions

- **Image-only mode surface** — `spec.image` + `volumeDataset` is deliberately
  thin (no env/ports in the schema). Whether single-image apps need more surface
  is a contract question, deferrable without changing this slice's approach.
- **nerdctl `driver_opts` bind support on the pinned version** — implementation
  detail, verified on the integration surface; the fallback (D-A2) is already
  designed.
- **`user:` override interplay with images that hard-set USER** — documented
  constraint; no spec impact.
- **`app` role dataset provisioning** — installer-time (ADR-0007), outside this
  slice; the controller treats its absence as `Pending`.