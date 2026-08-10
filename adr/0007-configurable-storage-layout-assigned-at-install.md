# ADR-0007: Configurable storage layout assigned at install time

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.7, decision 7
- Amendments: the `config` role is removed. System config state does not live on
  a pool at all — it lives on OS-disk partitions (spec store + keyfiles, and
  `config/var` for logs/audit and generated daemon config fragments), ADR-0011.
  The remaining roles are `app/images` (opt-in) and `data`; encryption keyfiles
  live on the OS disk (ADR-0015, resolve-control-plane-gaps D6).

## Context

The system has writable-state roles: app images and compose state, and user data.
Where those live on the pool layout is a hardware-dependent choice (pool count,
mirror/raidz, disk sizes) that cannot be fixed in the image. System config state
is *not* a pool role: it lives on the OS disk as plain partitions (ADR-0011), so
pools are pure user data and boot never depends on pool import.

## Decision

The roles of datasets (`app/images` when the apps feature is in use, and `data`)
are assigned at install time by the installer, not fixed in the image. Everything
else — controllers, compose, shares — points at the assigned datasets. System
config state is excluded by design: the spec store lives on an OS-disk partition
(ADR-0013) and the `config/var` partition holds logs/audit and generated daemon
config fragments (ADR-0011, ADR-0016).

## Alternatives considered

- **Fixed layout**: simpler, but cannot adapt to the admin's pool topology.
- **Post-install reconfiguration**: complex to migrate once data exists.
- **System config on a `config` pool dataset**: couples OS state to pool
  availability and re-couples boot to pools; rejected in favor of OS-disk
  partitions (ADR-0011).

## Consequences

- The installer performs a first-boot assignment step (feature 12, `installer`)
  for the data (and optional app) datasets.
- Pool creation and vdev layout are decided at install (feature 11,
  `pool-storage`); the layout is recorded in the spec store.
- Later changes to dataset roles are migration work, not routine config.
- Pools are data-only; the OS disk is never a pool member (ADR-0011).
