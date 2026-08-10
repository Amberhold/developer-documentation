# ADR-0007: Configurable storage layout assigned at install time

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.7, decision 7

## Context

The system has several writable-state roles: system/config (spec store), app
images and compose state, and user data. Where those live on the pool layout is a
hardware-dependent choice (pool count, mirror/raidz, disk sizes) that cannot be
fixed in the image.

## Decision

The roles of datasets (system/config, app/images, data) are assigned at install
time by the installer, not fixed in the image. Everything else — controllers,
spec store, compose, shares — points at the assigned datasets.

## Alternatives considered

- **Fixed layout**: simpler, but cannot adapt to the admin's pool topology.
- **Post-install reconfiguration**: complex to migrate once data exists.

## Consequences

- The installer performs a first-boot assignment step (feature 12, `installer`).
- Pool creation and vdev layout are decided at install (feature 11,
  `pool-storage`); the layout is recorded in the spec store.
- Later changes to dataset roles are migration work, not routine config.
