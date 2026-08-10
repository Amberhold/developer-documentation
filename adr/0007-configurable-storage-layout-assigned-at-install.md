# ADR-0007: Configurable storage layout assigned at install time

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.7, decision 7
- Amendments: the writable-state role is renamed `config` (the spec store is *not*
  part of it — the spec store lives on the OS disk, ADR-0013). It holds forwarded
  logs/audit (ADR-0016) and generated daemon config fragments (ADR-0001); it no
  longer holds encryption keyfiles, which live on the OS disk (ADR-0015,
  resolve-control-plane-gaps D6).

## Context

The system has several writable-state roles: config, app images and compose
state, and user data. Where those live on the pool layout is a hardware-dependent
choice (pool count, mirror/raidz, disk sizes) that cannot be fixed in the image.

## Decision

The roles of datasets (`config`, `app/images`, `data`) are assigned at install
time by the installer, not fixed in the image. Everything else — controllers,
compose, shares — points at the assigned datasets. The spec store is excluded by
design (ADR-0013).

## Alternatives considered

- **Fixed layout**: simpler, but cannot adapt to the admin's pool topology.
- **Post-install reconfiguration**: complex to migrate once data exists.

## Consequences

- The installer performs a first-boot assignment step (feature 12, `installer`).
- Pool creation and vdev layout are decided at install (feature 11,
  `pool-storage`); the layout is recorded in the spec store.
- Later changes to dataset roles are migration work, not routine config.
