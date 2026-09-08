# ADR-0031: Core daemon controller runtime and startup sequence

- Status: accepted
- Date: 2026-08-27
- Amended: 2026-09-08 — D6 gains a first-boot seed-manifest import step before the
  singleton seeds, and the `updates` singleton is seeded alongside the others
  (os-image-installer change)
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/02-core-daemon.md`; `docs/architecture/01-os-feature-map.md` §3.2, §4;
  ADR-0002, ADR-0008, ADR-0013, ADR-0017, ADR-0028; `docs/architecture/13-os-image-installer.md` D8, D17

## Context

The framework runtime mechanics (`core-daemon-design` D1–D4) are recorded in
ADR-0017/0013/0008, but two behaviors had no decision of record: the **startup
ordering** of the daemon (what must be read, mounted, started, and served in
what sequence) and the **routing** of action endpoints (imperative ops) to the
owning controller. Both follow directly from earlier ADRs but were never fixed
as a named decision; this ADR records them.

## Decision

### Startup sequence (D6)

After OS-disk unlock (host-encryption, ADR-0011), `core` starts in this order:

1. mount `config/var`; load the spec store, migrating schema forward (ADR-0013)
2. **import the installer seed manifest** (first boot only, before the singleton
   seeds): detect a versioned seed, validate it, and fold its install-time
   decisions into the store through the validated write path; a malformed seed
   aborts and the API never starts (fail-closed). When a seed is present, the
   singleton seed steps below skip and the seed's admin is authoritative over any
   config-provided seed admin (`os-image-installer` D8, D17, D23)
3. build default OTEL providers (sampling zero, no OTLP targets — the contract
   default) and the D8 provider handle (`core-daemon-design`)
4. start the event bus; register all controllers (each resolving its subsystem
   logger/meter through the provider handle — never holding providers itself, D8)
5. start controllers — reconciling immediately, **with pools absent** (ADR-0013:
   reconcile scope degrades, startup never blocks on pool import)
6. the storage controller imports pools as they become available; controllers
   dependent on pools (shares, apps, backup) converge after via the D3
   fail-and-retry/backoff loop
7. seed the singleton resources — `network`, `telemetry`, `oidc`, `certificates`,
   **`updates`** (empty spec: `channel=stable`, `autoApply=false`,
   `deferReboot=false`) and **`UnlockPolicy`** (empty spec, `presence-only`
   default) alongside the others — skipped when a seed was imported in step 2
8. start the API server (HTTPS, ADR-0028) — last, so the first admin request
   sees a converged pass

Spec-before-pool is fixed by ADR-0013; API-last makes the booted daemon present
already-reconciled status rather than a cold store. The seed-import step (2)
precedes the singleton seeds (7) so no empty-spec window exists and the update
and unlock-policy resources never 404 on an unconfigured host.

### Action-endpoint routing (D7)

The runtime routes `/v1/<kind>/{id}/actions/<name>` (e.g. `scrub`, `replace`,
`trigger`, `rollback`) to the handler it looks up from the owning controller's
`Actions()` map (`core-daemon-design` D1). Singleton resources (e.g. `updates`)
have no `{id}` segment — the path is `/v1/<kind>/actions/<name>`. Actions
may read spec, may mutate **status** (result observable, e.g. scrub status), and
never touch desired state (ADR-0002). They are audited exactly like spec writes.

Routing to the owning controller keeps one owner for the resource's lifecycle and
keeps actions out of the reconciler's diff. Actions are never encoded as
synthetic spec writes — that would pollute desired state and break the "action
does not change spec" contract scenario.

## Alternatives considered

- **API first, controllers later**: rejected — first requests would observe
  `Pending`/empty status and audit before controllers had a pass.
- **Actions as synthetic spec writes**: rejected — pollutes desired state and
  breaks the "action does not change spec" contract scenario.

## Consequences

- Boot ordering is independent of pools; startup never blocks on pool import.
- The API server comes up only after controllers have had at least one pass, so
  the first admin request observes reconciled status.
- Each imperative action is owned by a single controller, audited like a spec
  write, and observable through `status` and metrics — with no desired-state
  mutation.
- On first boot the seed manifest is folded in before the singleton seeds and
  before the API serves, preserving core as the only spec-store writer
  (ADR-0002); a malformed seed fails closed with no API (ADR-0013 posture).
- The `updates` and `UnlockPolicy` singletons are seeded with their default specs
  alongside the others, so their controllers reconcile from first boot and their
  resources never 404 on an unconfigured host.