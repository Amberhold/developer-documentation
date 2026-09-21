# ADR-0034: Writable dataset mount root on the read-only OS root

- Status: accepted
- Date: 2026-09-18
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/03-storage-controller.md` (D-S3, D-S14),
  `docs/architecture/13-os-image-installer.md` (D5, D30); ADR-0001, ADR-0007,
  ADR-0011, ADR-0013, ADR-0024

## Context

The installed root filesystem is a **read-only squashfs A/B slot**; the only
writable, persistent filesystem roots are the OS-disk partitions mounted at
`/config/var` (ext4) and the `/var` overlay built on top of it (ADR-0001,
ADR-0011, ADR-0013). No design document fixed where ZFS pool/dataset
mountpoints live.

ZFS defaults a pool's root dataset mountpoint to `/<pool>` and each child
dataset to its ZFS path (`/<pool>/<dataset>`). On the installed system ZFS
therefore tries to create a mountpoint directory directly under `/`, which is
read-only:

```
cannot mount '/testy': failed to create mountpoint: Read-only file system
```

`zpool create` reports this as a **failure of the whole command** even though
the pool itself was created, so `core` surfaces
`Error`/`pool_create_failed` and the pool is left half-provisioned. This was
observed on the macOS qemu dev harness when creating a pool from the discovered
unseeded disks.

A second, independent defect compounds it: `zpool status -j` reports a leaf
vdev's device as the **partition** it created (`/dev/disk/by-id/virtio-X-part1`),
while the desired member identity is the **whole-disk** by-id path
(`/dev/disk/by-id/virtio-X`). `core`'s exact-string topology comparison never
matches, so every pool on whole-disk members flips to `Error`/`topology_immutable`
on the next resync even when it is healthy.

## Decision

**The image provides one writable dataset mount root at `/mnt`, and all ZFS
mountpoints live beneath it.**

- The OS image bind-mounts the OS-disk `config/var` partition's `mnt/`
  directory over `/mnt` at boot, before ZFS pools are imported or mounted
  (`docs/architecture/13-os-image-installer.md` D30). `/mnt` exists in the
  read-only root as an empty directory, so it is a valid bind target and becomes
  writable without touching the immutable slot.
- Pools SHALL be created with their root dataset mountpoint set to
  `<mount-root>/<pool>` (default `/mnt/<pool>`). Datasets inherit the ZFS
  default (`/mnt/<pool>/<dataset>`); the mountpoint is controller/installer-owned
  and is **not** part of the `Pool`/`Dataset` API contract (the declared Dataset
  property set does not include `mountpoint`, so a client cannot move a dataset
  off the root).
- The installer creates each pool with `zpool create -m /mnt/<pool>` and then
  exports it: `-m` records the correct mountpoint in the pool so the installed
  system mounts it on the non-forced import. The live installer root is
  writable, so the create-time mount succeeds and the immediate export releases
  it. (`-N` is a `zpool import` option and is rejected by `zpool create`, so it
  cannot be used to suppress the create-time mount.)
- `core` creates pools with the same `-m /mnt/<pool>` convention and, after
  import, mounts the pool's datasets through ZFS's normal inheritance.

**`core` normalizes observed ZFS member identity to the whole-disk by-id path
before comparing topology.**

- The host facade resolves each leaf device reported by `zpool status -j`
  (partition or whole disk) to its parent whole-disk `/dev/disk/by-id` alias —
  the same partition→parent mapping the mdadm facade already performs for OS
  mirror members. Only then is the layout compared, so the immutable-topology
  check stays a **strict exact match on stable disk identity** and still catches
  a genuinely swapped disk; it simply stops comparing a partition alias against
  a whole-disk identity.
- A device that cannot be resolved to a by-id alias keeps its raw path
  (identity loss degrades, never hides).

## Alternatives considered

- **ZFS `altroot`** (`zpool create -m none` + `-R`/`altroot=/var/amberhold/...`)
  — prefixes mounts at import time, but `altroot` is an import-time property that
  is not persisted in the pool, must be re-applied on every import, and makes
  `zfs get mountpoint` disagree with the real path shares/apps must use.
  Rejected: hidden indirection for no benefit over a fixed mount root.
- **Mount under the writable `/var` overlay** (`/var/mnt/<pool>`) — needs no
  new bind, but places user data under system-state paths, leaks OS-disk layout
  into share/app paths, and still needs ordering against ZFS import. Rejected.
- **Pre-create `/mnt/<pool>` directories in the image** — pool names are dynamic
  at install/API time, so the image cannot know them. Rejected.
- **`zpool create -m none` and per-dataset explicit mountpoints** — every role
  dataset would have to carry its own mountpoint, and shares/apps would depend on
  each Dataset spec, spreading a filesystem convention across the content layer.
  Rejected: the pool-root convention is simpler and keeps inheritance.
- **Normalize only inside `sameLayout` (basename/strip `-partN`)** — would mask a
  swapped disk and weakens the immutable-topology check; the archived
  `pool-adoption-after-install` change explicitly rejected it. Resolving
  observed members to the stable whole-disk identity keeps the comparison strict.
- **Forced import (`zpool import -f`) or forced mount** — addresses nothing and
  can seize a pool in use elsewhere (ADR-0011 / `pool-adoption-after-install`
  D1). Rejected.

## Consequences

- Pools created or imported after this decision mount under `/mnt/<pool>` on the
  read-only-root system; `zpool create` no longer fails on the mountpoint step.
- Shares and apps reference conventional absolute paths under `/mnt` for
  dataset mountpoints, and the paths are stable across reboot.
- The pool topology comparison remains a strict identity match; ZFS's internal
  partition aliasing no longer reads as drift. The previously documented
  "deferred normalization" fallback in `docs/architecture/03-storage-controller.md`
  §18 `pool-adoption-after-install` D2 is superseded for the whole-disk case.
- Pools created **before** this change carry a root-level mountpoint (e.g.
  `/tank`). They are not migrated automatically; the mountpoint must be corrected
  (`zfs set mountpoint=/mnt/<pool> <pool>`) or the pool re-created. This affects
  only dev/harness artifacts, since no release shipped with the old convention.
- The `config/var` mountpoint directories for pools are tiny and live on the OS
  disk; losing the OS disk already loses the pool encryption keys (ADR-0011), so
  this introduces no new data-loss coupling.
