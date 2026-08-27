# ADR-0015: Native encryption keys — keyfile on the OS-disk partition, headless unlock

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — keyfiles now inside the OS-disk LUKS container (host-encryption change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 2; ADR-0001, ADR-0006,
  ADR-0007, ADR-0011, ADR-0031

## Context

Feature 2 lists native ZFS encryption as minimal scope, but nothing decided where
keys live or how encrypted datasets unlock. Because updates reboot headless
(ADR-0006) and the root is read-only (ADR-0001), any unlock that needs human input
breaks the update flow. The threat model is at-rest protection on a management
LAN (feature 10); network hardening is deferred (scope-missing-plane D6).

## Decision

Encrypted datasets use a keyfile stored on the OS-disk spec store partition (next
to the spec store, ADR-0013), auto-loaded by `core` at start before any pool import.
There is no passphrase unlock at boot; protection is at-rest only (keys are
co-located with the system that holds the data).

With host encryption (ADR-0031), the OS-disk spec store partition — and with it
the keyfiles — lives inside the OS-disk LUKS2 container. Keyfiles are therefore
no longer plaintext on an unencrypted partition; they are protected at rest by
the LUKS container and only readable once the container is unlocked. Headless
unlock (ADR-0006) is preserved through `presence-only` external factors (e.g. a
YubiKey FIDO2 credential or USB keyfile that is present at boot, requiring no
touch or PIN). An offline recovery passphrase is additionally enrolled as a
final LUKS keyslot for factor-loss recovery. Because keys live on the OS disk
rather than on a pool, boot never depends on pool import to read them — the
unlock opens the container, then the keyfiles are read from inside it. Pool and
app datasets are ZFS-native-encrypted with these keys (resolve-control-plane-gaps
D6); the threat model is protection against removal/theft of the data disks
alone.

## Alternatives considered

- **Passphrase unlock via UI/API**: more separation between key and data, but
  breaks headless reboot during updates and needs a boot-complete-but-pools-locked
  state.
- **External key management (TPM/KMS)**: better separation, but adds hardware
  dependence and a key-service dependency out of proportion to v1 scope.

## Consequences

- Reboot and update flows (ADR-0006) work unattended while a `presence-only`
  factor is present; a `touch`-mode factor or a missing factor blocks at the
  unlock prompt until attendance (per-mode unlock policy, ADR-0031).
- Losing the OS disk means losing the keys (and the spec store, ADR-0013) and with
  them access to encrypted data. This is the authoritative recovery story: the
  reinstall + `zpool import` path (ADR-0011, ADR-0013) recovers unencrypted data
  only; encrypted datasets are unrecoverable after OS-disk loss. The OS disk is a
  single point of failure (ADR-0011); its redundancy is out of v1 scope. This D1
  consequence is retained unchanged: the keys are now *more* exposure-resistant
  (inside the LUKS container, ADR-0031) but still co-located on the lost disk, so
  OS-disk loss still loses them. Factor loss (not OS-disk loss) is recoverable via
  the recovery passphrase.
- Encryption protects against removal/theft of the data disks and (with the
  host-encryption container, ADR-0031) whole-host theft, not against
  compromise of the running system.
- Forwarded logs/audit and generated daemon config fragments are encrypted at
  rest once the OS-disk container is unlocked (they live on `config/var` inside
  the LUKS container, ADR-0011/ADR-0031) and remain protected by rotation and
  regeneration (ADR-0016), not by snapshots (ADR-0011).
