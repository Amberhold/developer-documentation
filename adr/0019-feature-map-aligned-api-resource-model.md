# ADR-0019: Feature-map-aligned API resource model

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 6, §6; ADR-0002,
  ADR-0008, ADR-0018

## Context

Feature 6 (API + CLI) commits to a first-class HTTP API but the resource model was
open (§6). The API is declarative desired-state (ADR-0002) and RBAC runs at
admission (ADR-0018); the UI and CLI are thin clients. The shape must also carry
metrics names/labels as a contract (ADR-0008).

## Decision

One capability-first OpenAPI: resources mirror the feature map — pools, disks,
datasets, snapshots, shares (file + block exports), apps, users/roles, updates,
networking. Disks exist independent of pool membership (spares, unassigned, and
failed disks are not pool members), so disk inventory and per-disk health live on
their own resource (ADR-0021). Each resource has desired-state CRUD plus a status
object (wanted vs actual, per ADR-0002). A cron-like `Schedule` resource is part
of the model, consumed by the snapshot and scrub controllers (ADR-0022). The
contract lives in `contracts/`.

## Alternatives considered

- **Generic typed-object API (k8s-style)**: uniform, but a large abstraction to
  build before v1 has real users.
- **Grow contracts per subsystem independently**: less up-front work, but risks
  inconsistent conventions across subsystems.

## Consequences

- The OpenAPI contract is authored once and shared by `core`, UI, and CLI.
- Status objects surface drift per resource, feeding the metrics model (ADR-0008).
- Metrics names/labels are declared alongside the resource model.
- The `Schedule` resource carries snapshot/scrub cadence and retention; schedule
  semantics are fixed here and defaulted in the `storage` change (ADR-0022).
