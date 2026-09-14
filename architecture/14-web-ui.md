# Web-UI — the browser management console

> Discovery-phase design. Authored from the `web-ui-foundation` openspec change.
> The decisions D-W1–D-W17 here fix how the Web-UI (feature 8) is built, served,
> and authenticated: the React + TypeScript + Vite + Mantine static SPA built
> with Bun (D-W1), contract consumption with generated types (D-W2), single
> same-origin serving through a reverse-proxying static server (D-W3), the
> management-plane front door with loopback-bound `core` (D-W4), browser auth via
> session cookie + OIDC redirect with no tokens in the browser (D-W5), the
> current-principal capability endpoint (D-W6), the dev/e2e workflow (D-W7), and
> image integration through the A/B path (D-W8), plus the security, caching,
> data-layer, navigation, theming, and session-lifecycle decisions that fix how
> the console behaves (D-W9–D-W17). Serving topology is ADR-0033; the UI is a
> separate static server baked into the image (ADR-0012) and a thin client over
> the v1 API (ADR-0002) — the browser-side counterpart to the CLI
> (`docs/architecture/12-cli-client.md`). The API contract is the source of
> truth (`contracts/openapi/v1.yaml`, ADR-0019).

## 1. Purpose

The Web-UI is Amberhold's browser management console. It gives operators the full
management surface over the v1 API without SSH: declarative desired-state
editing, resource status, imperative actions, and the update/rollback flows —
all rendered against the operator's own capabilities. It is deliberately *thin*:
static assets only, no server runtime, no direct host access, and admission
(ADR-0018) remains the only enforcement point. This change is the foundation and
scaffold: project tooling, the serving topology, contract consumption, the app
shell, and the data layer. No resource feature screens land here.

## 2. Goals / Non-Goals

**Goals:**
- Fix the browser management-plane architecture end to end: build shape, contract
  consumption, serving topology, TLS ownership, browser auth, capability-aware
  rendering, image integration, and the dev/e2e workflow.
- Keep the Web-UI a thin client over the v1 API with no server-side runtime in
  the OS image.
- Reuse the existing machinery (certificates controller, A/B updates, telemetry)
  rather than inventing parallel paths.

**Non-Goals:**
- Designing or implementing feature screens beyond the app shell (this change is
  design + scaffold).
- A bespoke design system; Mantine 9 is the design system.
- SSR, a Node/BFF runtime in the image, or a separate UI update channel.
- Changing the server-side session/OIDC model (ADR-0020, ADR-0029).

## 3. Stack and build (D-W1)

The Web-UI is a React + TypeScript single-page application built with **Vite**,
using **Bun** as the package manager, script runner, and local runtime; **Mantine
9** as the component/design system; **TanStack Router** for type-safe routing; and
TanStack Query for API server-state. The build emits static assets only.

```
web-ui/  (repo nas/web-ui, module scoped to the SPA)
  src/            app code (router, client, data layer, screens)
  contracts/      vendored contracts/openapi/v1.yaml (D-W2)
  scripts/        typegen, drift check, dev/e2e helpers
  .bun-version    pinned Bun (D-W2)
  bun.lock        committed lockfile
  dist/           build output → baked under /usr/share/amberhold/web-ui (D-W8)
```

- **Bun is a build-time tool.** It is provisioned in the Linux build environment
  (the Lima VM and CI) that produces `web-ui/dist`. Neither Bun nor Node ships in
  the OS image — only the built static assets are baked.
- **Pinned.** The Bun version is pinned (`.bun-version`) and `bun.lock` is
  committed, so installs are reproducible and the build-time supply chain is
  controlled (ADR-0001 pinned-package pattern applied to the toolchain).
- **Why not Node/npm:** Bun is the project preference and delivers install/run in
  one tool with one lockfile. **Why not React Router:** TanStack Router's typed
  routes pair naturally with TanStack Query and the generated contract types.
  **Why not a server-rendered framework:** it needs a runtime in the image.

## 4. Contract consumption (D-W2)

`contracts/openapi/v1.yaml` is vendored into `web-ui/contracts/` (mirroring
`cli/contracts/openapi/v1.yaml`), TypeScript types are generated with a pinned
generator, and a conformance/drift check fails the build when the vendored
contract drifts from the source. Generated types are not committed; the vendor +
pinned generator + drift check are.

- **Generated types, hand-written client logic.** Type generation is pinned and
  runs at build time; the client (D-W5, D-W12) is hand-written and thin.
- **The conformance check** diffs `web-ui/contracts/openapi/v1.yaml` against the
  source `contracts/openapi/v1.yaml`; any divergence fails the build/CI. This is
  the same "contract is source of truth, logic is thin" stance as the CLI
  (12-cli-client.md D-C3), while accepting type generation where it clearly pays.

## 5. Serving topology (D-W3, D-W4)

A dedicated static server (Caddy, pinned Debian package) serves the SPA at `/`
and reverse-proxies `/v1/*` to `core`. The browser sees **one origin**; session
cookies (`SameSite=Lax`) and the OIDC redirect (`/v1/oidc/callback`) work without
any cross-origin configuration.

```
browser ──HTTPS──▶ front door (Caddy, :443)          loopback
                    ├── /            → /usr/share/amberhold/web-ui (SPA)
                    └── /v1/*        → core 127.0.0.1:8443 (HTTPS, trust mgmt CA)
```

- **D-W4 — the static server is the management-plane front door; `core` binds
  loopback.** Caddy is the only externally bound management-plane listener and
  the TLS terminator. It serves the management-plane serving certificate that
  the `certificates` controller manages under `/config/var/tls` (hot reload via a
  `systemd` path unit, ADR-0033), and proxies `/v1/*` to `core` on loopback.
  `core` keeps speaking HTTPS with the same managed material on `127.0.0.1:8443`;
  the front door trusts the management CA for that loopback hop and verifies the
  managed cert's `localhost` SAN — the cert controller always carries the
  loopback names (`localhost`, `127.0.0.1`) in the server-cert SANs
  (08-certificates-controller D-C2), so the hop stays verifiable in steady state,
  not just pre-network-reconcile.
- This is an image/config change, not a `core` code change: `core` already
  defaults to `--api-bind 127.0.0.1:8443`; only the image bake set `0.0.0.0:8443`
  before. The CLI (origin-only `--server https://nas`) is unaffected. The only
  direct-`:8443` consumer is the macOS harness, whose slirp `hostfwd` retargets
  from guest `8443` to the front-door guest port (`443`).
- **Why one terminator:** one certificate, one externally reachable origin, one
  place where fail-closed HTTPS is enforced (ADR-0028, ADR-0033). Two listeners
  with the same cert would double the surface and force CORS on `:8443`.

## 6. Browser auth (D-W5)

Password sign-in posts `/v1/sessions` (same-origin) and the browser carries the
HttpOnly session cookie; sign-out deletes the session. Sign-out is self-service —
any confirmed principal may destroy its own session without `sessions:write` —
and principals holding `sessions:write` (admin) may additionally destroy any
session, the incident-response path to force-log out a compromised account. OIDC
sign-in links to
`/v1/oidc/login`; `core` performs the authorization-code + PKCE exchange (ADR-0029)
and the callback sets the session cookie, after which the UI loads. **No bearer
token is ever issued to or stored by the browser** — nothing secret in
`localStorage` or cookie-accessible to script.

- Session cookie is HttpOnly + `SameSite=Lax`; admission adds an
  `Origin`/`Sec-Fetch-Site` check as defense-in-depth against drive-by cross-site
  requests. No CSRF token is introduced in v1 because the same-origin topology
  removes the cross-site state-changing surface.
- **Dev note:** password sign-in works through the Vite dev proxy (the proxy
  forwards `Set-Cookie`, which carries no `Domain`, and the browser binds it to
  the dev origin). OIDC is different: `redirectUri` must be `https` (ADR-0028),
  so OIDC development requires an HTTPS dev origin (e.g. run the front door
  locally), not a plain-HTTP Vite origin.

## 7. Capability endpoint (D-W6)

A read-only `GET /v1/sessions/current` returns the caller's principal identity
(`id`, `username`, `source`) plus its effective `roles` and `capabilities`. The
UI uses it to hide/disable actions it cannot perform; **admission remains
authoritative** — a rejected request is surfaced as a denial, never treated as a
UI error.

- Capability values reuse the exact ids already declared on the `Role` schema
  (`users:*`, `pools:*`, `shares:*`, `backups:*`, `apps:*`, `updates:*`,
  `network:*`, `telemetry:*`, `oidc:*`, `unlock-policy:*`, `metrics:read`,
  `audit:read`), and are declared as a closed `Capability` enum in the contract
  (D-W11) so contract, `core`, and the UI share one taxonomy.
- **Why:** the contract exposes no self-principal and no owner link on `Session`;
  without this a limited role could not render a permission-aware console.
  *Alternative — extend `Session` status with roles* was rejected: sessions and
  authorization are separate concerns, and OIDC roles are claim-derived.
  *Alternative — UI shows all actions and relies on 403s* was rejected: poor
  operator UX for read-only/auditor roles.

## 8. Dev and e2e workflow (D-W7)

- **Vite dev server** runs via Bun with `/v1` proxied to a configured `core`, so
  cookie/same-origin behavior is realistic in dev.
- **Contract-derived mocks** (MSW, fixtures generated from the vendored contract)
  allow UI-only development without a running `core`.
- **Playwright e2e** runs headless in CI against the built `dist` behind the real
  static server (or the harness's fake core), covering routing/proxy/cookie/auth
  behavior that unit tests cannot.
- Unit/component tests use Vitest + React Testing Library; axe runs on primary
  flows (D-W15).

## 9. Image integration (D-W8)

`web-ui/dist` is baked to `/usr/share/amberhold/web-ui` (the bake script already
anticipates this), the Caddy front-door unit is enabled, and Caddy's loopback
admin metrics are scraped and merged into `core`'s `/metrics` so the management
plane keeps one external Prometheus endpoint for all subsystems (ADR-0008). The
front-door admin `/metrics` URL is configurable via core's
`--frontdoor-metrics-url` flag (default matches the Caddyfile admin listener;
empty disables the merge), so the core↔front-door coupling is explicit rather
than a silent hardcode. The UI ships **only** through the A/B OS update path
(ADR-0006/0012) — no independent UI channel.

## 10. Serving and caching (D-W14)

Caddy serves the SPA with a history-API fallback (`try_files {path} /index.html`)
so deep links resolve on refresh. Content-hashed assets are served
`Cache-Control: public, max-age=31536000, immutable`; `index.html` is served
`no-store` so a new A/B slot is picked up immediately. No service worker in v1;
the app is served at the origin root.

## 11. Security headers (D-W9)

The front door sets a strict CSP alongside the served assets:

```
default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';
img-src 'self' data:; connect-src 'self'; frame-ancestors 'none';
base-uri 'none'; object-src 'none'
```

plus `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer`.
`style-src 'unsafe-inline'` is required in v1 because Mantine emits dynamic
inline styles. A nonce-based `style-src` is a hard follow-up and deferred.

## 12. Decisions D-W11–D-W17 (console behavior)

- **D-W11 — Capability taxonomy is contract-declared with an exhaustive UI map.**
  The v1 capability ids are a closed `Capability` enum in
  `contracts/openapi/v1.yaml`; `core`'s admission constants are covered by a
  conformance test asserting they match the enum (mirroring the CLI's contract
  conformance test). The Web-UI generates the enum as a TypeScript union and maps
  every member through an exhaustive `Record<Capability, ...>` so a new or renamed
  capability fails the build, and an unknown capability yields no affordance
  (fail-closed).
- **D-W12 — Reconcile-aware data layer with resourceVersion concurrency.** Reads
  use a key factory (`[kind]`, `[kind, id]`) and poll only non-terminal resources
  (`phase` in `Pending`/`Reconciling`/`Unknown`, or
  `status.observedGeneration < metadata.generation`) with backoff, pausing while
  the document is hidden; polling stops at a terminal phase
  (`Ready`/`Degraded`/`Error`). Writes carry the observed `metadata.resourceVersion`;
  a `409` triggers a refetch and a surfaced conflict (diff or retry), never a
  blind retry. Writes are pessimistic (no optimistic spec): a successful write
  seeds the cache from the response and polls to convergence; privileged action
  calls invalidate the affected resource and resume polling. Error envelopes
  (`code`, `message`, `details`, `requestId`) are surfaced with the request id for
  audit correlation.
- **D-W13 — Information architecture.** Top-level navigation groups mirror the
  feature map: **Overview**; **Storage** (pools, disks, datasets, zvols,
  snapshots, schedules); **Shares** (file-shares, block-shares); **Apps**
  (apps); **Backup** (backups); **Network** (network, certificates, oidc);
  **Identity** (users, roles, sessions, tokens); **System** (updates, telemetry,
  unlock-policy, audit, metrics). Navigation entries and actions are gated by
  read/write capabilities (D-W11); singleton resources get dedicated pages. A
  resource detail view presents spec, status (phase + reason + conditions), and
  the `observedGeneration` lag between wanted and actual. Editing uses generated
  forms for the common case with a YAML escape hatch for full-fidelity edits.
- **D-W15 — Testing strategy.** Vitest + React Testing Library with MSW backed by
  contract-derived fixtures; Playwright headless in CI against the built dist;
  axe on primary flows; visual regression deferred.
- **D-W16 — Session lifecycle edge cases.** A `401` on any request clears auth
  state and redirects to sign-in with a `next` parameter returning the operator to
  their original route after re-authentication. An OIDC failure query parameter
  renders a non-blocking banner. A `409` on a write refetches and shows a conflict
  notice while preserving the operator's edit. Session expiry during an unsaved
  form preserves the draft and prompts re-authentication before submitting.
  Multi-tab sign-out is handled by the next request in other tabs returning `401`.
- **D-W17 — Theming and branding.** The Mantine theme derives from the Amberhold
  brand tokens (the `docs`/`logo` palette) with light/dark modes; the logo asset
  is reused from the meta-repo `logo/`. No bespoke CSS system beyond Mantine and
  a small global stylesheet.

## 13. Risks / Trade-offs

- **Front-door trust of the loopback cert** → pin the management CA in the proxy
  config; the managed cert always carries the loopback SANs so the hop
  verifies; treat the loopback hop as internal and re-verify the front door's
  own external TLS fail-closed (ADR-0033).
- **Contract drift between generated types and `core`** → vendor + conformance
  check in CI, same pattern as the CLI.
- **Capability endpoint becomes a second authorization path** → spec it as
  read-only and advisory; admission stays the only enforcement, asserted by tests.
- **`SameSite=Lax` + no CSRF token** → same-origin topology removes cross-site
  state changes; `Origin`/`Sec-Fetch-Site` admission check now, revisit a token
  if a cross-site surface appears.
- **XSS in a management console** → no tokens in `localStorage`, strict CSP on the
  static server (D-W9), and dependency hygiene.
- **Front-door change breaks existing boot scenarios** → the harness `hostfwd`
  retargets to the front-door port and `wait_for_core`/`core-vs-real-host` poll
  `/v1` through the front door; verified before the change is considered green.
- **Metrics merge couples core to the front door** → the merge is best-effort and
  additive: core's own metrics remain authoritative and `/metrics` still serves
  when the front door is down or slow (bounded timeout, no error).
- **Caddy version skew / security** → pin the package in `versions.toml`; the
  trixie-security build carries CVE fixes, and the path unit reload picks up
  config without downtime.
- **Capability enum drift across contract/core/UI** → a `core` conformance test
  and an exhaustive TS `Record` make a new/renamed capability a build failure
  rather than a silent gap.
- **Polling load from many reconciling resources** → poll only non-terminal
  resources, back off, and pause on hidden tabs; terminal resources are not
  polled.
- **Stale-write conflicts frustrate operators** → refetch-and-diff rather than
  blind retry; keep the operator's edit so it can be re-applied deliberately.
- **Bun toolchain (build-time) supply chain / availability** → pin the version
  (`.bun-version`) and commit `bun.lock`; install Bun in CI/VM from the official
  release with a checksum; it never ships in the image, so OS-attack surface is
  unchanged.

## 14. Open Questions

- Component/theme token mapping from the docs brand — cosmetic, decided during
  scaffolding.
- Exact pinned Caddy version (trixie `2.6.2-12` vs `trixie-backports`) — decided
  with `infra` when the package is added.