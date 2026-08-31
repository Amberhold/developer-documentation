# Shares Controller Architecture — the First Data-Plane Consumer

> Discovery-phase design. Authored from the `file-shares-smb-nfs` openspec
> change. The decisions D-FS1–D-FS8 here fix how the `FileShare` controller
> reconciles the `FileShare` resource against two mechanism backends (SMB via
> samba, NFS via ZFS `sharenfs`) and how the shared identity service allocates
> the UIDs both mechanisms depend on. The ADRs fix the *decisions* (identity
> model ADR-0005, NFS via sharenfs ADR-0009, multi-protocol combos ADR-0010,
> samba config on config/var ADR-0023, RBAC ADR-0018, observability ADR-0008);
> this document fixes the *internal mechanics* of the shares slice, built on
> the framework-first runtime in `docs/architecture/02-core-daemon.md` (D1–D10,
> ADR-0031) and the storage data-plane anchor in
> `docs/architecture/03-storage-controller.md` (D-S1–D-S13).

## 1. Purpose

Storage (pools/disks/datasets/snapshots/schedules) and auth
(users/roles/sessions/tokens) are implemented and green. The `FileShare`
controller is the first feature that makes the appliance usable: it exposes a
dataset over SMB and/or NFS (ADR-0010 supported combo) while exercising the
dormant identity link (ADR-0005) — UIDs allocated from one shared space,
materialized as dataset ownership and samba tdbsam entries with no
`/etc/passwd` writes (ADR-0001 RO root).

## 2. Goals / Non-Goals

**Goals:**
- A `FileShare` controller that reconciles the `FileShare` resource against two
  injected mechanism backends (SMB + NFS), matching the supported
  multi-protocol combo — D-FS1.
- An identity **service** (not a controller) owning the single shared UID
  allocation space, keeping `status.uid` owned by the `User` controller (D4) —
  D-FS2.
- SMB grants materialized as samba config + tdbsam entries on `config/var`
  with no host account writes — D-FS3/D-FS5.
- NFS host/IP grants expressed via the existing `options` bag with no contract
  schema change — D-FS4.
- Dataset-root ownership set to the linked user's UID when a grant is
  materialized — D-FS6.
- `/v1/file-shares` API + admission, and the declared
  `amberhold.shares.file.*` + `amberhold.identity.uid.*` metrics emitted —
  D-FS7/D-FS8.

**Non-Goals:**
- Block shares (NVMe-oF, ADR-0003) — a separate change requiring a zvol slice.
- App workload UID allocation — the identity service reserves the space but app
  UIDs land with the apps change (ADR-0004).
- Backup/restore flows (ADR-0030) and the restic controller.
- NSS / system-account materialization under `/etc/passwd` — RO-root constraint
  (ADR-0001); UIDs resolve via the spec store + samba tdbsam only.
- OIDC identity links (ADR-0029).
- Any protocol combination beyond SMB + NFS on one dataset (ADR-0010).
- Recursive `chown`; v1 applies to the dataset root only.

## 3. D-FS1: One `FileShare` controller, two injected mechanism backends

The `FileShare` controller owns the `FileShare` kind (D1). SMB and NFS are not
controllers — they are mechanism backends injected into the controller:

```
FileShareController(host, identity, smbBackend, nfsBackend, registry)
```

Reconcile branches on `spec.protocols`: the NFS backend builds the grant from
`spec.options.nfs.hosts` and drives `zfs set sharenfs` via the host facade; the
SMB backend builds the samba config + tdbsam grants from `spec.access` (NAS
users). Status reflects per-protocol result, and the controller is the single
status writer (D4).

The controller reconciles each `FileShare` resource, but the **samba config is
one file** derived from the whole spec store: every pass rebuilds the desired
config from all SMB-enabled `FileShare` resources (single-sourced, ADR-0002),
so a protocol change or deletion converges the sections of every share on the
next pass. The NFS side is per-dataset (`sharenfs` is a dataset property), so
the NFS backend handles only the resource being reconciled. A share whose
dataset does not exist on the host is **excluded from the regeneration** — it
reports `Pending`/`dataset_missing` on its own reconcile, and its missing
mountpoint must not poison the config rebuild of every other healthy share.

**Rationale:** ADR-0010 allows SMB + NFS concurrently on one dataset from one
spec; two controllers owning the same kind would violate D1 (one owner writes
status). Alternatives rejected: two controllers over forked `smb-shares`/
`nfs-shares` kinds (diverges from the single `FileShare` contract resource);
inline branching (tangles two mechanisms in one file). The feature-map
skeleton's separate "SMB controller"/"NFS controller" lines are superseded (the
docs task amends that diagram).

## 4. D-FS2: Identity service — a UID ledger, not a D1 controller

`core/internal/identity` exposes an `Allocator` that hands out and reclaims
UIDs from a shared space (1000–65534, with the low range reserved for the
appliance's system accounts), backed by the spec store as a singleton
`UidLedger` resource so allocations survive restart. It is injected into the
`User` controller (allocate on create → `status.uid`, reclaim on delete via the
deletion finalizer) and the `FileShare` controller (resolve a linked user's UID
when materializing grants).

- **Allocation is idempotent** (D9): `Allocate(username)` returns the existing
  UID when already allocated; a converged user reconcile is a store read with
  no write. Optimistic concurrency (D5) protects concurrent allocators (the
  `User` controller's serialized queue and the `FileShare` controller share the
  ledger) — a resourceVersion conflict re-reads and retries instead of
  double-allocating.
- **Reclamation on deletion**: the `User` resource name mirrors the username
  (the API derives it), so the deletion finalizer's tombstone carries the
  principal whose UID to release. Released UIDs return to the space (the probe
  reuses them after wrapping — immediate low-UID reuse is not guaranteed, which
  avoids resurrecting a UID still referenced by stale caches).
- **App stacks share the space** (ADR-0004): the allocator is a service with no
  controller ownership, so the apps change draws from the same ledger without
  reworking it.
- **Metrics**: `amberhold.identity.uid.{allocated,released}` counters
  (catalog-added by this change), emitted by the allocator.

**Rationale:** `status.uid` lives on the `User` resource, whose owner is the
`User` controller (D4) — a separate D1 controller writing it would break
single-status-writer. A resource-less ledger is not a reconcile target, so it
is a service, not a controller. Alternatives rejected: folding allocation into
the auth `User` controller (bloats auth and fragments the space before apps
need it); lazy allocation inside `FileShare` (rework when the space goes
shared). ADR-0005's "identity controller" wording is amended by the docs task.

## 5. D-FS3: POSIX UID materialization without `/etc/passwd` writes (RO root)

No system account is created on the host. The UID lives in the spec store (via
the identity service) and in samba's tdbsam passdb on `config/var`. Dataset
root ownership is set by the `FileShare` controller to the linked user's UID
(`chown`) so both NFS clients (via `idmapd` domain mapping, ADR-0005) and SMB
(via tdbsam) see consistent ownership. `getpwnam()` for NAS users is a
documented v1 limitation: tools that require a system account resolve nothing.
The future path is a custom NSS module (`libnss_amberhold`) reading the spec
store, baked into the image — deferred.

**Rationale:** ADR-0001 forbids writing under `/`; `/etc/passwd` is RO. tdbsam
holds username↔UID independently of the system passwd, and samba resolves the
mapping itself. Alternatives rejected: bind-mounting a generated `/etc/passwd`
(violates the RO-root convention); a custom NSS module (real systems
engineering, must be baked into the image — deferred, recorded as the future
path).

**Integration note (pdbedit):** `pdbedit -a` normally prompts for a password
and resolves the Unix user via `getpwnam`. The host primitive (`PdbeditUpsert`)
runs `pdbedit -a -u <username>` through the D9 `Runner` with an argument
slice; the interactive prompt and the missing system account are integration
surface concerns — the design open questions below record the expected Linux
verification (a stdin-fed `-t` variant, or a placeholder passdb password) once
the D9 runner gains a stdin writer. The unit surface (arg slices, typed
`ExitError`) is fully covered over the fake runner.

## 6. D-FS4: NFS grants live in the existing `options` bag — no contract delta

The NFS client allowlist is `spec.options.nfs.hosts` (e.g.
`["192.168.1.0/24", "host.example.com"]`), consumed only when `protocols`
includes `nfs`. The `FileShare` schema's `options` object is already
`additionalProperties: true`, so **no schema change is required** (verified in
`contracts/openapi/v1.yaml` by the contracts task). The NFS backend builds the
`sharenfs` value from the list (`rw=<host1>:<host2>...`; an empty list exports
to all clients, the `sharenfs=on` semantics) and applies it via the host
facade:

- `zfs set sharenfs=<value> <dataset>` to apply,
- `zfs inherit sharenfs <dataset>` to clear when NFS is dropped from
  `protocols` or the share is deleted,
- `zfs get -H -o value sharenfs` to observe — a converged pass is a read with
  no host mutation (D9). There is **no `/etc/exports` writer** (ADR-0009).

Host grants are validated at the API boundary (`ValidNFSHost`: IP, CIDR, or a
conservatively-validated hostname); an invalid grant reports
`Error`/`grant_invalid` and never produces a partial export.

**Rationale:** keeps the contract surface stable; `options` is the designated
minimal-in-v1 extension point. Alternatives rejected: a dedicated `hosts` field
(schema change for one mechanism's input); typed `access` entries (conflates
user grants with IP grants).

## 7. D-FS5: SMB backend — generated config + tdbsam + reload

The SMB backend regenerates the samba config on `config/var` (the `/etc/smb.conf`
symlink target, ADR-0001) with one `[share]` section per SMB-enabled
`FileShare`, expressing `valid users` / `read only` per grant (ADR-0023).
Grants reference NAS users by username; the backend converges the tdbsam
passdb to the **union of granted users across all shares** — a user granted on
several shares keeps their entry while any share references them, and a
removed grant or deleted user loses their entry in the same reconcile cycle.
Reload is `smbcontrol all reload-config` (no service restart; samba runs from
the image), issued **once per pass only when something changed** — a converged
regenerate is a no-op (D9), verified by config diff against the observed file
and the observed `pdbedit -L` user list.

The generated config is fully owned by the daemon (derived state, ADR-0002):
a minimal `[global]` stanza plus the share sections, written atomically (temp +
rename) so a crashed reconcile never leaves a torn config samba would refuse.
Section names derive from the resource name (the dataset-path convention) with
path separators turned into dashes (`tank/photos` → `[tank-photos]`).

**Rationale:** samba sections are the only place per-user access control can be
expressed (ADR-0023); single-sourcing share state in the spec store makes the
config regenerable at any time.

## 8. D-FS6: Dataset ownership is the `FileShare` controller's job

When a grant to a linked user is materialized on a dataset, the `FileShare`
controller chowns the dataset root to that user's allocated UID (resolved via
the identity service) through the host facade (`chown uid:uid <mountpoint>`,
the mountpoint resolved via `zfs get mountpoint`). The first resolved grant
wins (deterministic order); a chown host failure is `Degraded`/`chown_failed`
with a timed retry — the export itself is not blocked. Storage stays
ownership-agnostic: the `Dataset` controller never couples to identity state.

**Rationale:** ownership is a consequence of sharing, not of dataset creation.
Alternatives rejected: the `Dataset` controller setting ownership at create
(couples storage reconcile to identity); installer-assigned ownership (wrong
layer, shares are dynamic).

**v1 scope:** chown applies to the dataset root only, never `-R`; recursive
ownership is a future path. Multiple `FileShare` resources over the same
dataset are prevented at the API boundary (one share per dataset; the
resource-name convention derives the name from the dataset).

## 9. D-FS7: API + admission follow the storage slice pattern

`/v1/file-shares` CRUD routes follow the storage per-kind handler pattern;
reads gate on `CapSharesRead`, writes on `CapSharesWrite` (the existing
`RoleShareAdmin`, ADR-0018), enforced at admission. Spec validation at the
boundary (the only RBAC/enforcement point):

- `dataset` must reference an existing `Dataset` resource;
- `protocols` must be a non-empty subset of `[smb, nfs]`;
- `access` entries must reference existing NAS users — and be **non-empty when
  SMB is enabled**: a section with no `valid users` would be an open export,
  against ADR-0023's per-user access control (the controller enforces the same
  rule on the *effective* grants, so all-dangling lifecycle drift is a hard
  Error too);
- NFS hosts, when present, must be well-typed strings that are valid
  IP/network/host identifiers (a non-string entry is rejected, never silently
  dropped into a reduced allowlist);
- a dataset may be shared by exactly one `FileShare`, and the dataset is
  **immutable** on an existing resource — a PUT/PATCH that changes it is
  rejected (422) so the stored name/spec stay consistent and the deletion
  finalizer always clears the actually-exported dataset.

Invalid grants are rejected (`422` at the API; the controller independently
guards `grant_invalid` for host lists that somehow slip through) and never
produce a partial export.

**Lifecycle drift after validation:** a user deleted *after* the spec write
leaves a dangling `access` reference. The controller drops the dangling grant
(recorded in `status.actual.droppedUsers`), removes its passdb entry, and keeps
exporting for the remaining grants — the resync converges within a pass. If
**every** grant is dangling, the share reports `Error`/`grant_invalid` (a
section with no `valid users` would be an open export) — "keeps exporting for
the remaining grants" assumes remaining grants exist. A user that exists but
has no allocated UID yet (the `User` controller allocates on its first pass) is
surfaced as `pendingUsers` and the share retries on a timed requeue — the
requeue is preserved on the no-op pass, so the retry cadence survives a status
write. Username renames are rejected at the API boundary (422) and persistently
by the `User` controller (`Error`/`username_immutable`, mirroring the storage
`name_immutable` rule): the identity service keys UID allocations by username,
so a rename would leak the old allocation and break the deletion finalizer's
release path. The immutability guards pin `status.Wanted` to the previous spec
on the error pass, so they fire on every reconcile — a one-pass tripwire would
let the next resync converge the change while the old host state leaked
(dataset re-export under a new name with the old `sharenfs` export orphaned).

## 10. D-FS8: Metrics follow the catalog

The backends/controller emit the already-declared
`amberhold.shares.file.exported` (gauge, labels `share` + `protocol`) and
`amberhold.shares.file.connections` (gauge; v1 does not track connections, the
series is declared at zero) on **every** reconcile pass — a
`Pending`/`Degraded`/`Error` share declares its not-exported 0 series rather
than an absent signal, and the deletion finalizer declares the 0 series for a
removed share so `/metrics` never keeps a stale 1 — plus the catalog-added
`amberhold.identity.uid.{allocated,released}` counters from the allocator. No
metric names are emitted outside the declared catalog (D10, ADR-0008).

| Metric | Kind | When |
|--------|------|------|
| `amberhold.shares.file.exported{share,protocol}` | gauge | every FileShare pass (1 exported / 0 not) |
| `amberhold.shares.file.connections{share,protocol}` | gauge | every FileShare pass (0 in v1) |
| `amberhold.identity.uid.allocated{username}` | counter | every UID allocation |
| `amberhold.identity.uid.released{username}` | counter | every UID reclamation |

## 11. Status shapes (contract envelope)

The controller writes the generic contract envelope (D4) with the
contract-specific fields under `status.actual`:

- `dataset` (path), `protocols` (desired subset), `state` (`exported`),
  `endpoints` (per-protocol identifiers, e.g. `smb:tank-photos`, `nfs:tank/photos`);
- `smb`: `{enabled, users}` — the effective SMB grants;
- `nfs`: `{enabled, hosts}` — the effective NFS allowlist;
- `ownerUid` when a grant materialized ownership;
- `droppedUsers` / `pendingUsers` when lifecycle drift was absorbed.

Phases: `Ready`/`share_exported` when converged, `Pending`/`dataset_missing`
for the missing-dataset prerequisite (timed retry, never a hard failure),
`Degraded` for host probes and chown failures, `Error` for invalid specs,
`grant_invalid` host lists, and `name_immutable` dataset changes on an existing
share (mirrors the Dataset controller's immutability rule). An unchanged share
emits metrics but skips the status write (idempotent no-op, D9).

## 12. Wiring into the daemon (D6)

In `app.New` (step 3, controller registration):

1. build the production host (default `ZFSHost`); the same facade implements
   the extended `SharesHost` surface (`NewZFSHost` returns the storage host,
   the concrete implementation carries the chown/pdbedit/smbcontrol binaries
   and the samba config path);
2. construct the identity `Allocator` over the spec store with the registry as
   its metric sink, and pass it to `auth.NewUserController` (status.uid);
3. construct the SMB and NFS backends and the `FileShareController(host,
   allocator, smb, nfs, registry)` and register it with the runtime — the
   store's events + resync drive its loop (D2/D3) with the shared 30 s
   reconcile timeout;
4. declare the shares + identity metric names in `declareCatalog`;
5. register the `/v1/file-shares` admission routes (reads `shares:read`,
   writes `shares:write`) and the API handlers — API last (D6 step 6).

`Config.StorageHost` remains the test/install seam; tests inject the fake
(which implements `SharesHost`) and the controller reconciles a seeded
`FileShare` end-to-end.

## 13. Samba state/lock directory layout on `config/var`

Samba needs writable state beyond the config (lock directory, tdbsam passdb,
cache). Per ADR-0011/0013 these live on the OS-disk `config/var` partition;
the image ships the samba `state directory`, `lock directory`, `cache
directory`, and `private dir` pointed at `config/var/samba/...`, with the
tdbsam passdb (`passdb.tdb`) and the generated `smb.conf` (reached via the
`/etc/smb.conf` symlink) under it. This change bakes the requirement into the
shares design so the `infra` image work (a later change) ships the layout; the
daemon never creates host files outside `config/var`.

## 14. `idmapd` domain mapping (NFS UID alignment)

NFSv4 ownership is coherent with SMB because both resolve to the NAS user's
allocated UID (ADR-0005). `idmapd` performs domain mapping so mismatched
client-side UIDs still resolve to the NAS UID; the appliance image configures
`idmapd` with the NAS domain and the domain-mapping policy, and the UIDs it
maps are the identity service's allocations. The concrete `idmapd.conf` and
reload wiring are settled on the Linux integration surface (the design open
question below); this change's contribution is the UID alignment itself — the
allocator guarantees SMB (tdbsam) and NFS (dataset owner) agree on one UID per
NAS user.

## 15. Risks / Trade-offs

- **RO-root samba state**: samba needs writable state beyond config → the
  state-dir layout on `config/var` is fixed here (§13); the image bakes it in.
- **`sharenfs` variance across ZFS versions** → the image pins the ZFS userland
  (ADR-0001); verified on the Linux integration surface.
- **tdbsam without system accounts**: some tools call `getpwnam()` for NAS
  users and fail → documented v1 constraint; the NSS module is the recorded
  future path (D-FS3).
- **Reload semantics**: `smbcontrol reload-config` may not pick up every
  section change → a reload failure is a retriable reconcile error (D9), never
  a silent partial export; samba config is regenerable on the next resync.
- **Concurrent multi-protocol writes**: SMB + NFS on one dataset is a coherency
  hazard by nature → ADR-0010 explicitly supports the combo; UID alignment
  keeps ownership coherent; no mitigation beyond the ADR's supported-matrix
  stance.
- **One share per dataset** (v1 simplification): a second `FileShare` over the
  same dataset is rejected at the API; the resource-name convention would
  otherwise alias the deletion finalizer and samba section.
- **Status write amplification** → reconciled status writes follow the
  runtime's coalescing (D5); unchanged shares emit metrics but skip the status
  write.

## 16. Implementation notes (settled open questions)

- **Samba state/lock directory layout on `config/var`**: `config/var/samba`
  with `smb.conf`, `passdb.tdb`, lock/cache/state dirs (§13); baked into the
  `infra` image work, tunable without spec impact.
- **`idmapd` domain configuration**: the appliance image fixes the domain and
  reload wiring (§14); verified on the Linux integration surface.
- **`chown` recursion scope**: dataset root only in v1 (`-R` deferred); no spec
  impact.
- **`amberhold.identity.uid.*` metric names**: catalog naming settled in §10
  and `contracts/metrics/catalog.yaml`; tunable without spec impact.
- **Samba section-name collisions**: the default section name (dataset path
  with separators dashed) is not injective — a collision across the desired
  set appends a short deterministic hash suffix so two distinct datasets never
  render a duplicate section.
- **pdbedit password prompt / system-account resolution**: the D9 runner has no
  stdin writer yet and pdbedit resolves Unix accounts via `getpwnam` — recorded
  as the integration-surface verification (D-FS3/§5) with the future runner
  stdin extension and NSS module as the follow-ups.