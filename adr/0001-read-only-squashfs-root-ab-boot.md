# ADR-0001: Read-only squashfs root with A/B boot

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — initramfs boot step and LUKS support constraint (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, decision 1; ADR-0006,
  ADR-0009, ADR-0011, ADR-0031

## Context

The OS image must be updatable and must not drift from a known-good state. Two
candidate root strategies exist: keep the root on ZFS (rich snapshot/rollback of
OS state) or keep it on a plain read-only partition.

## Decision

The root filesystem is a read-only squashfs image; ZFS is reserved exclusively
for data pools. Updates are delivered as new images in an A/B dual-slot layout;
the bootloader selects the active slot and a rollback falls back to the other
slot. The slot/bootloader mechanism is delegated to a standard A/B tool (rauc /
ostree / ABRoot candidates); the specific tool is pinned in the image and the
slot layout is fixed on the dedicated OS disk (ADR-0011), resolved in the
`os-image` design.

With host encryption (ADR-0031), the OS disk's contents live inside a LUKS2
container and the rootfs is not mountable until it is unlocked. Kernel and
initramfs therefore live per slot on the plain ESP: the bootloader selects the
active slot's kernel+initramfs by bootenv, the initramfs unlocks the OS-disk
container, and only then mounts the active slot's rootfs (A/B-at-ESP +
unlock-then-mount, ADR-0011). No unlock is needed to *select* a slot — only to
*mount* it — so boot-fail fallback still works at the ESP level. The A/B tooling
candidate must support LUKS-encrypted slots with initramfs unlock; candidates
that cannot unlock before mounting the rootfs are excluded.

## Alternatives considered

- **ZFS-on-root**: rich OS-state snapshots, but couples boot to pool state and
  complicates the bootloader/kernel-module story.
- **Plain read-only partition**: simpler but offers no atomic slot switching or
  update staging.

## Consequences

- No post-deploy package installs. Kernel modules (ZFS, `nvme-target`), samba,
  and containerd must be baked into the image; image build is a first-class
  feature.
- The `core` daemon is also baked into the image; core updates are OS updates and
  flow through the update feature (ADR-0006), keeping the root immutable.
- All writable state (spec store, keyfiles, generated configs, logs/audit, app
  images, user data) lives on the OS-disk partitions (ADR-0011) or configured
  app/data datasets (ADR-0007), never on `/`. With host encryption (ADR-0031)
  the OS-disk writable state is inside the LUKS2 container and only readable
  after unlock.
- Boot includes an initramfs step: kernel + initramfs per slot on the plain ESP
  (ADR-0011), the initramfs unlocks the OS-disk container with an enrolled
  factor (YubiKey FIDO2 / USB keyfile / recovery passphrase, ADR-0031) before
  mounting the active slot's rootfs. A/B slot selection and rollback happen at
  the ESP level and do not require the unlock.
- No subsystem writes host files under `/` — including `/etc/exports`, which is
  not used (NFS shares are ZFS `sharenfs` properties, ADR-0009).
- Daemons that need writable config follow one convention: the image ships a
  symlink from the RO `/etc` path to a generated file on the OS-disk `config/var`
  partition (e.g. `/etc/smb.conf`), and the owning controller regenerates the file
  on the `config/var` partition. No daemon config lives in the immutable image
  (ADR-0007, ADR-0011).
- A/B = two slots; rollback = fall to the other slot; boot failure falls back
  automatically.
