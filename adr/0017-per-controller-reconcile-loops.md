# ADR-0017: Per-controller reconcile loops over a shared event source

- Status: accepted
- Date: 2026-08-10
- Amended: 2026-08-27 — pinned the controller/event-source mechanics: one
  owned resource kind per controller, push events over spec-store writes with a
  periodic full resync, exponential backoff + resync cadence, and the owning
  controller as the single persisted status writer (`core-daemon-design`)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, §4, §6;
  `docs/architecture/02-core-daemon.md`; ADR-0002, ADR-0013

## Context

The feature map (§6) left reconcile-loop granularity open: a single global pass
over all wanted vs actual state, or one loop per controller. The skeleton already
lists a controller per subsystem (ZFS, SMB, NFS, NVMe-oF, Apps, Identity, Update).

## Decision

Each controller runs its own reconcile loop over its slice of the spec store,
sharing a single event source (spec-store writes). A controller observes only the
resources it owns; failures are contained to that controller. There is no declared
dependency ordering: when a controller needs a prerequisite owned elsewhere (e.g.
an identity link before a share bind, or a network address before a share binds),
it fails and retries on its next loop; reconciliation is idempotent
(resolve-control-plane-gaps D12).

The `core-daemon-design` change pins the mechanics (`02-core-daemon.md`):

- **One owned resource kind per controller** (D1). Each controller implements a
  typed `Controller` interface (`Resource()`, `Reconcile(ctx, ReconcileRequest)
  Result`, `Actions()`) registered against exactly one kind, keeping ownership —
  and status-writing — unambiguous.
- **Push-based event source** (D2). The spec store publishes an event on every
  accepted spec write (upsert/delete) into an in-process bus; a controller
  subscribes only to its owned kind. Each loop also runs a periodic full resync
  over its owned resources as the drift safety net.
- **Backoff and resync cadence** (D3). On failure a controller requeues with
  exponential backoff (capped); a full resync of owned resources runs on an
  interval independent of event flow. Resources whose prerequisites are not ready
  are reported `Pending`/`Reconciling` with a reason and retried — never a hard
  failure.
- **Single persisted status writer** (D4). The owning controller computes and
  persists the `status` object (phase, reason, conditions, wanted, actual,
  `observedGeneration`) through the spec store's serialized write path. The API
  server is a pure reader; "aggregation" is storage plus reads, not a runtime
  merge step.

## Alternatives considered

- **Single global pass**: simpler sequencing, but any slow or failing controller
  blocks all others and couples subsystem cadences.

## Consequences

- Controllers are independent units, matching the skeleton's controller list.
- Spec-store writes fan out to interested controllers; `core` aggregates status
  from all controllers for the API (persisted status reads, D4).
- Each controller registers its own Prometheus collectors (ADR-0008); the
  reconcile-loop, backoff/resync, and event-source mechanics are provided by the
  framework runtime (`02-core-daemon.md` D1–D4), so controllers never hand-roll
  loops, events, or status persistence.
