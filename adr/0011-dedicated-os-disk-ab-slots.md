# ADR-0011: Dedicated OS disk for the A/B squashfs slots

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — OS disk is now LUKS2-encrypted (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, §3.7;
  `resolve-architecture-gaps` D1; ADR-0001, ADR-0007, ADR-0031

## Context

ADR-0001 fixes the root as a read-only squashfs image with A/B dual-slot boot,
and ADR-0007 makes the roles of *data* datasets configurable at install. Neither
constrains where the two OS slots and bootloader physically live. The boot media
is the one part of the topology that cannot be a configurable dataset, and it is
a prerequisite for the `os-image`, `installer`, and `system-update` designs.

## Decision

The A/B squashfs slots, the bootloader, and all system config state live on a
dedicated OS disk (e.g. a 128–256 GB SSD/NVMe). ZFS pools are pure data; ZFS is
never the boot media. The OS disk is fixed (one disk, two slots + bootloader +
two persistent partitions), so boot never depends on pool import. Only the
`data` (and optional `app`) roles are configurable per ADR-0007; system config
state is fixed on this disk, not a pool. With host encryption (ADR-0031), the
OS disk is sealed in a single LUKS2 container (slots, spec store + keyfiles,
and `config/var`); only the ESP stays a plain filesystem. There are no
fs-level snapshots. Two distinct failure/recovery
senses apply: accidental corruption or loss of on-disk state is protected by
spec-store self-versioning (ADR-0013), config-fragment regeneration, and
journald rotation/retention (ADR-0016); whole-disk loss is not protected that
way — recovery is reinstall + `zpool import`, which recovers unencrypted data
only (consequences, and ADR-0015 for the encryption caveat).

Layout of the OS disk (with host encryption, ADR-0031):

```
┌──────┬──────────────────────┬──────────────────────────────────────────┐
│  ESP │  OS disk: LUKS2 container (unlocked at boot)                    │
│(plain)│ ┌──────────┬──────────┬──────────────────────┬───────────────┐ │
│boot- │ │ Slot A   │ Slot B   │ spec store + keyfiles │ config/var   │ │
│loader│ │ RO sqfs  │ RO sqfs  │ (self-versioned)      │ logs·audit·conf│ │
│+ k+ir│ └──────────┴──────────┴──────────────────────┴───────────────┘ │
│per-slot                                                               │
└──────┴───────────────────────────────────────────────────────────────┘
```

Kernel + initramfs live per slot on the plain ESP; the initramfs for the active
slot unlocks the LUKS2 container before mounting its rootfs (A/B-at-ESP +
unlock-then-mount, ADR-0001, ADR-0031). The sizing floor accounts for the LUKS
container overhead plus per-slot kernels/initramfs on the ESP.

## Alternatives considered

- **Partitions on a pool disk**: saves a disk, but couples boot to pool
  membership and reintroduces the boot-depends-on-pool-state coupling ADR-0001
  exists to avoid.
- **EFI/ESP-based slots**: no separate disk needed, but the ESP is small and
  sizing two squashfs images on it is atypical.
- **System config on a `config` pool dataset (earlier ADR-0007 role)**: gave ZFS
  snapshots of config/logs, but coupled OS state to pool availability and made
  boot depend on pool import; rejected in favor of plain OS-disk partitions.

## Consequences

- The installer and hardware planning must account for a dedicated OS disk
  (feature 12, `installer`), including a 128–256 GB sizing floor for slots +
  spec store + `config/var`, plus the LUKS container overhead and per-slot
  kernels/initramfs on the ESP (ADR-0031).
- The bootloader selects the active slot at the ESP level — A/B selection and
  rollback operate on the per-slot kernel+initramfs on the plain ESP and never
  touch pools; only mounting the active rootfs requires the unlock
  (ADR-0001, ADR-0031).
- Data pools can be created, imported, or rebuilt independently of the OS disk —
  including the reinstall recovery path (feature 12 `zpool import`). With native
  encryption (feature 2, ADR-0015) this holds for unencrypted datasets only: the
  encryption keyfiles live on the OS disk, so encrypted pools are lost with it.
- The spec store and keyfiles live inside the OS-disk LUKS2 container
  (ADR-0013, ADR-0015, ADR-0031), so boot never depends on pools; spec-store
  versioning/snapshot headroom drives the disk size check at install. The
  container must be unlocked before these are readable.
- `config/var` sits inside the OS-disk LUKS2 container (ADR-0031): forwarded
  logs/audit (ADR-0016) and generated daemon config fragments (ADR-0001) are
  encrypted at rest once the container is unlocked and remain protected by
  rotation and regeneration, not snapshots. App images and compose state stay on
  the ZFS `app` dataset (ADR-0004), so they are unaffected by OS-disk loss —
  unless that dataset is encrypted (ADR-0015), in which case the encryption
  caveat below applies.
- The OS disk is a single point of failure: it now holds slots, the bootloader,
  the spec store + keys (ADR-0013, ADR-0015), and `config/var` — all inside the
  LUKS2 container except the ESP. Unencrypted
  data pools and app images are unaffected by OS-disk loss and recoverable via
  reinstall + `zpool import`; encrypted datasets are not recoverable because
  their keys are lost with the disk (ADR-0015). Redundancy for the OS disk is
  accepted as out of scope for v1.
