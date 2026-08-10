# ADR-0006: Updates as a product feature

- Status: accepted
- Date: 2026-08-10
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
store (ADR-0002). Because `core` is baked into the read-only image (ADR-0001
amendment), core updates are OS updates and flow through this same path.

## Alternatives considered

- **Image mechanics only**: shipping images and leaving trigger/rollback to
  operators; fails the no-SSH, product-feature requirement.

## Consequences

- An update flow with a reboot in the middle: stage to slot B, set bootenv,
  reboot, boot-fail detection, manual and automatic rollback paths.
- Update status (progress, active slot, staged slot) is observable via API/UI and
  Prometheus metrics (ADR-0008).
- `infra` image builds and the update payload are a first-class part of the
  system, not an ops afterthought.
