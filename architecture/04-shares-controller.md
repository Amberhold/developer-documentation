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

No system account is created on the host and `/etc/passwd` is never written
(ADR-0001). The UID lives in the spec store (via the identity service) and in
samba's tdbsam passdb on `config/var`. To make `getpwnam()` resolve NAS users —
required by `pdbedit -a -t -u <user>`, which resolves the account before writing a
passdb entry — the identity service materializes the allocated username↔UID map
into the image's `libnss-extrausers` files on the writable `/var` overlay
(`/var/lib/extrausers/{passwd,group}`, config/var-backed). The image ships
`libnss-extrausers` and adds the `extrausers` source to the passwd/group
nsswitch chain, so a lookup resolves without any write under the read-only root.
The materialized map is derived state: it is rebuilt idempotently from the
ledger on every pass (atomic temp + rename), so a missed reclaim is healed on
the next reconcile. Entries use a home of `/` and `/usr/sbin/nologin` as the
shell; the image lists that shell in `/etc/shells` so samba's account check
accepts it.

Dataset root ownership is set by the `FileShare` controller to the linked user's
UID (`chown`) so both NFS clients (via `idmapd` domain mapping, ADR-0005) and SMB
(via tdbsam) see consistent ownership.

**Rationale:** ADR-0001 forbids writing under `/`; `/etc/passwd` is RO. The
allocated map is materialized on the writable overlay and exposed through a
stock NSS module rather than by mutating the system passwd. tdbsam holds
username↔UID independently of the system passwd, and samba resolves the mapping
itself. Alternatives rejected: bind-mounting a generated `/etc/passwd`
(violates the RO-root convention); creating real system accounts with `useradd`
(`/etc/passwd` is RO); a custom `libnss_amberhold` module (real systems
engineering and couples NSS to the store format — deferred, remains the future
alternative to the stock module).

**Integration note (pdbedit):** `pdbedit -a` normally prompts for a password and
resolves the Unix user via `getpwnam`. The host primitive (`PdbeditUpsert`)
ensures the entry exists — running `pdbedit -a -t -u <username>` through the D9
`Runner`'s stdin path when the observed passdb has no entry (`-t`,
`--password-from-stdin`, makes the prompt non-interactive) — and then installs a
usable credential with `pdbedit --set-nt-hash=<hex> -u <username>`. The hex is
the **NT hash of the principal's NAS password** (`MD4(UTF-16LE(password))`),
stored as internal credential state on the `User` resource (never returned by
the API) and derived on password set/change or folded from the installer seed
for the bootstrap admin. The account resolves through the image-provided
`libnss-extrausers` link (the materialized map above), so materialization
succeeds for a NAS user with no system account. A grant whose principal has no
local password (an OIDC/federated principal) has no derivable NT hash: no entry
is created for it and the share reports `Degraded`/`smb_credential_unavailable`.
A disabled account is gated out the same way the sign-in path gates it — no
credential is installed and a stale entry is removed — but it is intentional, so
the share stays `Ready` and the grant stays in `valid users` (§9). A stored value
that is not a well-formed NT hash is normalized to "no credential", so a
corrupted entry is never passed to `pdbedit` and never leaves the share
reporting converged over an entry that was never written.
The unit surface (argument sequence, typed `ExitError`) is covered over the fake
runner, and the shipped invocation is verified on the dev-harness guest.

## 6. D-FS4: NFS grants live in the existing `options` bag — no contract delta

The NFS client allowlist is `spec.options.nfs.hosts` (e.g.
`["192.168.1.0/24", "host.example.com"]`), consumed only when `protocols`
includes `nfs`. The `FileShare` schema's `options` object is already
`additionalProperties: true`, so **no schema change is required** (verified in
`contracts/openapi/v1.yaml` by the contracts task). The NFS backend builds an
explicit `sharenfs` value from the list that preserves the `sharenfs=on`
defaults and always adds `insecure` (ADR-0009 amendment):

- no hosts: `rw=*,crossmnt,no_subtree_check,insecure` (exports to all clients),
- hosts: `rw=<host1>:<host2>,crossmnt,no_subtree_check,insecure`.

The explicit list keeps the generated value deterministic and testable (versus
the opaque `on`) and makes the export mountable by a client whose NFS requests
originate from an unprivileged source port — notably the `qemu`/slirp dev
harness, whose `hostfwd` cannot preserve a privileged port (`MNT3ERR_ACCES`
otherwise). Host/IP grants still gate which clients may mount. It is applied via
the host facade:

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
Each credentialed grant carries the NT hash of the user's NAS password, which
the backend installs via `pdbedit --set-nt-hash` and converges when the observed
hash differs (so a password change re-provisions the entry on the next pass). A
grant whose principal has no local password is not materialized and is reported
as `Degraded`/`smb_credential_unavailable`; a grant to a disabled account is
likewise never materialized (any stale entry is removed) but is neither reported
nor made `Degraded` (§9). Reload is `smbcontrol all
reload-config` (no service restart; samba runs from the image), issued **once
per pass only when something changed** — a converged regenerate is a no-op (D9),
verified by config diff against the observed file and the observed
`pdbedit -L -w` (username + NT hash) user list.

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

**Password-less SMB grants:** a principal with no local password (an
OIDC/federated principal, or a local user whose password has not been set since
the credential-link change) has no derivable SMB credential. Such a grant is
recorded in `status.actual.smbCredentialUnavailable`, no credential-less passdb
entry is created for it, and the share reports
`Degraded`/`smb_credential_unavailable` while NFS and the credentialed SMB grants
continue to export. This mirrors `chown_failed`: the export is not blocked, the
unsupported grant is surfaced rather than silently dropped or silently broken.
A stored credential the backend cannot normalize to a 32-hex NT hash is
classified the same way, so the backend never refuses to write it while the
share reports converged.

**Disabled accounts:** a disabled `User` is gated out of SMB by the same
account-state rule the sign-in path applies. Its credential is never installed
and any stale passdb entry is removed on the next pass, but — because a disabled
account is a deliberate administrative state, not an unsupported grant — it is
not recorded in `smbCredentialUnavailable` and does not flip the share
`Degraded`. The grant stays in the section's `valid users` (removing it would
change the section's access list and could render a single-grant section open),
so a disabled principal can see the share and simply fails to authenticate,
exactly as at sign-in.

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
- `droppedUsers` / `pendingUsers` when lifecycle drift was absorbed;
- `smbCredentialUnavailable` when an SMB grant has no derivable credential.

Phases: `Ready`/`share_exported` when converged, `Pending`/`dataset_missing`
for the missing-dataset prerequisite (timed retry, never a hard failure),
`Degraded` for host probes, chown failures, and password-less SMB grants
(`smb_credential_unavailable`), `Error` for invalid specs, `grant_invalid` host
lists, and `name_immutable` dataset changes on an existing share (mirrors the
Dataset controller's immutability rule). An unchanged share emits metrics but
skips the status write (idempotent no-op, D9).

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
cache). Per ADR-0011/0013 these live on the OS-disk `config/var` partition. The
installed image ships the layout: the generated `smb.conf` is reached through
the `/etc/smb.conf` symlink (with the daemon's default `/etc/samba/smb.conf`
chaining to it, so smbd, `testparm`, and `pdbedit` all read the regenerated
file), core renders `state directory`, `lock directory`, `cache directory`, and
`private dir` under `config/var/samba/{state,lock,cache,private}` into the
generated `[global]` stanza (design D-FS5), and the image creates those
directories at boot through `tmpfiles.d`, ordering `smbd`/`nmbd` after
`config-var.mount`. The tdbsam passdb (`passdb.tdb`) lands in the private dir.
The daemon never creates host files outside `config/var`.

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

## 15. NFS server runtime (fixed ports, `zfs-share`, single-LAN binding)

The NFS backend only sets or clears the dataset's `sharenfs` property (§6,
ADR-0009); the actual export is performed by ZFS's share path, which on Linux
shells out to the kernel NFS server userspace (`exportfs`/`rpc.nfsd`/
`rpc.mountd`). The installed image therefore ships and enables that userspace
(`add-nfs-server-and-share-ports`; package list and bake in
`docs/architecture/13-os-image-installer.md` D31): `nfs-kernel-server`
(`rpc.nfsd`, `rpc.mountd`, `exportfs`), `nfs-common` (`rpc.statd` and the client
tools), and `rpcbind` (the portmapper). Without it a dataset whose `sharenfs`
property is set is never actually exported and an NFS `FileShare` cannot serve
traffic.

- **Fixed auxiliary ports.** The image pins `nfsd` 2049, `mountd` 20048, `statd`
  32765, and `rpcbind` 111 (the protocol fixes rpcbind) in `/etc/nfs.conf`.
  Slirp `hostfwd` can only forward known ports, so a dynamic `mountd`/`statd`
  would make NFSv3 unreachable through the dev harness; fixed ports are also
  firewall-friendly.
- **`zfs-share` at boot.** `zfs-share.service` runs `zfs share -a`, exporting
  every dataset whose `sharenfs` property is set — including across a reboot, so
  a converged `FileShare` is served again after restart without waiting for a
  core reconcile pass.
- **Single-LAN binding.** When the `network` resource declares no data plane (the
  v1 default), a share binds on the single LAN; the export's client allowlist is
  the `sharenfs` grant built from `spec.options.nfs.hosts` (§6). Under the dev
  harness a host-originated share connection appears to the guest as the slirp
  gateway `10.0.2.2`, so a grant must allow it (or use the `sharenfs=on`
  all-clients default).

The NFS backend itself is unchanged: no contract delta and no `core` change.

## 16. Risks / Trade-offs

- **RO-root samba state**: samba needs writable state beyond config → the
  state-dir layout on `config/var` is fixed here (§13); the image bakes it in.
- **`sharenfs` variance across ZFS versions** → the image pins the ZFS userland
  (ADR-0001); verified on the Linux integration surface.
- **tdbsam without system accounts**: tools that call `getpwnam()` for NAS
  users resolve through the image-provided `libnss-extrausers` link (D-FS3),
  backed by the identity-materialized map; a custom `libnss_amberhold` module
  remains the recorded future alternative.
- **Reload semantics**: `smbcontrol reload-config` may not pick up every
  section change → a reload failure is a retriable reconcile error (D9), never
  a silent partial export; samba config is regenerable on the next resync.
- **Concurrent multi-protocol writes**: SMB + NFS on one dataset is a coherency
  hazard by nature → ADR-0010 explicitly supports the combo; UID alignment
  keeps ownership coherent; no mitigation beyond the ADR's supported-matrix
  stance.
- **Always-`insecure` NFS export**: the privileged-source-port requirement is
  waived (ADR-0009 amendment) so unprivileged-source-port clients (the slirp
  dev harness) can mount → host/IP grants remain the access gate; there is no
  `secure`/`insecure` toggle in v1.
- **NT hash stored at rest**: the SMB credential (NT hash) is recorded on the
  `User` resource in the spec store, which lives on the encrypted OS-disk LUKS
  container (ADR-0011) → it is never exposed by the API and never echoed in
  status; `--set-nt-hash` installs it without plaintext ever reaching the
  passdb.
- **NT hash visible in the `pdbedit` argv**: `--set-nt-hash=<hex>` is observable
  in `/proc/<pid>/cmdline` for the duration of the call → inherent to the
  `pdbedit` interface (accepted limitation); the value is a validated 32-hex
  string passed as a single argv element (no shell string, no injection
  surface), and the daemon runs as root on the appliance.
- **`--set-nt-hash` create-then-set is two shell-outs and can partially fail** →
  the primitive reports a typed error and the reconcile retries the whole pass;
  the share reports `Error`/`smb_reconcile_failed` until it converges, never a
  silent partial export.
- **One share per dataset** (v1 simplification): a second `FileShare` over the
  same dataset is rejected at the API; the resource-name convention would
  otherwise alias the deletion finalizer and samba section.
- **Status write amplification** → reconciled status writes follow the
  runtime's coalescing (D5); unchanged shares emit metrics but skip the status
  write.

## 17. Implementation notes (settled open questions)

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
- **pdbedit credential provisioning**: the host primitive creates the entry
  when absent (`pdbedit -a -t -u <user>`, non-interactive via the D9 stdin path;
  user resolution via `getpwnam` is satisfied by the image-provided
  `libnss-extrausers` link, D-FS3) and installs the credential with `pdbedit
  --set-nt-hash=<hex> -u <user>`, where the hex is the NT hash of the user's NAS
  password. A converged pass with a matching hash is a read-only no-op; a
  credential-less principal (OIDC/federated) is reported as
  `Degraded`/`smb_credential_unavailable` and never gets an entry.