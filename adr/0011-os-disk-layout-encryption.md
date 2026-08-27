# ADR-0011: OS disk — layout, encryption, and data encryption keys

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — OS disk is now LUKS2-encrypted, keyfiles inside the
  container (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, §3.7, feature 2,
  feature 12; ADR-0001, ADR-0006, ADR-0007, ADR-0013 (writable state),
  ADR-0030; `host-level-encryption` change

> Consolidates the former ADR-0011 (dedicated OS disk + A/B slots), ADR-0031
> (whole-OS-disk LUKS2 encryption with external factors), and ADR-0015 (data
> encryption keyfile / headless unlock). One OS-disk story: physical layout,
> sealing, and where data-encryption keys live. The writable *state* that lives
> on this disk (spec store, `config/var`) is ADR-0013.

## Context

ADR-0001 fixes the root as a read-only squashfs image with A/B dual-slot boot,
and ADR-0007 makes the roles of *data* datasets configurable at install. Neither
constrains where the two OS slots, the bootloader, and system config state
physically live. The boot media is the one part of the topology that cannot be a
configurable dataset, and it is a prerequisite for the `os-image`, `installer`,
and `system-update` designs. Separately, feature 2 lists native ZFS encryption as
minimal scope but nothing decided where keys live or how encrypted datasets
unlock. The former ADR-0015 protected only against removal/theft of the
*data disks*; stealing
the whole host yielded keys + data together, so whole-host-theft protection
requires sealing the OS disk itself and gating boot on an external factor, while
preserving the headless update flow.

## Decision

The A/B squashfs slots, the bootloader, and all system config state live on a
**dedicated OS disk** (e.g. a 128–256 GB SSD/NVMe). ZFS pools are pure data; ZFS
is never the boot media. The OS disk is fixed (one disk, two slots + bootloader +
two persistent partitions), so boot never depends on pool import. Only the `data`
(and optional `app`) roles are configurable per ADR-0007; system config state is
fixed on this disk, not a pool. There are no fs-level snapshots on the OS disk.

### Layout and sealing

The entire post-ESP OS disk is sealed in a **single LUKS2 container** — A/B
squashfs slots, the spec store + data-encryption keyfiles, and `config/var` —
with the ESP as the only plain partition. The container is unlocked **only at
boot**, by the initramfs, using external factors; there is no post-boot unlock
path.

```
+---------------------------- ESP (plain) ---------------------------+
| bootloader | slotA: kernel+initramfs | slotB: kernel+initramfs | bootenv |
+----------------------------------------------------------------------+
                        | unlock via FIDO2 / USB keyfile / passphrase
                        v
+------------------- OS disk: LUKS2 container -------------------------+
| slot A (RO sqfs) | slot B (RO sqfs) | spec store + keyfiles | config/var |
+-------------------------------------------------------------------------+
```

- **Boot chain (A/B-at-ESP + unlock-then-mount)**: kernel + initramfs live per
  slot on the plain ESP. The bootloader selects the active slot's kernel+initramfs
  by bootenv — no unlock needed to *select* a slot, only to *mount* it. The
  initramfs unlocks the container, then mounts the active slot's rootfs.
  Boot-fail fallback to the other slot operates at the ESP level.
- **Unlock factors**: YubiKey FIDO2 (LUKS2 systemd token, enrolled with
  `--fido2-with-user-presence`), USB keyfile keyslot, and an offline recovery
  passphrase as a final keyslot. Multiple factors are enrollable concurrently.
- **Per-mode unlock policy**: `presence-only` (factor present, no touch/PIN — the
  headless update reboot keeps working, ADR-0006) vs `touch` (interactive proof,
  e.g. YubiKey touch or PIN — update reboots need attendance). The mode is a
  property of how the FIDO2 credential is enrolled; the host-level `UnlockPolicy`
  resource (spec store, read post-boot) mirrors the enrolled slots and drives the
  Update controller's pre-reboot warning.
- **Factor rotation**: on a booted host, `core` shells out to
  `cryptsetup luksAddKey/luksRemoveKey` and `systemd-cryptenroll enroll/wipe`
  (pinned versions, ADR-0001 pattern) to add/remove/replace factors. ZFS keys are
  never touched — factor changes mutate only LUKS keyslots. Authorization requires
  an existing enrolled factor or the recovery passphrase.

### Data-encryption keys and headless unlock

Encrypted datasets use a keyfile stored on the OS-disk spec store partition (next
to the spec store, ADR-0013), auto-loaded by `core` at start before any pool
import. There is no passphrase unlock at boot; protection is at-rest only (keys
are co-located with the system that holds the data). With host encryption, the
keyfiles live **inside** the OS-disk LUKS2 container — no longer plaintext on an
unencrypted partition, only readable once the container is unlocked. Headless
unlock is preserved through `presence-only` external factors; the offline recovery
passphrase provides factor-loss recovery. Because keys live on the OS disk rather
than on a pool, boot never depends on pool import to read them — the unlock opens
the container, then the keyfiles are read from inside it. Pool and app datasets are
ZFS-native-encrypted with these keys; the threat model is protection against
removal/theft of the data disks alone, and with the OS-disk container, whole-host
theft.

## Alternatives considered

- **Partitions on a pool disk**: saves a disk, but couples boot to pool membership
  and reintroduces the boot-depends-on-pool-state coupling ADR-0001 exists to avoid.
- **EFI/ESP-based slots**: no separate disk needed, but the ESP is small and sizing
  two squashfs images on it is atypical.
- **System config on a `config` pool dataset (earlier ADR-0007 role)**: gave ZFS
  snapshots of config/logs, but coupled OS state to pool availability and made boot
  depend on pool import; rejected in favor of plain OS-disk partitions.
- **Passphrase unlock via UI/API**: more separation between key and data, but
  breaks headless reboot during updates and needs a boot-complete-but-pools-locked
  state.
- **External key management (TPM/KMS)**: better separation, but adds hardware
  dependence and a key-service dependency out of proportion to v1 scope.
- **LUKS per partition**: more unlock events and keyslot bookkeeping per partition;
  rejected — one container = one unlock event = one keyslot set.
- **Secret-partitions-only LUKS with plain slots**: encrypting only the secret
  partitions fails the uniform "host is sealed" story; rejected even though
  encrypting the public squashfs adds little confidentiality on its own — the
  decision trades that for a single, uniform sealing model.
- **Network/remote unlock** (dropbear-in-initramfs style): rejected — factors are
  physical; unlock is physical/console only.
- **In-place re-encryption of existing installs**: encryption is an install-time
  decision (ADR-0007 pattern); there are no legacy installs in v1.
- **A dedicated keyslot-manager daemon**: more moving parts than shell-out;
  rejected in line with the restic/zfs shell-out pattern.

## Consequences

- The installer and hardware planning must account for a dedicated OS disk
  (feature 12, `installer`), including a 128–256 GB sizing floor for slots + spec
  store + `config/var`, plus the LUKS container overhead and per-slot
  kernels/initramfs on the ESP. The installer creates the container and enrolls
  initial factors (USB keyfile + YubiKey FIDO2 + recovery passphrase) at first
  boot.
- The bootloader selects the active slot at the ESP level — A/B selection and
  rollback operate on the per-slot kernel+initramfs and never touch pools; only
  mounting the active rootfs requires the unlock. The A/B tooling candidate must
  support LUKS-encrypted slots with initramfs unlock (ADR-0001 constraint).
- Reboot and update flows (ADR-0006) work unattended while a `presence-only`
  factor is present; a `touch`-mode factor or a missing factor blocks at the unlock
  prompt until attendance (Update controller warns first).
- Data pools can be created, imported, or rebuilt independently of the OS disk —
  including the reinstall recovery path (`zpool import`). With native encryption
  this holds for unencrypted datasets only: the keyfiles live on the OS disk, so
  encrypted pools are lost with it.
- Whole-host theft no longer yields keys + data: the OS disk is sealed and the data
  pools stay ZFS-native-encrypted, whose keys are inside the sealed container.
- A locked host cannot boot the OS at all — there is no locked-but-booted state and
  no network unlock. Observability while locked is console-level (unlock prompt)
  until boot completes; once booted, `core` records the unlock event (factor kind,
  success/failure) and exposes a host lock-status gauge via resource status and
  Prometheus metrics (ADR-0008).
- The OS disk is a single point of failure: it holds slots, the bootloader, the
  spec store + keys, and `config/var` — all inside the LUKS2 container except the
  ESP. OS-disk loss loses keys, spec store, repository password, and data. Losing
  the OS disk means losing access to encrypted data; the reinstall + `zpool import`
  path (ADR-0013) recovers unencrypted data only. Factor loss (not OS-disk loss) is
  recoverable via the recovery passphrase. Redundancy for the OS disk is accepted
  as out of scope for v1.
- The restic repository password (ADR-0030) lives on the spec-store partition
  inside the container and is auto-loaded only after unlock; its recovery boundary
  is unchanged.

## Open questions

- Exact A/B tool selection (rauc/ostree/ABRoot) — deferred to the `os-image`
  design; this ADR pins the LUKS-with-initramfs-unlock requirement.
- Whether `touch` mode should also require a PIN (FIDO2 user-verification) in
  addition to presence — a per-host enrollment option, no architecture impact.
