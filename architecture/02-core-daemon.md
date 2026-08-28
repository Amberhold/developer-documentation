# Core Daemon Architecture — the Framework-First Controller Runtime

> Discovery-phase design. Authored from the `core-daemon-design` openspec change.
> The decisions D1–D10 here are the framework contract every subsystem controller
> (storage, shares, apps, backup, disk-health, update) builds against. The ADRs
> fix the *decisions*; this document fixes the *internal mechanics* of how a
> controller actually works inside `core`.

## 1. Purpose

The ADRs fix the decisions (per-controller reconcile loops, ADR-0017; versioned
spec store, ADR-0013; OTEL with a `Telemetry` controller, ADR-0008) and the
contracts fix the external shape (resource envelope, status, metric catalog).
This document describes the internal architecture of the `core` daemon —
**framework-first**: a generic controller runtime/SDK that every subsystem
controller plugs into, so no controller hand-rolls loops, status, events, or
telemetry.

The design is implementable as the `core` skeleton (a later change) and stable
enough that storage, shares, apps, backup, disk-health, and update all build
against it without rework.

## 2. Goals / Non-Goals

**Goals:**
- A single controller runtime providing once and uniformly: the reconcile loop,
  event delivery, status persistence, metrics + tracing, backoff/resync, and the
  provider handle.
- A controller's job is reduced to "reconcile one owned resource kind against
  host state."
- The design surfaces the decisions that belong in ADRs (amendments to
  0017/0013/0008 and new ADR-0031).

**Non-Goals:**
- No per-subsystem controller design — those are their own changes against this
  framework.
- No HA/multi-instance — the daemon is single-instance on the appliance; the
  event bus is in-process.
- No installer/boot-image work — the startup sequence is designed here,
  implemented elsewhere.

## 3. Controller interface (D1) — one owned resource kind per controller

Each controller implements a typed `Controller` interface registered against
exactly one resource kind:

```
type Controller interface {
    Resource() string                       // kind, e.g. "Pool"
    Reconcile(ctx, ReconcileRequest) Result // converge actual toward wanted
    Actions() ActionMap                     // imperative ops, keyed by name (see D7)
}

type Action func(ctx context.Context, ActionRequest) Result // one imperative op
type ActionMap map[string]Action                             // action name → handler
```

**Rationale:** `core` runs one loop per controller (ADR-0017); one controller →
one kind keeps ownership unambiguous (who writes status), matches the contract
(each resource has one `status`), and lets the runtime deliver events, run the
loop, and emit telemetry uniformly.

Rejected alternative: a controller owning several kinds (e.g. a "storage"
controller handling pools+disks+datasets). That violates the per-controller
isolation and status-ownership clarity in ADR-0017; the granularity is
sub-controllers per kind.

## 4. Event source (D2) — spec-store writes pushed to an in-process bus, with a resync safety net

The spec store publishes an event on every accepted spec write (upsert/delete)
into an in-process bus. A controller subscribes to events for its owned kind only
(ADR-0017: a controller observes only the resources it owns). Each loop also runs
a **periodic full resync** over its owned resources.

**Rationale:** push matches the contract's "one root `reconcile` span per pass
over the shared event source" and avoids N controllers polling and diffing the
whole store every cycle. The resync is the drift safety net — hand-edited host
files, missed events, and the ADR-0017 "fails and retries" recovery path all
converge on the next resync. Idempotency is guaranteed per D9.

Rejected alternatives: controllers polling the store and diffing every interval
(loses the event source's purpose, scales poorly, no event granularity for
tracing); a separate persistent event log/WAL replayed at boot (deferred —
events are derived from the spec store, which is already the source of truth,
and the resync covers anything replayed events would).

## 5. Loop cadence, backoff, and resync (D3)

Per controller: drain its event queue, then reconcile each affected resource; on
failure, requeue with exponential backoff (capped); a full resync of owned
resources runs on an interval independent of event flow. Controllers observe no
dependency ordering (ADR-0017): a resource whose prerequisite is not ready is
reported `Pending`/`Reconciling` with a reason and retried next loop — never a
hard failure that wedges the controller.

**Rationale:** fail-and-retry idempotent is the ADR-0017 contract; backoff
prevents a storm when a dependency (e.g. pool import, identity link) is absent;
resync catches drift by construction. Rejected alternative: fixed-interval-only
reconciling (no event responsiveness; status latency bounded by the interval).

## 6. Status ownership (D4) — the owning controller is the single writer, the API server is a pure reader

The controller computes and persists the `status` object for each resource it
owns, through the spec store's serialized write path (D5) — **not** through API
admission. It sets `phase` (Pending/Reconciling/Ready/Degraded/Error/Unknown),
`reason`, `conditions`, `wanted` = the spec being converged to, `actual` = last
observed host state, and `observedGeneration` = the `metadata.generation` the
status reflects. The API server reads status from the store.

**Rationale:** ADR-0017 says "`core` aggregates status from all controllers for
the API" — aggregation is storage plus reads, not a runtime merge step.
Persisted status survives restarts; a controller crash leaves the last truthful
status (with `observedGeneration` < `generation` so staleness is visible), not a
lost in-memory report. `observedGeneration` makes the wanted-vs-actual contract
self-describing.

Rejected alternative: controllers report status in-memory to an aggregator
(status vanishes on restart; the API would recompute it; the contract treats
status as a resource property).

## 7. Spec-store write path (D5) — one serialized writer, versioned snapshots

The spec store is the source of truth (ADR-0002) on the OS-disk partition
(ADR-0013). Two write flows, both serialized through the store's single writer:

1. **Spec writes**: API admission → RBAC → optimistic concurrency check on
   `resourceVersion` → commit as an atomic, fsync'd bbolt transaction (bump
   `metadata.generation`, new `resourceVersion`) → publish event → audit.
2. **Status writes**: the owning controller (D4), serialized, coalesced on a short
   minimum interval to avoid write amplification on busy resources; they commit
   through the same bbolt transactions and never bump generation or
   `resourceVersion`.

The concrete on-disk format is an **embedded bbolt KV store** (`store.db`, one
bucket per resource kind, key = resource id, value = JSON envelope). The
`resourceVersion` ordinal is a `/_meta` revision counter incremented in the
same transaction as the write, so optimistic concurrency and the commit are
atomically one operation. Reads are snapshot-consistent bbolt read
transactions: MVCC means they never block the single writer, so the skeleton's
in-memory read index was dropped. The store records a schema version in
`/_meta` and migrates forward on open; rollback restores a prior versioned
snapshot (ADR-0013) — the composite-rollback revert half (ADR-0002).

**Versioned snapshots** are JSON exports (`snapshot-<schemaVersion>.<n>.json`,
N generations retained) taken from a single read transaction on the resync
cadence; they are the rollback export artifact, not the live engine. Restore
loads a chosen snapshot into the store and requires a `core` restart (startup
sequence D6 reloads the spec), so rollback is an explicit, audited operation.

**Rationale:** a single writer avoids torn spec/status state and makes optimistic
concurrency (the contract's `resourceVersion`) coherent. bbolt is a plain
embedded KV — the store's `Get/List/Create/Update/Delete` shape maps 1:1 — with
crash-durable, fsync'd, multi-key-atomic transactions and MVCC reads. Versioned
JSON snapshots are the ADR-0013 rollback mechanism. Rejected alternatives: etcd
(1-node Raft for a single-instance appliance, no HA benefit), SQLite (query
machinery the store does not need), and hardened JSON (whole-file rewrites give
no multi-key atomicity).

**Note:** the on-disk concrete format open question (versioned JSON snapshot
directories vs an embedded KV) is settled at implementation as the embedded KV
branch, with JSON retained as the versioned-snapshot export format (see the
`spec-store-bbolt-backend` change and the skeleton notes in §15).

## 8. Startup sequence (D6)

After OS-disk unlock (host-encryption, outside this change), in order:

1. mount `config/var`; load spec store, migrating schema forward (ADR-0013)
2. build default OTEL providers (sampling zero, no OTLP targets — contract
   default) and the provider handle (D8)
3. start the event bus; register all controllers (each resolving its subsystem
   logger/meter through the provider handle — never holding providers itself, D8)
4. start controllers — reconciling immediately, **with pools absent** (ADR-0013:
   reconcile scope degrades, startup never blocks on pool import)
5. storage controller imports pools as they become available; controllers
   dependent on pools (shares, apps, backup) converge after via D3 retry
6. start the API server (HTTPS, ADR-0028) — last, so the first admin request
   sees a converged pass

**Rationale:** spec-before-pool is fixed by ADR-0013; API-last makes the booted
daemon present already-reconciled status rather than a cold store. Rejected
alternative: API first, controllers later (first requests would observe
`Pending`/empty status and audit before controllers had a pass).

## 9. Action endpoints (D7) — routed to the owning controller, never a spec mutation

The runtime routes `/v1/<kind>/{id}/actions/<name>` (e.g. `scrub`, `replace`,
`trigger`, `rollback`) to the handler it looks up from the owning controller's
`Actions()` map (D1). Singleton resources (e.g. `updates`) have no `{id}`
segment — the path is `/v1/<kind>/actions/<name>`. Actions may read spec, may
mutate **status** (result observable, e.g. scrub status), and never touch
desired state (ADR-0002). They are audited exactly like spec writes.

**Rationale:** ADR-0002/contract fix actions as imperative ops outside desired
state; routing to the owning controller keeps one owner for the resource's
lifecycle and keeps actions out of the reconciler's diff. Rejected alternative:
actions encoded as synthetic spec writes (pollutes desired state and breaks the
"action does not change spec" scenario in the `api-contract` spec).

## 10. Telemetry provider handoff (D8) — one handle, resolved indirection

The `Telemetry` controller owns the MeterProvider/TracerProvider/LoggerProvider
and reconverges them on spec change without restart (ADR-0008). Controllers
receive a **provider handle** at registration and resolve the current provider
per use (or per loop) — they never build or hold providers themselves.

**Rationale:** the contract scenario "controllers reference the running
providers and never build their own" and "reconfigure without restart" force
indirection: a handle whose resolution tracks the telemetry controller's
reconvergence. Without it, a controller holding a stale provider would keep a
dead pipeline after a reconfigure. Rejected alternative: pass immutable
providers at startup and rebuild controllers on reconfigure (requires a
restart-equivalent orchestration the contract explicitly rules out).

## 11. Idempotency and drift (D9)

Each `Reconcile` diffs the spec against observed host state; every mutation is a
no-op when already converged; shell-outs (zfs, smartctl, nerdctl, restic, samba)
run through a single wrapper with predictable error mapping. This is what makes
the ADR-0017 "no dependency ordering, fail and retry" model safe, and what makes
the composite rollback (spec revert via D5 snapshot restore + a ZFS-snapshot
rollback action, ADR-0002) convergent.

## 12. Metrics and tracing are runtime-owned (D10)

The runtime emits `amberhold.controller.reconcile.{loops,errors,duration}` per
subsystem from the catalog and creates the root `reconcile` span per pass + child
span per controller step carrying resource id, wanted `generation`, and actual
version (contract convention). Controllers never hand-roll telemetry; they only
set attributes that flow through.

**Rationale:** the catalog and span convention are contract-enforced (OTEL Views
reject unregistered metrics); centralizing in the runtime makes every controller
conform by construction.

## 13. Risk / trade-offs

- **Framework-first over-abstraction** → the SDK is exercised immediately: the
  storage controller (the data-plane anchor) is the next change built on this
  framework; the SDK grows only from real needs, and the interface stays minimal
  (D1: one method, one kind).
- **Event loss (in-process bus, single instance)** → resync (D3) and idempotency
  (D9) make events an optimization, not a correctness requirement.
- **Persisted status write amplification** → status writes are coalesced (D5);
  `observedGeneration` exposes staleness rather than masking it.
- **Single serialized writer as a bottleneck** → spec writes are low-rate admin
  ops; status writes are coalesced; MVCC bbolt read transactions keep API reads
  off the write path.
- **Shell-out failure mid-mutation** → idempotent reconcile converges on retry;
  audit + status `Error`/`Degraded` with reason make it observable.
- **No HA** → deliberate v1 constraint; the in-process event source assumes a
  single daemon.

## 14. Open questions (implementation-time)

- **Resync interval, backoff caps, and status-coalescing window defaults** —
  tunable during implementation; they do not change the approach.
- **Spec-store on-disk concrete format** (versioned JSON snapshot dirs vs
  embedded KV) — settled at implementation as the embedded KV branch (bbolt,
  `spec-store-bbolt-backend` change); JSON is retained as the versioned-snapshot
  export/rollback format.
- **Go package layout** inside `core/` (e.g.
  `internal/{store,eventbus,controller,runtime,api,admission,otel}`) — settled
  when the skeleton change is written, derived from this design.

## 15. Skeleton implementation notes (2026-08-27)

The first `core` implementation (`core-skeleton-auth-slice`) confirmed D1–D10
and recorded its package layout and any deviation:

- **Package layout**: `cmd/core` (entrypoint) plus
  `internal/{app,resource,store,eventbus,controller,runtime,telemetry,
  admission,auth,api,audit}` — the layout open question is settled as derived
  from the design. `app` is the composition root implementing D6.
- **D1 (one kind per controller)**: the `auth` subsystem registers four
  controllers (`User`, `Role`, `Session`, `Token`), one per owned kind, sharing
  a backing service — no controller owns multiple kinds.
- **D5 event publication**: the spec store publishes events through a
  `Publisher` interface wired to the in-process bus at startup; publication
  happens after the write transaction is committed and is best-effort (a
  saturated subscriber drops the event, covered by the D3 resync). Reads return
  fresh copies from bbolt read transactions, so no goroutine aliases a stored
  resource.
- **D3 serialization**: each controller drains a single queue; events, the
  periodic resync marker, and backoff retries all funnel through that one
  consumer, so a controller's reconciles are strictly serialized (no concurrent
  passes over the same resource).
- **D4 status preservation**: spec writes preserve the persisted status
  (clients never write status); only the owning controller mutates it through
  the store's serialized path.
- **Telemetry**: the skeleton ships a catalog-enforcing registry (the
  `amberhold.*` names from `contracts/metrics/catalog.yaml` declared at
  startup) rendering the Prometheus exposition format on `/metrics`, a
  subsystem-scoped `slog` logger, and a no-op tracer (sampling zero, no OTLP
  targets — the ADR-0008 default). The D8 handle resolution is in place; the
  OTLP exporters land with the telemetry controller change.
- **On-disk store format** (updated by `spec-store-bbolt-backend`): the live
  engine is an embedded bbolt KV store (`store.db`) with the schema version and
  the `resourceVersion` ordinal counter in a `/_meta` bucket; forward migration
  on open and rejection of newer schemas are implemented (ADR-0013). The
  skeleton's `snapshot.json` engine was never built or shipped, so no legacy
  data or import path exists. Versioned JSON snapshots
  (`snapshot-<schemaVersion>.<n>.json`, N=5 retained, exported from read
  transactions on the resync cadence) are the rollback export artifact, and
  restore requires a `core` restart (D6).
