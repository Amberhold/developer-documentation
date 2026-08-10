# ADR-0022: Shared `Schedule` resource for snapshots and scrubs

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` features 2, 14; ADR-0008,
  ADR-0017, ADR-0019, ADR-0021; `resolve-control-plane-gaps` D5, D13
- Amendments: the `config` dataset no longer exists (ADR-0007) — system config
  state lives on OS-disk partitions without fs snapshots (ADR-0011), so the D13
  clause snapshotting the `config` dataset is dropped. `Schedule` covers data/app
  dataset snapshots and scrubs only.

## Context

Feature 2 gains scheduled snapshots with retention (D5), and feature 14 needs a
scrub cadence (ADR-0021). Both are recurring, time-based work driven by the
reconciler. Without a shared model, each feature builds its own scheduler and
duplicates cadence, retention, and cron-like expression handling.

## Decision

A single cron-like `Schedule` resource in the API (ADR-0019) is consumed by the
snapshot controller (feature 2) and the disk-health/scrub controller (ADR-0021).
Schedules are declarative spec resources like any other; controllers observe the
schedules that reference them and reconcile accordingly.

## Alternatives considered

- **Per-feature schedulers**: two engines, duplicated cadence/retention logic.
- **systemd timers**: re-arises the read-only-root problem — timers live under
  `/etc`, which ADR-0001 makes immutable.

## Consequences

- `Schedule` joins the API resource model (ADR-0019) and the `contracts` surface.
- The `storage` change decides schedule expression syntax and defaults; the
  resource shape is fixed here.
