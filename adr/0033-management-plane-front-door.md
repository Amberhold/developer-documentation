# ADR-0033: Management-plane front door — static server terminates browser TLS, `core` binds loopback

- Status: accepted
- Date: 2026-09-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/14-web-ui.md` (D-W3, D-W4, D-W8, D-W9, D-W10);
  `docs/architecture/01-os-feature-map.md` features 8, 10; ADR-0012, ADR-0018,
  ADR-0020, ADR-0028, ADR-0029

## Context

ADR-0012 fixes the Web-UI as a separate static server baked into the image, but
explicitly leaves the serving topology, TLS ownership, and same-origin story
open. ADR-0028 commits the management plane to HTTPS-only in v1 and ADR-0020/0029
authenticate the browser with an HttpOnly session cookie and an OIDC redirect,
both of which need a stable same-site origin. The `certificates` controller
(ADR-0028, `docs/architecture/08-certificates-controller.md`) owns the managed
serving certificate under `/config/var/tls` and serves it via hot-reloading
`GetCertificate` in `core`. Nothing yet decides *where* the browser TLS
terminates, whether the API and UI share one origin, or how the API server is
bound.

## Decision

The management plane is a **single same-origin entry point** fronted by a
dedicated static server (Caddy, pinned Debian package) — the **management-plane
front door**:

- The front door is the **only externally bound** management-plane listener and
  the **TLS terminator**. It serves the managed serving certificate under
  `/config/var/tls` (reloaded on rotation via a `systemd` path unit) and
  reverse-proxies `/v1/*` to `core` on loopback.
- `core` **binds loopback** (`127.0.0.1:8443`) and continues to speak HTTPS with
  the same managed material. The front door trusts the management CA for the
  loopback hop.
- The front door serves the SPA at `/` with a history-API fallback, hashed-asset
  immutable caching, a `no-store` entry document, and the D-W9 security headers.
- The front door exposes loopback-only admin metrics that `core` scrapes and
  merges into `/metrics` (best-effort, core authoritative; ADR-0008).

Browser auth stays the ADR-0020/0029 model: password posts `/v1/sessions`
same-origin; OIDC redirects through `/v1/oidc/login` and the callback; no token
is ever issued to the browser. Admission (ADR-0018) remains the only enforcement
point; the current-principal capability endpoint (`GET /v1/sessions/current`,
`docs/architecture/14-web-ui.md` D-W6) is read-only and advisory.

## Alternatives considered

- **`core` remains the external TLS terminator and the static server is a second
  listener with the same cert**: two externally reachable terminators, more
  surface, and CORS on `:8443` for a cross-origin UI. Rejected — one certificate,
  one externally reachable origin, one place where fail-closed HTTPS is enforced
  (ADR-0028).
- **Plain HTTP on the loopback hop**: muddies the HTTPS-only posture of
  ADR-0028; the front door trusts the management CA instead.
- **nginx as the front door**: heavier config and no native Prometheus metrics.
  Rejected in favor of Caddy (ADR-0012's example; fits the pinned-package
  pattern).
- **An in-repo Go front door**: owning a reverse proxy is more security surface
  than it is worth when a pinned, security-updated package exists.

## Consequences

- One origin for browser sessions and OIDC redirects; the CLI/API keep working by
  pointing at the same origin (`https://nas/v1`). No CORS, no `SameSite=None`.
- Fail-closed serving (ADR-0028) now applies at the front door; the `certificates`
  status reports degraded with a reason until a valid certificate converges.
- This is an image/config change, not a `core` code change: `core` already
  defaults to `--api-bind 127.0.0.1:8443`; only the image bake had set
  `0.0.0.0:8443`. The only direct-`:8443` consumer today is the macOS dev
  harness, whose slirp `hostfwd` retargets from guest `8443` to the front-door
  guest port in the same change.
- Certificate rotation takes effect without restarting `core` or the front door
  (`systemd` path unit reload).
- The Web-UI ships and updates only through the A/B path (ADR-0006/0012); the
  front door and its assets are baked into the image.
- Front-door process/HTTP metrics are observable through the single
  management-plane `/metrics` endpoint (ADR-0008) even when the front door is
  unreachable (best-effort merge).