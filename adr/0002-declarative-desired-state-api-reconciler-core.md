# ADR-0002: Declarative desired-state API with reconciler core

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, decision 2
- Amendments: rollback is composite — spec-store revert plus ZFS snapshot
  rollback (design decision D2).

## Context

The management surface (web UI, CLI) must stay a thin client over a stable API.
Host subsystems (ZFS, samba, containerd, NVMe-oF) are stateful and can drift from
admin intent (e.g. hand-edited config files).

## Decision

The API is declarative desired-state. Clients write the desired configuration;
a `core` daemon reconciles actual host state toward it. The spec store is the
source of truth. UI and CLI are thin clients that only issue desired-state
requests. RBAC is enforced at API admission, not in the UI.

## Alternatives considered

- **Imperative API**: simpler to implement initially, but drift and rollback are
  hard and admin actions are not auditable as a whole.
- **Hybrid**: imperative plus a state layer; inherits most of the imperative
  downsides without a clean model.

## Consequences

- The `core` daemon is a reconciler with a spec store (persisted on a configured
  dataset).
- Host state is never mutated outside the daemon.
- Drift detection falls out of the model: status surfaces actual-vs-desired.
- Rollback is composite: reverting desired state in the spec store (which the
  reconciler then enforces) *plus* restoring data from ZFS snapshots. Spec revert
  alone fixes forward drift but cannot undo data written by apps/users; the spec
  store therefore needs versioning/snapshot support for the revert half.
- Every controller (ZFS, SMB, NFS, NVMe-oF, Apps, Identity, Update) reconciles
  its slice of the spec store.
