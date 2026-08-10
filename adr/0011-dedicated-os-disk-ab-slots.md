# ADR-0011: Dedicated OS disk for the A/B squashfs slots

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, §3.7; ADR-0001, ADR-0007

## Context

ADR-0001 fixes the root as a read-only squashfs image with A/B dual-slot boot,
and ADR-0007 makes the roles of *data* datasets configurable at install. Neither
constrains where the two OS slots and bootloader physically live. The boot media
is the one part of the topology that cannot be a configurable dataset, and it is
a prerequisite for the `os-image`, `installer`, and `system-update` designs.

## Decision

The A/B squashfs slots and the bootloader live on a dedicated small OS disk
(e.g. a 64-128 GB SSD/NVMe). ZFS pools are pure data; ZFS is never the boot media.
The OS disk is fixed (one disk, two slots + bootloader); everything else — system
config, app images, user data — is configurable per ADR-0007.

## Alternatives considered

- **Partitions on a pool disk**: saves a disk, but couples boot to pool
  membership and reintroduces the boot-depends-on-pool-state coupling ADR-0001
  exists to avoid.
- **EFI/ESP-based slots**: no separate disk needed, but the ESP is small and
  sizing two squashfs images on it is atypical.

## Consequences

- The installer and hardware planning must account for a dedicated OS disk
  (feature 12, `installer`).
- The bootloader selects the active slot from the OS disk; rollback and boot-fail
  fallback (ADR-0001) operate entirely on this disk and never touch pools.
- Data pools can be created, imported, or rebuilt independently of the OS disk —
  including the reinstall recovery path (feature 12 `zpool import`).
