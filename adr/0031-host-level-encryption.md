# ADR-0031: Whole-OS-disk LUKS2 encryption with external unlock factors

- Status: accepted
- Date: 2026-08-27
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 2 (feature 12, §3.7);
  ADR-0001, ADR-0006, ADR-0011, ADR-0013, ADR-0015, ADR-0016, ADR-0030;
  `host-level-encryption` change

## Context

ADR-0015 protects only against removal/theft of the *data disks*: ZFS keyfiles
sit in plaintext on the OS-disk spec-store partition, and ADR-0011 deliberately
kept the OS disk unencrypted ("no LUKS"). Stealing the whole host therefore
yields keys + data together. Whole-host-theft protection requires sealing the
OS disk itself and gating boot on an external factor, while preserving the
ADR-0006 headless update flow and the ADR-0015 D1 recovery posture.

## Decision

The entire post-ESP OS disk is sealed in a single **LUKS2 container** — A/B
squashfs slots, the spec store + ZFS keyfiles, and `config/var` — with the ESP
as the only plain partition. The container is unlocked **only at boot**, by the
initramfs, using external factors; there is no post-boot unlock path.

- **Boot chain (A/B-at-ESP + unlock-then-mount)**: kernel + initramfs live per
  slot on the plain ESP (ADR-0011, ADR-0001). The bootloader selects the active
  slot's kernel+initramfs by bootenv — no unlock needed to *select* a slot, only
  to *mount* it. The initramfs unlocks the container, then mounts the active
  slot's rootfs. Boot-fail fallback to the other slot operates at the ESP level.
- **Unlock factors**: YubiKey FIDO2 (LUKS2 systemd token, enrolled with
  `--fido2-with-user-presence`), USB keyfile keyslot, and an offline recovery
  passphrase as a final keyslot. Multiple factors are enrollable concurrently.
- **Per-mode unlock policy**: `presence-only` (factor present, no touch/PIN —
  the ADR-0006 headless reboot keeps working) vs `touch` (interactive proof,
  e.g. YubiKey touch or PIN — update reboots need attendance). The mode is a
  property of how the FIDO2 credential is enrolled; the host-level `UnlockPolicy`
  resource (spec store, read post-boot) mirrors the enrolled slots and drives the
  Update controller's pre-reboot warning (ADR-0006).
- **Factor rotation**: on a booted host, `core` shells out to
  `cryptsetup luksAddKey/luksRemoveKey` and `systemd-cryptenroll enroll/wipe`
  (pinned versions, ADR-0001/ADR-0006 pattern) to add/remove/replace factors.
  ZFS keys are never touched — factor changes mutate only LUKS keyslots.
  Authorization requires an existing enrolled factor or the recovery passphrase.
- **Observability**: while locked there is no OS (the root is inside the
  container), so the locked state surfaces only at the unlock prompt. Once
  booted, `core` records the unlock event (factor kind, success/failure) and
  exposes a host lock-status gauge via resource status and Prometheus metrics
  (ADR-0008).
- **Backup coupling (ADR-0030)**: the restic repository password lives on the
  spec-store partition inside the container and is auto-loaded only after
  unlock; its D1 recovery boundary is unchanged.

## Alternatives considered

- **LUKS per partition**: more unlock events and keyslot bookkeeping per
  partition; rejected — one container = one unlock event = one keyslot set.
- **Secret-partitions-only LUKS with plain slots**: encrypting only the secret
  partitions fails the uniform "host is sealed" story; rejected even though
  encrypting the public squashfs adds little confidentiality on its own — the
  decision trades that for a single, uniform sealing model.
- **Network/remote unlock** (dropbear-in-initramfs style): rejected — factors
  are physical; unlock is physical/console only.
- **TPM/KMS sealing**: rejected in ADR-0015; not revisited (non-goal).
- **In-place re-encryption of existing installs**: encryption is an
  install-time decision (ADR-0007 pattern); there are no legacy installs in v1.
- **A dedicated keyslot-manager daemon**: more moving parts than shell-out;
  rejected in line with the restic/zfs shell-out pattern.

## Consequences

- Whole-host theft no longer yields keys + data: the OS disk is sealed and the
  data pools stay ZFS-native-encrypted (ADR-0015), whose keys are inside the
  sealed container.
- Headless updates (ADR-0006) work in `presence-only` mode with the factor
  inserted; `touch` mode or a missing factor blocks at the unlock prompt until
  attendance (Update controller warns first).
- A locked host cannot boot the OS at all — there is no locked-but-booted state
  and no network unlock. Observability while locked is console-level (unlock
  prompt) until boot completes.
- The OS disk remains a single point of failure (ADR-0011): OS-disk loss loses
  keys, spec store, repository password, and data (D1, ADR-0015/ADR-0030
  unchanged). Factor loss is recoverable via the recovery passphrase.
- The installer creates the container and enrolls initial factors (USB keyfile +
  YubiKey FIDO2 + recovery passphrase) at first boot; the OS-disk sizing check
  (ADR-0011) accounts for the container overhead and per-slot kernels/initramfs.
- The A/B tooling candidate must support LUKS-encrypted slots with initramfs
  unlock (ADR-0001 constraint).

## Open questions

- Exact A/B tool selection (rauc/ostree/ABRoot) — deferred to the `os-image`
  design; this ADR pins the LUKS-with-initramfs-unlock requirement.
- Whether `touch` mode should also require a PIN (FIDO2 user-verification) in
  addition to presence — a per-host enrollment option, no architecture impact.