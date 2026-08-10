# ADR-0012: Web-UI as a separate static server baked into the image

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §4 (skeleton), §3.2; ADR-0001, ADR-0002, ADR-0006

## Context

The Web-UI (feature 8) is a thin client over the API (ADR-0002), but the feature
map never said where it runs. Because the root is a read-only squashfs image
(ADR-0001), the UI cannot be installed or mutated post-deploy, so its hosting
model must be decided up front. This affects the update story: whatever hosts the
UI rides the A/B path or a separate channel.

## Decision

The Web-UI is static assets served by its own small HTTP server baked into the
image, served independently — not proxied by or served from the `core` daemon.
Because it lives in the read-only image, it updates through the A/B update
feature (ADR-0006), same as `core`.

## Alternatives considered

- **Served by `core`**: one less process, but couples the API process and the UI
  lifecycle and release cadence.
- **UI as a compose app workload**: decouples UI from OS updates, but makes the
  management console depend on the app plane (containerd) being up — a
  chicken-and-egg problem at install and during app-plane failures.

## Consequences

- The image contains a small static-file server (e.g. `caddy` or an equivalent)
  plus the UI build artifacts.
- The Web-UI is observable like any subsystem: its own process, HTTP server, and
  metrics (ADR-0008).
- UI updates ship as OS updates; there is no independent UI update channel.
- The skeleton diagram shows the Web-UI as its own server in the image, talking
  to the API over HTTP + RBAC admission.
