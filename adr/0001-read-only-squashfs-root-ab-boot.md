# ADR-0001: Read-only squashfs root with A/B boot

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, decision 1; ADR-0006,
  ADR-0009
- Amendments: `core` baked into the RO image; no subsystem writes host files
  under `/` (including `/etc/exports`).

## Context

The OS image must be updatable and must not drift from a known-good state. Two
candidate root strategies exist: keep the root on ZFS (rich snapshot/rollback of
OS state) or keep it on a plain read-only partition.

## Decision

The root filesystem is a read-only squashfs image; ZFS is reserved exclusively
for data pools. Updates are delivered as new images in an A/B dual-slot layout;
the bootloader selects the active slot and a rollback falls back to the other
slot.

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
- All writable state (configs, spec store, container images) lives on configured
  datasets, never on `/`.
- No subsystem writes host files under `/` — including `/etc/exports`, which is
  not used (NFS shares are ZFS `sharenfs` properties, ADR-0009).
- A/B = two slots; rollback = fall to the other slot; boot failure falls back
  automatically.
