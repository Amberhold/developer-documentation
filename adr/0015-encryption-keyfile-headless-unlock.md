# ADR-0015: Native encryption keys — keyfile on the OS-disk partition, headless unlock

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 2; ADR-0001, ADR-0006,
  ADR-0007

## Context

Feature 2 lists native ZFS encryption as minimal scope, but nothing decided where
keys live or how encrypted datasets unlock. Because updates reboot headless
(ADR-0006) and the root is read-only (ADR-0001), any unlock that needs human input
breaks the update flow. The threat model is at-rest protection on a management
LAN (feature 10); network hardening is deferred (feature-map D6).

## Decision

Encrypted datasets use a keyfile stored on the OS-disk config partition (next to
the spec store, ADR-0013), auto-loaded by `core` at start before any pool import.
There is no passphrase unlock at boot; protection is at-rest only (keys are
co-located with the system that holds the data).

The `config` dataset (and any encrypted pool) is itself encrypted; because keys
live on the OS disk rather than on a pool, boot never depends on pool import to
read them, and the `config` dataset no longer holds plaintext keys
(resolve-control-plane-gaps D6).

## Alternatives considered

- **Passphrase unlock via UI/API**: more separation between key and data, but
  breaks headless reboot during updates and needs a boot-complete-but-pools-locked
  state.
- **External key management (TPM/KMS)**: better separation, but adds hardware
  dependence and a key-service dependency out of proportion to v1 scope.

## Consequences

- Reboot and update flows (ADR-0006) work unattended.
- Losing the OS disk means losing the keys (and the spec store, ADR-0013) and with
  them access to encrypted data; the `config` dataset no longer holds plaintext
  keys and can itself be encrypted. The OS disk is a single point of failure
  (ADR-0011); its redundancy is out of v1 scope.
- The `config` dataset is snapshotted on the shared `Schedule` resource
  (ADR-0022), protecting keys and forwarded logs/audit (ADR-0016) against
  accidental data-path loss (resolve-control-plane-gaps D13).
- Encryption protects against disk removal/theft, not against compromise of the
  running system.
