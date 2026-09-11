# ADR-0032: rauc as the A/B tooling

- Status: accepted
- Date: 2026-09-08
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/13-os-image-installer.md` (D1–D6, D21, D22);
  `docs/architecture/01-os-feature-map.md` feature 1, feature 12, §6;
  ADR-0001, ADR-0006, ADR-0011; `os-image-installer` change

## Context

ADR-0001 fixes the root as a read-only squashfs image in A/B dual slots and
ADR-0006 makes updates a product feature (trigger, progress, reboot, rollback,
repo-channel delivery, signed images). ADR-0011 pins the OS-disk layout — a
single LUKS2 container (optionally on an `md0` mirror) with the ESP as the only
plain partition, A/B-at-ESP slot selection, and initramfs unlock-then-mount —
and explicitly defers the A/B tool selection (rauc/ostree/ABRoot candidates).
Nothing produces a bootable system yet: `infra` is empty and the fresh-install
and update paths both need one honest slot-writing mechanism. This ADR records
the deferred choice.

## Decision

**rauc** is the A/B tooling, shared by the fresh install and the update path
(`os-image-installer` D1). Rationale against the ADR-0001/0011 constraints:

- **Explicit slot model** matches the two-squashfs-slots layout on the OS disk:
  slots are named (`rootfs.0`/`rootfs.1`) with per-slot mount handling and a
  `booted` marker, so activation and rollback are declarative slot operations
  rather than bespoke bootenv scripting.
- **Signed `.raucb` bundles** satisfy ADR-0006's signed-image requirement: CMS
  signatures verified against a baked-in trust anchor, with the same anchor
  carried by the installer ISO so fresh install verifies identically to an
  update (`os-image-installer` D21).
- **EFI/systemd-boot backend** implements A/B-at-ESP slot selection: per-slot
  kernel+initramfs on the plain ESP and a boot-counted bootenv, so selection and
  boot-fail fallback need no container unlock (ADR-0011). The initramfs-tools
  hook (D3) reports rauc boot-status good/bad post-mount so a pre-unlock boot
  failure and a post-unlock mount failure both surface to rauc.
- **LUKS-encrypted slots are supported** through the `rauc-writes-to-LUKS`
  convention: rauc `system.conf` targets `/dev/mapper` slot devices (the
  container is always unlocked on a booted host), never raw partitions.

Two custom pieces are owned by the design, not by rauc:

1. **Dual-ESP sync** — rauc has one ESP concept; on a mirrored install
   (ADR-0011) the ESP content is duplicated per member, so a post-install hook
   copies the freshly written slot's kernel+initramfs to the mirror member's ESP
   and activation proceeds only when both ESPs carry identical per-slot content.
2. **Boot-fail reporting boundary** — selection is pre-unlock but rootfs mount
   success is post-unlock, so an initramfs hook bridges rauc's boot-status into
   the bootloader's boot count; a mark-good service records a successful boot
   (`os-image-installer` D22).

## Alternatives considered

- **ABRoot** — general-distro dual-root transactionality, but single-ESP-native
  and without rauc's slot/activation vocabulary or built-in signature
  verification; would need more custom glue for the dual-ESP + LUKS layout.
- **ostree** — deployment-graph model fights the two-slot A/B-at-ESP layout
  ADR-0011 pins; more machinery than two squashfs slots need.

## Consequences

- rauc is a first-class baked-in image component (pinned, ADR-0001 pattern); the
  fresh-install `rauc install` and the update-path `rauc install` share one
  verified slot-writing code path (`os-image-installer` D6, D21).
- The Update controller shells out to a pinned rauc binary through a thin `Rauc`
  host facade, mirroring the smartctl/nerdctl/restic external-tool pattern
  (`os-image-installer` D10).
- Automatic boot-fail rollback (boot-counted fallback to the prior slot) and the
  manual `rollback` action together fulfill ADR-0006's manual-and-automatic
  rollback promise (D22).
- The A/B update flow (ADR-0006 §consequences) is implemented on rauc's
  activate/mark-good/boot-count semantics; boot-fail status feeds the `updates`
  resource status (last result incl. automatic fallback).
- rauc is the only A/B tooling; there is no migration path from it
  (`os-image-installer` non-goal).
