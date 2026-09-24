# OS Image, Installer, and Update Delivery

> Discovery-phase design. Authored from the `os-image-installer` openspec change.
> The decisions D1–D31 here fix how the Amberhold OS is **built** (mkosi image
> build for arm64 and amd64), **installed** (live-ISO console TUI), and
> **updated** (rauc A/B bundles through a per-arch repo channel), and how the
> install-time decisions reach the spec store (seed-manifest handoff), plus the
> macOS qemu/HVF dev harness that exercises the boot chain locally. ADR-0001
> fixes the squashfs A/B root, ADR-0011
> the OS-disk layout + encryption + unlock factors, ADR-0006 updates as a product
> feature, and ADR-0013 the writable state; this document fixes the *internal
> mechanics* on top, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (ADR-0031). The feature map (features 0,
> 1, 12) is in `docs/architecture/01-os-feature-map.md`. ADR-0032 records the
> A/B tooling choice that resolves the deferred ADR-0001/0011 decision.

## 1. Purpose

The control plane is fully designed and its controllers implemented, but nothing
produces a bootable system: `infra` is empty, there is no OS image or installer,
and the Update controller (feature 1) has a contract and CLI verbs but no core
implementation. This document closes that gap end-to-end:

- **OS image build** (`infra`): mkosi-built Debian image — read-only squashfs
  A/B slots, per-slot kernel+initramfs on a plain ESP, baked `core` daemon,
  web-ui static, and pinned daemons per ADR-0001.
- **Installer** (`infra`): bootable live ISO with a console TUI (Go + host
  facade) performing the ADR-0011 install steps.
- **Provisioning handoff**: the installer writes a versioned seed manifest;
  `core` folds it into the spec store once at first boot via its validated write
  path (D8).
- **Update delivery**: repo-channel + signed rauc bundle build reusing the same
  install path as fresh install.
- **Update controller** (`core`): reconciles the existing `updates` singleton
  against a rauc host facade.

It follows the singleton-controller and thin-facade conventions of the
certificates (ADR-0028), networking (ADR-0014), and storage (ADR-0024/0021)
slices.

## 2. Goals / Non-Goals

**Goals:**

- A bootable OS image and live-ISO installer that realize the ADR install-time
  decisions (ADR-0001/0006/0007/0011/0013/0014) end-to-end (D1–D12).
- rauc as the single A/B mechanism shared by fresh install and updates (D1).
- Preserve "core is the only spec-store writer" (ADR-0002) through the
  seed-manifest handoff (D8).
- The Update controller implemented against the existing `updates` contract,
  with trigger/rollback, `repoUrl`+channel reconcile, `autoApply`, store
  snapshot coordination, automatic boot-fail rollback, and observable status +
  metrics (D14–D16, D22).
- Unlock policy exposed as a spec-store resource that drives the update flow
  (D13).
- The resolved threads D13–D22 recorded: UnlockPolicy, repoUrl,
  deferReboot/autoApply, store snapshot, seed ordering, Argon2id params,
  hostname/DNS/NTP, Pool+Dataset seed, ISO trust anchor, automatic rollback.
- arm64 image support (D24), the per-arch repo channel layout (D25), the macOS
  qemu/HVF dev harness (D26), and the installer-only live ISO carrying the trust
  anchor + initial bundle (D27) recorded as the additive dev-harness capability.

**Non-Goals:**

- PXE / network-boot installer (console live-ISO only, per ADR-0011
  physical-factor enrollment).
- Alternative A/B tooling beyond rauc; no migration path from rauc.
- Offline / air-gapped update delivery (repo-channel only, ADR-0006).
- Encrypted-pool recovery on OS-disk loss (reinstall + `zpool import` covers
  unencrypted data only, ADR-0011).
- Web-based installer; no direct host access via installer (feature 8 posture).
- The Web-UI and the image-baked web-ui static serve path — the web-ui repo
  remains a stub; this change bakes whatever static exists.
- Removing the Linux CI boot matrix: amd64 remains the canonical product gate;
  the local arm64 harness is additive, not a replacement.
- Supporting any architecture beyond arm64 and amd64.
- Running the macOS harness under x86_64 emulation (TCG) — the value is the
  native arm64 loop; x86_64 macOS is out of scope.

## 3. Decisions

### D1: rauc is the A/B tooling
Resolves the deferred ADR-0001/0011 choice. rauc's explicit slot model matches
the two-squashfs-slots layout, its signed `.raucb` bundles satisfy ADR-0006's
trust anchor, and its EFI/systemd-boot backend does slot selection at the ESP
level (A/B-at-ESP, no unlock needed to select). Alternatives: ABRoot
(general-distro dual-root but single-ESP-native) and ostree
(one-root-many-deployments paradigm fight) were rejected. Decision-of-record:
ADR-0032.

Three custom pieces are owned by this design:

1. **Dual-ESP sync** — a post-install hook copies the ESP content to the mirror
   member's ESP before activation (ADR-0011). The installed OS does not mount
   its ESP, so an OS-initiated update has the hook mount the primary ESP from
   the `amberhold.esp=` kernel-cmdline reference (and, on a mirror, derive the
   other member's ESP from the `md0` array) before copying, and reconcile the
   bootspec entry's initrd list with what it actually staged (the
   kernel-modules initrd is not present in every build).
2. **Boot-fail reporting** — selection is pre-unlock but rootfs mount success is
   post-unlock; an initramfs hook reports rauc boot-status good/bad.
3. **rauc-writes-to-LUKS** — on a booted host the inactive slot is
   `/dev/mapper/...`; rauc `system.conf` targets mapper devices, not raw
   partitions.

### D2: systemd-boot bootloader + bootenv
`loader.conf` default entry per slot; per-slot kernel+initramfs on the plain
ESP. Matches rauc's EFI backend. GRUB rejected as heavier and unnecessary.

### D3: initramfs-tools with a custom hook
Debian-native; the hook assembles `md0` (degraded when a member is missing),
unlocks the container via the enrolled factor, reports boot status, mounts the
active slot. dracut rejected as non-Debian-default.

### D4: mkosi image build
Declarative Debian image builder; injects the `core` binary and web-ui static at
build time; produces the squashfs slot images, per-slot kernel+initramfs, and
ESP content. Pinned package versions per ADR-0001.

The staged build sources are consumed at a **fixed mount path**,
`/work/src/build`. The documented invocation and CI pass an explicit
build-sources target, `--build-sources <staged-dir>:/build`, so the mount target
is part of the invocation rather than an implicit mkosi default (a bare
`--build-sources <dir>` lands the tree at `/work/src`, not `/work/src/build`).
`90-bake-amberhold.sh.chroot` and the live-ISO bake both read `$BUILD` at that
path, and the OS bake preflight fails with a diagnostic naming the expected path
when `$BUILD/core` is absent — a mount-target mismatch fails immediately instead
of at an opaque post-install step.

### D5: ext4 for the container's writable partitions
Spec store + `config/var` on ext4 inside the container; slots remain squashfs.
Matches the current bbolt host; xfs/f2fs deferred as unneeded.

The installed root is a read-only squashfs slot, so `/var` is an **overlay
mount** at boot: lower = the baked (read-only) `/var` in the slot, upper + work
on the writable `config/var` partition (mounted at `/config/var` by the fstab
the bake writes). The overlay is brought up by a small
`amberhold-var-overlay.service` unit ordered after `config-var.mount` and before
`basic.target` (and before `systemd-networkd-persistent-storage.service` and
`systemd-timesyncd.service`, which touch `/var` before `basic.target`), so
daemons that keep state under `/var` (containerd, systemd-logind, samba,
systemd-timesyncd) start over a writable `/var` while the A/B slots stay
read-only squashfs (ADR-0001/0011, ADR-0013 writable state on the OS-disk
partitions). Without this, the installed system boots to multi-user but those
daemons crash-loop on the read-only `/var`.

### D6: Update payload is a rauc bundle
Deferred to D1 by definition: `.raucb` bundle, CMS-signed, trust anchor baked
into the image. No separate payload design.

### D7: Live ISO + console TUI installer
Factor enrollment needs physical presence (YubiKey insertion, recovery
passphrase) — a console TUI on a bootable ISO is the honest fit. Web-based
rejected (conflicts with the no-direct-host-access posture and complicates
enrollment); PXE deferred.

The installer **flushes and releases the ESP before reporting success**. It
mounts the OS disk's ESP(s) for `rauc install` (the primary at `/boot/efi`, and
a mirror's second member at `/boot/efi-b`) so the `rauc-post-install` hook can
copy the per-slot kernel+initramfs into them. After `rauc install` returns,
`install.WriteOSImage` calls `sync` and then unmounts every ESP it mounted,
failing the install with a diagnostic naming the ESP if the flush or unmount
fails. This makes the per-slot boot files durable on disk the moment the
installer reports success, independent of the caller's teardown timing — an
abrupt termination of the installing system (for example, a test harness killing
the guest) can no longer leave 0-byte `vmlinuz`/`initrd` files on the ESP. The
hook also ends with a `sync` as cheap defense-in-depth, which matters on an
OS-initiated update where the hook owns the ESP mounts and there is no later
teardown.

### D8: Seed manifest handoff (option C)
The installer writes a versioned, validated seed manifest to the spec-store
partition inside the container; `core` folds it into the spec store once via its
normal validated write path, marks it consumed, then serves the API. This keeps
ADR-0002's single-writer invariant and keeps the installer store-format-agnostic
(it writes JSON, not bbolt). Alternatives rejected: installer writes the bbolt
store directly (duplicates format + validation, breaks the invariant);
first-boot wizard over the API (two-stage dance plus a CA bootstrap
chicken-egg). The seed carries desired state only (admin user + Argon2id hash,
network planes + per-interface IP, storage layout as Disk + Pool + Dataset
declarations including the OS-disk device, unlock-policy mirror); physical
provisioning stays on the disk directly. A signature is unnecessary — the
installer is the root of trust at install time. The seed's field set is expanded
in D19 (hostname/DNS/NTP) and D20 (full Disk + Pool + Dataset declarations), and
the unlock-policy mirror is realized as the `UnlockPolicy` singleton (D13).

### D9: Fresh and recover modes ship together
Recover mode detects existing pools, imports them, and reassigns data/app roles
(ADR-0013 recovery path) and clearly states encrypted data is unrecoverable on
OS-disk loss. Scoping it later would leave the documented recovery path
unrealized. The recover-mode seed is generated from the **observed** pool
topology (introspected via `zpool status`/`zpool get`), because the pool
controller errors on layout drift — the seed's Pool, Disk, and Dataset
declarations must match what the installer actually imported.

### D10: Go TUI installer + host facade
A Go binary with a target facade seam shelling out to cryptsetup, mdadm, parted,
zpool/zfs, and systemd-cryptenroll — mirroring core's facade pattern and keeping
install logic unit-testable behind fakes. Shell+dialog rejected as untestable.

### D11: qemu + nspawn test strategy
nspawn for fast image-level checks; qemu for the boot-chain matrix
(single/mirror x fresh/recover) covering bootloader, LUKS unlock, and
degraded-boot paths that nspawn cannot exercise.

### D12: One change, staged task ordering
One openspec change across `docs`, `infra`, `contracts`, and `core`. Deliberately
large; tasks.md sequences it as docs → os-image → installer → contracts → core →
harnesses so each phase has a verifiable result. The core Update controller is
included per scope decision.

### D13: UnlockPolicy singleton resource
ADR-0011's host-level `UnlockPolicy` resource is realized as a new singleton
kind: the spec holds the **desired** per-mode policy (seeded by the installer,
operator-writable), and a small core controller shells out to
`cryptsetup luksDump` / `systemd-cryptenroll` to mirror enrolled factors into
**status** after boot; drift between the desired policy and observed factors
surfaces in status rather than being silently overwritten. Observation targets
the **backing** LUKS device (resolved from the configured container mapper via
`cryptsetup status`), because the opened mapper's leading sectors are the
decrypted inner GPT, not the LUKS header. The initial all-zero luksFormat key is
probed for (`open --test-passphrase` with the zero key) to distinguish a
format-only container from one whose initial key was removed after a real factor
was enrolled — `luksDump` alone cannot distinguish them. The seed's
unlock-policy mirror initializes the spec at install; the Update controller
reads the resource for the touch-mode pre-reboot warning. The singleton is
seeded with an empty spec at startup alongside the other singletons (so it never
404s), and exposed as GET-only with an admin-read capability — factor rotation
is a separate change, so no spec-write surface is added here.

### D14: Updates spec gains repoUrl
The `Updates` singleton spec adds `repoUrl` (validated HTTPS URL); `channel`
(stable/preview) selects within the repo. Enables self-hosted mirrors and matches
ADR-0006's "repo URL + channel".

### D15: deferReboot governs trigger; autoApply is specified behavior
`trigger` stages, verifies, and sets the bootenv; it reboots immediately unless
`deferReboot=true`, in which case the new slot is booted at the next reboot and
completion is not claimed. The schema default for `deferReboot` flips to
`false`. `autoApply=true` uses the same path automatically when the configured
`repoUrl`+`channel` offers a newer bundle.

### D16: Store snapshot on update
The Update controller snapshots the spec store before staging/trigger (reusing
the store's `Snapshot()`/reload machinery) and the `rollback` action restores
that snapshot before flipping the bootenv and rebooting — so an older
rolled-back `core` never reads a newer schema (ADR-0013 restore, never
reverse-migrate).

### D17: Seed import precedes singleton seeding; seedUpdates added
`core` folds the seed in `New()` immediately after store load, before
`seedNetwork`/`seedTelemetry`/`seedOIDC`/`seedCertificates`/`seedAdmin`. When a
seed exists those singleton seeds skip (no empty-spec window, no
`resourceVersion` collision on the validated path) and the seed's admin is
authoritative over config-provided `SeedAdmin`. The `updates` singleton is
seeded alongside the others (empty spec: `channel=stable`, `autoApply=false`,
`deferReboot=false`) so the Update controller reconciles from first boot.
ADR-0031's D6 is amended with the seed-import step.

### D18: Argon2id parameters are a shared contract
The installer hashes the bootstrap admin password with core's pinned Argon2id
parameters (m ≤ 1 GiB, t ≤ 10, p ≤ 8); the parameters are declared once in
`contracts/` and consumed by both `infra` and `core`. The seed carries the hash,
not plaintext. ADR-0020's server-side-hashing rule applies to the API boundary;
the seed path is a deliberate carve-out recorded here.

### D19: Seed carries hostname / DNS / NTP
The seed body adds `hostname` (required) and `dns`/`ntp` (optional) alongside
network planes + per-interface IP (ADR-0014), matching the `Network` schema.

### D20: Seed carries full Disk + Pool + Dataset declarations
The seed's storage section carries `Disk` resources (by-id device, role incl.
`os` mirror membership per ADR-0011) plus `Pool` resources (poolName, immutable
vdev topology, spares per ADR-0024) and role-bearing `Dataset` resources,
because the storage controllers are resource-driven (adopt-on-boot) and the pool
controller resolves members from `Disk` resources — without seeded Disk
resources a seeded Pool would sit `pool_members_absent` forever. The installer's
`zpool create` matches the declared topology exactly. The seed also carries the
OS-disk by-id device, which the Disk controller's exclusion guard needs at
runtime and which the generic image cannot know at build time.

### D21: Installer ISO carries the trust anchor
The live ISO carries the same rauc trust anchor baked into the image, so the
initial `rauc install` verifies identically to an update — one honest
slot-writing path.

### D22: Automatic boot-fail rollback
A mark-good service runs after a successful boot; the bootloader boot-counts and
falls back to the prior slot on repeated failure; the controller reports the
automatic fallback in status. Alongside the manual `rollback` action this
fulfills ADR-0006's manual-and-automatic rollback promise.

The installed OS ships `efibootmgr`: rauc's EFI backend (`bootloader=efi`) shells
out to it for `rauc status mark-good` and slot activation, so the boot-time
mark-good service and the Update controller's rauc facade both need it present in
the OS root (the live ISO carries it too, D27). The rauc D-Bus service wiring
(system policy + `rauc.service`) that the live ISO bake adds is baked into the
installed OS as well, because the mark-good CLI and the Update controller are
D-Bus clients of `de.pengutronix.rauc` (D26/7.x verification).

### D23: Malformed seed fails closed
A seed that fails schema or semantic validation aborts the import and the API
never starts, surfacing a clear console error; reinstall is the only path to
re-seed. This is a deliberate fail-closed posture — degrading to an unconfigured
boot would silently drop the install-time state and mask real problems. Recorded
in the proposal's non-goals and enforced by the core seed-import step.

### D24: The OS image is built for arm64 and amd64
The mkosi build is parametrized by architecture: each arch selects its own
`linux-image-*` meta-package and carries its own per-slot kernel+initramfs on
the ESP, with the same A/B squashfs slot model and disk layout on both. Both
archs are product-supported; the amd64 build remains the canonical CI gate
(D11), and an arm64 build + smoke runs alongside it in CI.

### D25: Per-architecture repo channel segments
rauc bundles are architecture-specific (the payload includes the kernel and
initramfs), so each repo channel carries an architecture segment:
`<repoUrl>/<channel>/<arch>/latest.json` plus `<arch>/<version>/manifest.json`,
where `arch` is `arm64` or `amd64`. `Updates.repoUrl` stays a single HTTPS URL
and `channel` semantics are unchanged; the Update controller resolves the
`latest.json` for its **host** arch (an injectable resolver defaulting to
`runtime.GOARCH` on the baked per-arch core binary) and verifies the
repo-declared `arch` equals the host arch before staging — a second line of
defense because rauc's `compatible` stays `amberhold` for both archs and would
otherwise only fail at the next boot. This is a repo-format contract only; the
`Updates` OpenAPI schema is unchanged.

### D26: macOS qemu/HVF dev harness replaces the Linux-only qemu-matrix for local iteration
A Lima arm64 VM hosts the Linux-only build stage (mkosi, rauc bundle, repo
publish); the macOS host boots the built image natively under
`qemu-system-aarch64 -accel hvf` with edk2-aarch64 firmware, so systemd-boot
slot selection, LUKS unlock, and core startup are exercisable on a dev Mac
without Linux CI. The Lima VM mounts a host directory; mkosi output and the
published repo write there, the host serves the repo over HTTPS (core rejects a
non-HTTPS `repoUrl`, D14), and the qemu guest fetches it at the slirp gateway
`10.0.2.2`. The installer live ISO is built as a dedicated installer-only mkosi
preset (D27); the e2e currently drives that ISO headless (`amberhold.headless=1`)
as an interim, with driving the real console TUI over the qemu serial console
required by the follow-up task.

### D27: Installer-only live ISO carries the trust anchor and the initial bundle
The live ISO is a dedicated installer-only mkosi preset (console TUI installer +
host deps: mdadm, cryptsetup, kpartx, rauc — not the OS root). It carries the
same rauc trust anchor baked into the image (D21) **and** an initial signed OS
bundle, mounted at `/media/amberhold/amberhold.raucb` per the installer's
default, so a fresh install has a bundle to `rauc install` without reaching the
repo and verifies identically to an update.

### D28: The images bake the ZFS kernel module at build time
Both the OS image and the installer live image carry the ZFS kernel module
(`zfs.ko`) for their own kernel. Debian ships no prebuilt module — `zfs-modules`
is a virtual package provided only by `zfs-dkms`, which compiles against the
target kernel headers — so the mkosi build compiles the module at build time and
installs it into the image module tree (D4/ADR-0001). Without it the installed
system's `zfs-load-module.service` fails, `modprobe zfs` fails, and every `zpool`
operation (the `Pool` controller's probe, `zpool create`, import/scrub) is
unreachable — the storage plane designed in `docs/architecture/03-storage-controller.md`
and ADR-0004 is dead. The installer's `CreateStorageRoles` (`zpool create`) fails
the same way, so the live image bakes the module too. Implementation:

- **Build-time DKMS, per image tree.** The kernel version is discovered from the
  image's `/usr/lib/modules/*` and never hardcoded, so it tracks the per-arch
  `linux-image-*` meta-package (D24). The module is built with `dkms` from
  `zfs-dkms` against the matching per-arch `linux-headers-*` and staged into the
  image module tree; mkosi then runs `depmod` for the image kernel, so
  `modprobe zfs` resolves.
- **Build-only tooling never ships.** `dkms`, `zfs-dkms`, the kernel headers, and
  the compiler are `BuildPackages` (build overlay only, D4's package flow), not
  image packages: the final OS image carries `zfsutils-linux` plus the built
  module, no toolchain. A build-time assertion fails the build if the module (or
  its `modules.dep` entry) is missing, so the regression cannot ship silently.
- **Userspace version match.** The module builds from the `zfs-dkms` source, which
  is the same source version as the shipped `zfsutils-linux` userspace; the build
  asserts `zfs-dkms` == `zfsutils-linux` and that the built module version equals
  the userspace version, so module and userspace cannot silently desynchronize.
  ZFS stays on trixie `contrib` (no version upgrade, no backports move).
- **Not in the initramfs.** The initramfs unlocks the dm-crypt container and
  mounts the squashfs root; it never touches ZFS, so `mkosi.initrd.conf`'s
  `KernelModules=` list is unchanged and the module is loaded from the real root
  by `zfs-load-module.service`. There is no runtime DKMS: the A/B image replaces
  the kernel wholesale on update, so a baked-at-build-time module is enough.

### D29: The installer exports created pools and seeds stable member identity
`CreateStorageRoles` creates the data/app pools on the live installer system and
**exports every pool it created before the install completes** (host facade
`zpool export <name>`; "no such pool"/"not imported" is treated as success so an
idempotent re-run is safe). Without the export the pool is "last accessed by
another system" and `core`'s **non-forced** `zpool import` fails — the seeded
`Pool` sits `Degraded`/`pool_import_failed` and the storage plane is unusable
until manually imported. The installer owns the pool it just created and is
about to hand the OS to the installed system, so releasing it is the correct,
least-privilege action; `zpool import -f` in `core` is deliberately **not** used
(it can seize a pool in active use by another system and would hide an installer
that leaves host state dirty). An export failure aborts the install rather than
leaving a pool imported for a system that cannot adopt it.

Pool members, the seed's `Disk.byIdDevice`, and the `Disk` resource's `device`
SHALL all be the stable `/dev/disk/by-id` path, so the controller's topology
comparison converges across the install→boot kernel device-node renaming
(`docs/architecture/03-storage-controller.md` §5, D-S3). Real hardware
(NVMe/SATA) always exposes by-id (WWN/serial); the qemu dev harness attaches its
virtio-blk data disks with stable serials so `/dev/disk/by-id/virtio-<serial>`
exists (D26). `core` additionally resolves the partition paths ZFS reports for a
whole-disk vdev (`/dev/disk/by-id/virtio-X-part1`) back to the parent whole-disk
by-id identity before comparing topology (ADR-0034, D-S14), so a healthy pool is
never reported as `topology_immutable` on ZFS's partition alias.

### D30: The image provides a writable dataset mount root at `/mnt`

The installed root is a read-only squashfs slot, so ZFS's default pool mountpoint
`/<pool>` cannot be created and `zpool create` fails with
`cannot mount '/<pool>': ... Read-only file system` even though the pool itself
was created (ADR-0034). The image therefore provides one writable dataset mount
root:

- `/mnt` exists in the read-only root as an empty directory; the image
  bind-mounts the OS-disk `config/var` partition's `mnt/` directory over it, so
  the mount root is writable and persistent without mutating the immutable slot.
- The bind is ordered **before** ZFS pool import/mount
  (`Before=zfs-import.target zfs-mount.service`, `After=config-var.mount`), so
  every dataset mountpoint can be created when a pool is imported.
- Pools carry the mountpoint `/mnt/<pool>`; the installer records it with
  `zpool create -m /mnt/<pool>` before exporting — the live installer root is
  writable, so the create-time mount succeeds and the export releases it — and
  `core` creates with `-m /mnt/<pool>` on the installed system
  (`docs/architecture/03-storage-controller.md` §16, D-S14). (`-N` is a `zpool
  import` option, not a `zpool create` option.) Datasets inherit
  `/mnt/<pool>/<dataset>`; the mountpoint is not part of the API contract.
- Pools created before this decision keep a root-level mountpoint and are not
  migrated automatically (ADR-0034).

### D31: The image ships and runs the NFS server for file-share exports

`core`'s NFS backend drives the ZFS `sharenfs` dataset property, but the actual
export is performed by ZFS's share path, which on Linux shells out to the kernel
NFS server userspace (`exportfs`/`rpc.nfsd`/`rpc.mountd`). The image previously
shipped only the SMB server, so a dataset with `sharenfs` set was never exported
and an NFS `FileShare` could not serve traffic. The OS image now
(`add-nfs-server-and-share-ports`; shares design §15 in
`docs/architecture/04-shares-controller.md`):

- adds `nfs-kernel-server`, `nfs-common`, and `rpcbind` to `[Content] Packages`
  (`os-image/mkosi/mkosi.conf`);
- bakes `/etc/nfs.conf` with the auxiliary ports pinned — `nfsd` 2049, `mountd`
  20048, `statd` 32765 (`rpcbind` 111 is fixed by the protocol) — so the exports
  are reachable through the port-forwarding dev harness (slirp `hostfwd` can
  only forward known ports) as well as a normal network, and are
  firewall-friendly;
- enables `nfs-server.service`, `rpcbind.socket`/`rpcbind.service`, and
  `zfs-share.service`, and loads the `nfsd` module at boot, in the bake postinst
  (`os-image/mkosi/mkosi.postinst.d/90-bake-amberhold.sh.chroot`). `zfs-share`
  runs `zfs share -a`, re-exporting `sharenfs` datasets across a reboot.

The installer live ISO does not serve shares and is unchanged. No contract change
and no `core` change: the NFS backend already drives `sharenfs`.

## 4. Reconciliation and boot

The `UpdateController` owns the singleton `updates` kind and reconciles the
desired `repoUrl`+`channel` against the `Rauc` host facade (task 6.2). Each pass:

- **Repo/channel**: when the host repo URL or channel differs from the spec,
  reconfigure the rauc repo and re-list available bundles for the host
  architecture (D25); idempotent no-op when converged. The repo listing is
  throttled (so a fast resync never hammers the HTTPS repo), but the throttle is
  keyed on the configured URL + channel: a changed repo is listed immediately
  rather than masked by the previous repo's cached availability.
- **autoApply**: when `autoApply` is true and the repo offers a newer bundle than
  the active slot, stage/verify/activate through the same path as `trigger`,
  honoring `deferReboot` (D15).
- **Status**: active/staged slot names, pending-reboot state, last result
  (including automatic fallback, D22), last error, available version, progress,
  and the UnlockPolicy-driven pre-reboot warning.
- **Actions**: `trigger` and `rollback` are imperative (ADR-0031 D7); they never
  mutate desired state. A store snapshot is taken before staging and restored on
  rollback (D16).

The `UnlockPolicyController` owns the `UnlockPolicy` singleton (D13): it mirrors
observed enrolled factors from `cryptsetup luksDump` / `systemd-cryptenroll`
into status, keeps the desired per-mode policy in spec, and reports drift.

**Boot chain** (D2/D3/D22): systemd-boot selects the active slot's per-slot
kernel+initramfs from the plain ESP (no unlock to select); the initramfs-tools
hook assembles `md0` degraded when a mirror member is missing, unlocks the
container with an enrolled factor, reports rauc boot-status, and mounts the
active slot. A mark-good service records a successful boot; repeated failure
boot-counts and falls back to the prior slot automatically (D22).

## 4a. macOS dev harness and per-arch repo

The arm64 image and the boot chain are exercised locally on a macOS development
host (D24–D26). The division of labor across the two Linux environments is:

```
   macOS host                                    Lima arm64 VM (Linux)
   ─────────────────────────                    ─────────────────────────
   qemu-system-aarch64 -accel hvf               mkosi build (arm64 .raw)
   edk2-aarch64 firmware                        rauc bundle + publish
   harness driver (scenarios, disks,            -> repo served to qemu guest
     injects, asserts)                           (per-arch segment)
```

The harness boots what Lima builds; Lima publishes what qemu consumes — no CI in
the loop. The Lima VM mounts a host directory (virtiofs/9p); mkosi's output and
`publish-bundle.sh` write into it so the image/ISO/repo are host-visible without
a copy-out. The host serves the repo over HTTPS with a locally-generated
CA/cert (the CA is baked into the image so the Update controller trusts it) and
the qemu guest reaches it at the slirp gateway `10.0.2.2`. The core-vs-real-host
"fake repo" (D25) is this same host HTTPS server over the shared dir.

The Update controller resolves `<repoUrl>/<channel>/<arch>/latest.json` for its
host arch and verifies the repo-declared bundle `arch` before staging (D25). The
per-arch repo layout:

```
   /repo/                               # Updates.repoUrl root (unchanged)
     stable/                            # channel from Updates.spec
       arm64/
         latest.json                    # current arm64 stable bundle pointer
         1.2.3/amberhold-1.2.3.raucb
         1.2.3/manifest.json            # version, sha256, arch, publishedAt
       amd64/
         latest.json
         1.2.3/amberhold-1.2.3.raucb
         1.2.3/manifest.json
```

`publish-bundle.sh` takes an `arch` argument and writes both the version
`manifest.json` (declaring `arch`) and the per-arch `latest.json`; each arch
publishes independently and its `latest.json` points at its own arch's bundle.

Under qemu the UnlockPolicy observed-factor kinds are exercised with a
USB-keyfile factor (presence-only, real cryptenroll, no hardware); FIDO2
touch-mode stays deferred to hardware/Linux CI. The installer e2e is intended to
drive the real console TUI over the qemu serial console (not the `--headless`
env path), which keeps the admin password off `/proc/cmdline`; as an interim the
scenario currently drives the headless installer (D26).

The `core-vs-real-host` scenario (D25) installs with data disks, boots the
installed image, and requires the seed `tank` pool to reach **`Ready`** — the
end-to-end assertion of D1 (installer export) and D2 (stable by-id member
identity); it fails loudly on `pool_import_failed`/`pool_probe_failed`. It then
re-asserts `Ready` after one periodic resync (D-S3: a whole-disk vdev is
reported by `zpool status` as ZFS's partition alias, so a pool that only looked
Ready transiently would flip to `Error`/`topology_immutable` here), requires the
seeded `tank/data` dataset to report `mounted` (its ZFS mountpoint inherits
`/mnt/tank/data`, so a mounted dataset proves the writable `/mnt` dataset mount
root is in place — D30/D-S14), and fails on a `cannot mount '/tank'`/read-only
mount error in the boot log. Two harness prerequisites it documents: the repo CA
must be baked into the image (the CA is generated by `serve_repo` and staged
into the build sources, so the image is built **after** that staging step), and
the qemu data disks are attached with stable serials so
`/dev/disk/by-id/virtio-<serial>` exists (the serial-less virtio default has no
by-id alias, which is exactly the identity instability D2 fixes). The manual
companion check is `zfs get mountpoint tank` → `/mnt/tank` inside the guest.

The harness boots edk2 with a **writable vars pflash** (`amberhold-vars.fd`,
initialized from the firmware) shared across the install and target boots: on a
UEFI install rauc's EFI backend writes the per-slot `BootOrder`/`Boot####` NVRAM
entries at install time and reads them back for `rauc status mark-good` and slot
activation (D22). With only the read-only code pflash the NVRAM is volatile and
`rauc status mark-good` fails with "Did not find primary boot entry" — the
boot-count fallback (7.x) is only exercisable when the NVRAM persists.

The interim headless install boots the live image with qemu `-kernel`/`-initrd`
(and `root=/dev/vda2`, which avoids an initrd by-partuuid race), which bypasses
UEFI and leaves the guest without efivars. The installer detects the missing
UEFI runtime and skips only the `efibootmgr` A/B entry creation, still writing
the systemd-boot files + per-slot bootspec entries to the ESP; the installed
target then boots under edk2 via systemd-boot's removable path, so slot
selection and the boot chain are still exercised. A real UEFI install has
efivars and still fails hard if entry creation fails, so a production system is
never left unable to switch slots.

Before it boots the installed target, the harness **verifies the installed
boot files** (D26, D3): `wait_install` waits for the `installer ok` marker, gives
the guest a bounded flush window, then calls
`verify_installed_boot_files <image> <bootname...>`. That helper is
self-contained on the host — no Lima and no loop mounts — parsing the image's
GPT to locate the EFI System Partition and then the ESP's FAT32 directory
entries (including long-file-name entries, since `amberhold` exceeds 8.3) to
assert each `/amberhold/<bootname>/{vmlinuz,initrd}` has a non-zero size. A
missing or zero-size boot file fails the install with a diagnostic naming the
file, so a truncated install surfaces immediately instead of looping on the
edk2 firmware assert. The harness's qemu drives also use `cache=writethrough`
(and the installer's own sync+unmount teardown above), so a guest killed with
`SIGTERM` right after `installer ok` cannot drop the last ESP writes.

In addition to the management front door (`:443`) and the update repo, `boot_vm`
forwards the guest's data-plane share listeners to the host so an operator can
mount the guest's SMB/NFS shares from the dev Mac for interactive testing
(`add-nfs-server-and-share-ports`): SMB `:445`/`:139` and NFS `:2049`, plus
rpcbind `:111`, mountd `:20048`, and statd `:32765` (tcp and udp). Host ports
default to unprivileged values (`1445`/`1139`/`12049`/`1111`/`12048`/`13265`) so
the harness needs no root and does not collide with a host SMB/NFS server, and
are overridable through `AMBERHOLD_SMB_PORT`/`AMBERHOLD_NETBIOS_PORT`/
`AMBERHOLD_NFS_PORT`/`AMBERHOLD_RPCBIND_PORT`/`AMBERHOLD_MOUNTD_PORT`/
`AMBERHOLD_STATD_PORT`. Two caveats are printed at boot: a host-originated share
connection appears to the guest as the slirp gateway `10.0.2.2`, so an NFS
`hosts` grant / SMB access rule must allow it (or use the single-LAN default);
and because the host forwards non-standard ports, an NFSv3 client must be given
them explicitly (`mount -t nfs -o vers=3,port=<nfs>,mountport=<mountd> ...`),
with NFSv4 (`vers=4,port=<nfs>`) the simpler fallback.

## 5. Seed-manifest handoff

```
installer (live ISO, TUI)
   │  1. preflight + sizing      (D9, ADR-0011 sizing floor)
   │  2. partition / md0 / LUKS2
   │  3. enroll factors + policy  (ADR-0011)
   │  4. rauc install initial bundle (verified vs ISO trust anchor, D21)
   │  4a. sync + unmount the ESP(s)   (per-slot boot files durable, D7)
   │  5. storage roles: fresh zpool create / recover zpool import (D9)
   │  6. network + admin capture   (ADR-0014, D18)
   └─► writes seed manifest → spec-store partition (JSON, versioned, D8/D19/D20)
                     │
                     ▼ first boot
   core  store load → seed import (validate + fold, D23 fail-closed)
         → seed consumed → singleton seeds skip (D17) → serve API
```

The seed is validated against the `contracts` seed-manifest schema; `core`
imports it through the existing validated write path, so validation is core's,
not duplicated. The OS-disk by-id device in the seed feeds the Disk controller's
exclusion guard (D20).

## 6. Status and observability

The Update controller writes the extended `UpdatesStatus` shape (active/staged
slot, pending-reboot, last result incl. automatic fallback, last error,
available version, progress, pre-reboot warning) and publishes
`amberhold.updates.*` metrics from the catalog (`contracts/metrics/catalog.yaml`)
through the daemon registry (ADR-0008). The UnlockPolicy controller mirrors
enrolled factors into `UnlockPolicy` status. All controllers write status through
the store's serialized path and never build providers themselves.

## 7. Risk / trade-offs

- **rauc dual-ESP + LUKS integration is custom** → the qemu matrix tests the
  exact boot/activate/rollback paths; the initramfs boot-status hook is
  validated against boot-fail scenarios before the update path relies on it
  (D11, ADR-0006).
- **Seed-manifest validation drift from the store's own validators** → the seed
  schema lives in `contracts/` and core imports it through the existing
  validated write path (D8).
- **One large change is review-heavy** → tasks.md groups by submodule with
  per-task verification; each phase is independently buildable/testable (D12).
- **mkosi build reproducibility across host environments** → pinned package
  versions (ADR-0001) and the nspawn image check in CI (D11).
- **Update reboot interruption (touch-mode factor)** → the unlock-policy mirror
  from the seed and the pre-reboot warning (D13, D15) surface attendance needs
  before the reboot.
- **bbolt on ext4 inside LUKS** → covered by the existing store's fsync path;
  the nspawn image check exercises boot-to-core on ext4 (D5, D11).
- **Rollback across a schema change boots an older core** → the Update
  controller snapshots the store before trigger and restores it on rollback
  (D16).
- **The qemu-matrix never exercised systemd-boot under UEFI** → the harness
  boots the edk2-aarch64 pflash firmware, the first real test of slot selection;
  expect firmware/bootloader iteration (D26).
- **Nested virtualization is unavailable in Lima** → the harness boots qemu on
  the macOS host (HVF), not inside Lima; Lima is only the build/rauc stage
  (D26).
- **arm64 parity in CI** → adds a second image arch to build and verify; the
  amd64 gate is unchanged and authoritative (D24).
- **A mispublished wrong-arch bundle** → the repo `arch` field is verified
  against the host arch before staging, since rauc `compatible` is shared across
  archs (D25).
- **An install target without by-id device aliases** → the installer prefers
  `/dev/disk/by-id`, but an unusual virtio/loop setup can lack an alias and fall
  back to a kernel path the kernel renames across boot, leaving adoption's
  exact-string topology comparison reading drift (D29). Real NVMe/SATA hardware
  always exposes by-id; the harness attaches disks with serials. Normalizing
  device identity inside `core`'s comparison is the documented, deferred
  fallback (it weakens the immutable-topology check and can mask a swapped
  disk).

## 8. Migration plan

Greenfield: no existing installs. Deploy order within the change: os-image +
installer first (produces a bootable system), then the update repo + Update
controller on top. Rollback: revert the `infra` and `core` commits; there is no
state-migration risk because the spec store is only ever written by core via the
validated path. The seed manifest is consumed once and never re-imported, so a
reinstall is the only path to re-seed.

## 9. Open questions (implementation-time)

None that would change the specs, approach, or task breakdown. rauc `system.conf`
slot naming and whether the EFI backend needs kernel/initramfs slots declared
explicitly vs. via a custom post-install hook, and whether `touch` mode should
also require a FIDO2 PIN, are settled during implementation without changing
specs (design open questions).
