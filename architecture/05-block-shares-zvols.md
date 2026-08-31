# Block Shares & Zvols — Completing the Data Plane

> Discovery-phase design. Authored from the `block-shares-nvmeof-zvols` openspec
> change. The decisions D-B1–D-B6 here fix how the `Zvol` controller reconciles
> the `Zvol` resource through the storage host facade (`zfs create -V` /
> `zfs list -t volume` / `zfs destroy`) and how the `BlockShare` controller
> exports zvols over NVMe-oF by writing the kernel `nvmet` configfs target
> directly. ADR-0003 fixes the *decision* (kernel nvmet, serve-only, NQN
> allowlist per export, `allow_any_host` never); this document fixes the
> *internal mechanics*, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031), the storage
> data-plane anchor in `docs/architecture/03-storage-controller.md` (D-S1–D-S13),
> and the shares slice in `docs/architecture/04-shares-controller.md`
> (D-FS1–D-FS8, ADR-0005/0009/0023). The feature map (feature 2 zvols, feature 4
> NVMe-oF) is in `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

Storage (pools/disks/datasets/snapshots/schedules) and file shares are
implemented and green. Block shares — feature 4 — are the remaining data-plane
gap: the `BlockShare` schema existed in `contracts/openapi/v1.yaml` and the
`amberhold.shares.block.*` metrics were catalog-declared, but there was no
controller and no zvol primitive anywhere in the host facade. This slice adds
both: a standalone `Zvol` resource kind (the zvol slice) and the `BlockShare`
controller exporting those zvols over NVMe-oF via a new nvmet host facade.

## 2. Goals / Non-Goals

**Goals:**
- A `Zvol` resource kind reconciled through the storage host facade (create /
  status / destroy), following the `Dataset` controller's lifecycle and
  immutability conventions — D-B1.
- A `BlockShare` controller exporting a zvol over NVMe-oF via a separate
  `NvmetHost` facade that writes nvmet configfs directly, with per-export NQN
  allowlists and `allow_any_host` never enabled (ADR-0003) — D-B2/D-B3.
- One fixed nvmet port bound at install; the controller never creates or
  reconfigures ports — D-B4.
- `/v1/zvols` (storage family: `pools:read`/`pools:write`) and
  `/v1/block-shares` (shares family: `shares:read`/`shares:write`) CRUD through
  the generic storage handlers, with boundary validation — D-B5/D-B6.
- The declared `amberhold.shares.block.exported`/`connections` gauges emitted
  on every pass (including the not-exported 0 series and the finalizer).
- No new Go dependencies: nvmet via configfs writes, zvols via the existing
  `zfs` shell-outs.

**Non-Goals:**
- CHAP authentication — the schema keeps the `authMethod` enum (`none`/`chap`);
  v1 accepts `none` only and rejects `chap` at the boundary (422). CHAP secret
  storage/rotation is a future change — D-B5.
- Transport beyond TCP / any RDMA; no NVMe-oF client (serve-only, ADR-0003).
- nvmet port management — one install-configured port (data plane, ADR-0014);
  no port CRUD — D-B4.
- Zvol snapshot scheduling beyond the shared `Schedule` resource (zfs snapshots
  work on volumes; a future enhancement, no spec impact).
- Zvol resizing — zfs `volblocksize` is immutable and `volsize` is grow-only;
  v1 applies sizes at create and surfaces drift (see §3.1).

## 3. Decisions

### D-B1: zvols are a standalone `Zvol` resource kind (not `Dataset.type`)

The `Zvol` spec carries `pool`, `name` (full dataset path, immutable),
`volsize`, and optional `volblocksize` — properties that do not fit the
`Dataset` property pass-through (`quota`/`refquota`/`recordsize` are
filesystem-only). A `Dataset.type` discriminator would force zvol semantics into
a filesystem-shaped spec and muddle the storage controller's property
validation (D-S8). A standalone kind gets clean validation, its own deletion
finalizer, and the same CRUD/admission pattern as `Dataset`. The v1 consumer is
exactly one (block-shares), but the feature map lists zvols as a v1 storage
primitive regardless. *Alternative considered:* `Dataset.type:
filesystem|volume` — rejected for the property-shape mismatch and controller
coupling.

**Host primitives.** The storage host facade (D-S1) gains four zvol methods,
all through the D9 `Runner` with argument slices:

| Primitive | Command |
|-----------|---------|
| `ZvolExists` | `zfs list -H -o name -t volume` (scan for the path) |
| `ZvolCreate` | `zfs create -V <volsize> [-o volblocksize=<vbs>] <path>` |
| `ZvolStatus` | `zfs list -H -p -t volume -o name,volsize,used,volblocksize <path>` |
| `ZvolDestroy` | `zfs destroy <path>` |

Sizes are validated before any command is built (`ValidVolsize` > 0;
`ValidVolblocksize` power of two, 512 B – 128 MiB), so a wrapped size or an
invalid block size is never handed to zfs.

**Lifecycle.** The `ZvolController` owns the `Zvol` kind (D1): create on spec
appearance, retry a missing pool as `Pending`/`pool_missing` (D3: never a hard
failure), report the observed state, and destroy on removal (deletion
finalizer). The path is immutable, mirroring the `Dataset` name rule (D-S8) — a
changed path is a hard `Error`, never a destructive rename.

**Size drift.** zfs `volblocksize` is immutable after create and `volsize` is
grow-only, and v1 ships no resize primitive (there is no `ZvolSetVolsize` in
the facade). A spec change away from the observed sizes is therefore surfaced
as `Degraded`/`zvol_size_drift` with the observed values in status — reported,
never silently dropped (the D-S8 discipline) — and retried on the resync
cadence. Resizing is a future change.

**Deletion safety (D-B6 guard).** The API rejects deleting a `Zvol` that a
`BlockShare` still references (`422`), and the finalizer re-checks the store:
a still-referenced zvol is left intact so a race can never orphan a dangling
nvmet subsystem. The finalizer also handles the "already gone" case cleanly
instead of requeueing forever.

### D-B2: the nvmet host facade is a separate `NvmetHost`, not an extension of `ZFSHost`

configfs is filesystem state, not a binary contract: the facade needs
`mkdir`/`WriteFile`/symlink primitives plus exactly one `zfs` shell-out to
resolve the zvol's `/dev/zvol/<path>` device path. `SharesHost` extended
`ZFSHost` because SMB/NFS reuse zfs primitives (`sharenfs`, `mountpoint`);
nvmet shares nothing but that single device-path lookup. A standalone
`NvmetHost` interface (production impl + in-memory fake) keeps the "thin
facade, faked in tests" property without bending the `Runner`-based host. The
one shell-out still goes through the D9 `Runner`.

The facade surface (controllers never touch configfs paths):

- `DevicePath` — verifies the volume exists (`zfs get -H -o value volsize`) and
  the `/dev/zvol/<path>` node is present.
- `SubsystemExists` / `CreateSubsystem` / `SetNamespace` — subsystem create and
  namespace bind.
- `Allowlist` / `AllowHost` / `DisallowHost` — per-export NQN allowlist.
- `PortBound` / `BindPort` / `PortID` — the fixed install port binding.
- `Remove` — detach the export (deletion finalizer).
- `ExportsForDevicePath` — find subsystems by their bound device path (the
  finalizer receives only the resource name, the zvol path; D-B6).

### D-B3: direct configfs writes, per-export reconcile

Each `BlockShare` reconcile diffs observed configfs state against the desired
export and writes only its own subsystem/namespace/binding:

```
/sys/kernel/config/nvmet/
  subsystems/<nqn>/
    attr_allow_any_host = 0            (never 1, ADR-0003)
    allowed_hosts/<host-nqn>/attr_nqn = <host-nqn>   (per-export allowlist)
    namespaces/1/
      device_path = /dev/zvol/<pool>/<name>
      enable = 1
  ports/<portid>/subsystems/<nqn>  ->  symlink (bind to the install port)
```

Removal is the deletion finalizer: unbind the port symlink, remove the
namespace + allowlist entries + subsystem (`rmdir`, recursive, children
first — configfs semantics). Writes are idempotent (write-only-when-different,
D9); a half-applied export from a crash is converged on the next pass.
*Alternatives considered:* `nvmetcli restore` whole-config regeneration (mirrors
the samba D-FS5 pattern, but adds a python tool to the image and a
config-format dependency); both rejected in favor of the minimal,
dependency-free direct writes the ADR-0001 image already supports.

**Configfs layout variant (task 3.5).** The port binding was checked against
the Linux integration surface: the kernel configfs layout binds a subsystem to
a port by **symlinking** `ports/<portid>/subsystems/<nqn>` to
`../../subsystems/<nqn>` (kernel nvmet docs, RHEL NVMe-oF guide). There is no
attr-file alternative for subsystem binding — the `addr_*` attrs configure the
port *address* (installer-owned, ADR-0014), not which subsystems it serves.
The controller shape is unaffected either way: the variant is isolated behind
`NvmetHost.BindPort`, so a kernel that wanted an attr write would change one
facade method, never the controller. The facade creates the link with a
**relative** target (`../../../subsystems/<nqn>` from `ports/<portid>/subsystems`);
configfs resolves the link target at creation, and relative port-binding links
are used in the wild, so this resolves to the same directory as the absolute
form shown in the guides.

### D-B4: one fixed nvmet port, bound at install

nvmet requires a port to expose subsystems. v1 assumes a single TCP port on
the data plane, created/configured by the installer (ADR-0014) — the controller
binds subsystems to it and never creates, deletes, or reconfigures ports. A
missing port is a `Degraded`/`port_missing` retry, not a hard failure. The
configured port id is reported in status (`status.actual.portId`) via the
facade's `PortID()`.

### D-B5: CHAP deferred; `authMethod` ships `none` only

The schema declares `authMethod: [none, chap]`; v1 accepts `none` and rejects
`chap` at the boundary (`422`). CHAP adds a secret-storage concern (where
secrets live, per-export rotation, reconcile of shared secrets) that does not
block the data-plane completion. The enum stays so the contract is
forward-compatible.

### D-B6: `BlockShare.zvol` references a managed `Zvol` resource

Mirrors the FileShare→Dataset boundary rule (D-FS7): the zvol must exist as a
`Zvol` resource and is immutable on an existing block share (one share per
zvol). The zvol path is the resource-name convention, exactly like `FileShare`
uses the dataset path — so the deletion finalizer (which receives only the
resource name) resolves the device path and finds the export(s) by it
(`ExportsForDevicePath`), always detaching the actually-exported zvol. NQN
format is validated at the boundary (`nqn.YYYY-MM.reverse.domain:suffix` with a
conservative charset that keeps the NQN a single safe configfs path segment); a
malformed NQN is rejected and never produces a partial export. `allow_any_host`
is never enabled (ADR-0003): the allowlist is enforced per export, and a
changed `allowlist` converges on the next pass.

## 4. Status shapes and metrics

**Zvol status** (`contracts/openapi/v1.yaml` `Zvol`):

- `Pending`/`pool_missing`, `Pending` on a missing pool (retry ~10 s)
- `Degraded`/`pool_probe_failed` | `zvol_probe_failed` | `zvol_status_failed`
  | `zvol_size_drift` (retry)
- `Error`/`name_immutable` | `zvol_create_failed`
- `Ready`/`zvol_created` with `actual.state = "created"` and the observed
  `volsize`/`volblocksize`/`used` byte counts

**BlockShare status** (`BlockShare`): `actual.state` ∈ `active` (contract enum)
with `actual.zvol`, `actual.nqn`, `actual.allowlist`, `actual.portId`;

- `Pending`/`zvol_missing` (retry ~10 s)
- `Degraded`/`device_unavailable` | `export_probe_failed` |
  `allowlist_probe_failed` | `port_probe_failed` | `port_missing` (retry)
- `Error`/`name_immutable` (zvol change) | `export_create_failed` |
  `export_rebind_failed` | `allowlist_update_failed` | `export_detach_failed`
- `Ready`/`share_exported` when the export is confirmed

**Metrics** (catalog `amberhold.shares.block.*`, ADR-0008): the controller
emits `amberhold.shares.block.exported` (1/0, label `share`) and
`amberhold.shares.block.connections` (0 in v1 — connections are not tracked) on
**every** pass — a `Pending`/`Degraded`/`Error` share declares its not-exported
0 series, and the deletion finalizer zeroes the series so `/metrics` never
keeps a stale 1 (the registry has no series removal). Zvols emit no gauges —
there is no catalog zvol metric in v1.

## 5. Daemon wiring

`app.go` (D6 step 3) wires both slices:

```
storage.ZvolController(storageHost)      // owns the Zvol kind
blockshares.NvmetHost (configfs)         // production: NewNvmetHost()
blockshares.BlockShareController(nvmetHost, registry)  // owns BlockShare kind
```

`declareCatalog` registers `amberhold.shares.block.exported` and
`amberhold.shares.block.connections`. Admission (`routes()`) gates
`/v1/zvols` reads on `pools:read` and writes on `pools:write` (storage family),
and `/v1/block-shares` reads on `shares:read` and writes on `shares:write`
(ADR-0018), matching the file-shares surface. The API boundary validates:
`Zvol` specs (`pool`/`name`/`volsize`/`volblocksize`) and `BlockShare` specs
(zvol references a managed `Zvol`, well-formed NQNs, `none`-only auth, one
share per zvol), and the zvol delete guard protects referenced zvols.

## 6. Risks / Trade-offs

- **configfs is a new facade primitive class** → [risk] the facade pattern
  bends from "shell-out wrapper" toward direct filesystem state → [mitigation]
  all configfs access is isolated behind `NvmetHost` with the same
  narrow-interface/fake-test discipline as `ZFSHost`; controllers never touch
  configfs paths.
- **nvmet port lifecycle assumed, not owned** → [risk] a missing/misconfigured
  port leaves exports unserved → [mitigation] `port_missing` Degraded + retry;
  the installer owns the port (ADR-0014); this is recorded here (§3).
- **`/dev/zvol/<path>` resolution depends on zfs device nodes** → [risk] a
  zvol without a device node can't be exported → [mitigation] the facade
  resolves the path via `zfs` and reports the export
  `Degraded`/`device_unavailable`, retriable.
- **zvol destruction races an active export** → [risk] deleting a `Zvol` that
  a `BlockShare` still references orphans a dangling subsystem → [mitigation]
  the boundary guard rejects deleting a referenced zvol (D-B6), and the
  deletion finalizer re-checks on the host.
- **zvol size drift** → [note] zfs `volblocksize` is immutable and `volsize`
  grow-only; v1 applies sizes at create and surfaces drift as
  `Degraded`/`zvol_size_drift`. Resizing is a future change.
- **SMART/resilver observability gap** → [note]
  `amberhold.storage.disk.resilver.progress` stays declared-but-unemitted in
  this change (the host facade still doesn't parse resilver percentages,
  ADR-0021); tracked as a follow-up, not a blocker.

## 7. Migration Plan

No existing data or host state to migrate: this change only adds resources and
routes. Rollback = archive the change and remove the new controllers/routes; no
running subsystem depends on nvmet exports in v1.

## 8. Open Questions

- Exact nvmet configfs layout variant on the pinned kernel (symlink vs attr
  bind path for port binding) — **resolved on the Linux integration surface:
  symlink** (see §3, task 3.5). The variant is isolated behind
  `NvmetHost.BindPort`.
- Whether `Zvol` snapshots ride the existing Snapshot controller (`zfs
  snapshot` works on volumes) or stay out of v1 — a future enhancement, no spec
  impact.
- CHAP authentication mechanics (secrets storage, per-export rotation) — a
  future change; v1 rejects `chap` at the boundary (D-B5).