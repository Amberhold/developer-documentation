# ADR-0006: Updates as a product feature

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-09-01 — update flow keeps both OS-mirror ESPs in sync before
  activation (os-disk-redundancy change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.6, decision 6; ADR-0001

## Context

A/B dual-slot gives the mechanism to swap OS images atomically. But updates are a
user-facing workflow — trigger, progress, reboot, rollback — and must be managed
like any other capability, not left as raw image mechanics.

## Decision

System updates are a product feature exposed through the UI/API: trigger an
update, report progress, perform the reboot, and roll back on failure. The Update
controller is a normal core controller reconciling the update slice of the spec
store (ADR-0002). Because `core` is baked into the read-only image (ADR-0001),
core updates are OS updates and flow through this same path. Delivery is
repo-channel-only — images are pulled from a configured repo channel and there is
no manual image-upload path (resolve-architecture-gaps D8).

## Alternatives considered

- **Image mechanics only**: shipping images and leaving trigger/rollback to
  operators; fails the no-SSH, product-feature requirement.

## Consequences

- An update flow with a reboot in the middle: stage to slot B, set bootenv,
  reboot, boot-fail detection, manual and automatic rollback paths. When the OS
  disk is mirrored (ADR-0011), staging writes slot B's kernel+initramfs to
  **both** member ESPs and the update is activated only after both ESPs carry the
  same per-slot content (ADR-0001 dual-ESP constraint); boot-fail rollback falls
  to the other slot's kernel+initramfs on whichever ESP survives.
- Update status (progress, active slot, staged slot) is observable via API/UI and
  Prometheus metrics (ADR-0008).
- `infra` image builds and the update payload are a first-class part of the
  system, not an ops afterthought.
- Delivery is repo-only: the Update controller pulls images from a configured
  update channel (repo URL + channel). No manual upload path exists — an
  air-gapped/offline update flow is deliberately out of scope for v1, consistent
  with the mgmt-LAN networking posture (feature 10).
- Image integrity: update images are signed and the trust anchor is baked into
  the read-only image; the Update controller verifies the inactive slot's
  signature before it is activated (ADR-0001, ADR-0011).
