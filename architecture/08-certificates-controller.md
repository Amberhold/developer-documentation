# Certificates Controller — Management-plane TLS Serving Certificates

> Discovery-phase design. Authored from the `certificates-controller` openspec
> change. The decisions D-C1–D-C7 here fix how the `Certificates` controller
> reconciles the singleton `certificates` resource (ADR-0028) so the management
> plane reliably serves HTTPS: a dedicated resource and controller, three trust
> modes (built-in CA, ACME, manual import), fail-closed serving, and hot
> reload on rotation. ADR-0028 fixes the *decision* (HTTPS on the management
> plane in v1, built-in CA / ACME / manual-import trust); this document fixes
> the *internal mechanics*, built on the framework-first runtime in
> `docs/architecture/02-core-daemon.md` (D1–D10, ADR-0031). The feature map
> (feature 10 networking, management-plane HTTPS) is in
> `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

Storage, shares, block shares, apps, and backup are implemented and green. The
management plane's HTTPS (ADR-0028) is the remaining gap: the API server serves
HTTPS from static `--tls-cert/--tls-key` file paths wired in the auth slice, but
nothing manages those files — there is no CA, no renewal, no trust-mode
selection, and no contract surface for certificates. OIDC (ADR-0029) is blocked
on an HTTPS `redirect_uri`; the web-UI cannot be served securely until a cert
controller exists. This slice delivers the certificate lifecycle so the
management plane reliably serves HTTPS. The contract surface is fully declared
(the `certificates` singleton in `contracts/openapi/v1.yaml`, the
`amberhold.certificates.*` metric catalog in `contracts/metrics/catalog.yaml`),
but there is no controller and no pki primitive in the host facades. This slice
adds both: a `PKIHost` facade (CA generation, server-cert issue, read, atomic
rotate, install to `config/var`) with an in-memory fake, and the
`CertController` owning the `certificates` kind.

## 2. Goals / Non-Goals

**Goals:**
- A dedicated singleton `certificates` resource reconciled by a dedicated cert
  controller (D1: one kind per controller, D-C1).
- SANs derived from the `network` resource's hostname (plus management-plane
  IPs when declared), avoiding a second source of truth for identity — D-C2.
- `builtin` mode with lazy CA bootstrap on first reconcile (no installer
  dependency) and server-cert renewal inside the renewal window — D-C3.
- `manual` mode: validate the uploaded PEM chain + key (key matches the
  certificate, certificate not expired), install, never renew — D-C2/D-C7.
- `acme` mode behind a provider facade with a fake for tests; full lego/autocert
  client deferred — D-C4.
- Fail-closed serving: no valid serving cert ⇒ the management plane refuses
  plain-HTTP fallback and reports degraded status with a reason — D-C5.
- Hot reload: `tls.Config.GetCertificate` on a stable cert/key path with
  reload-on-rotate (atomic temp+rename), so rotation needs no daemon restart —
  D-C6.
- Certificate material on the OS-disk `config/var` partition, referenced via
  `/etc` symlinks (writable-config convention, ADR-0001) — D-C7.
- Cert-controller observability: expiry/next-rotation/state gauges per the
  all-features-observable rule — task 3.4.

**Non-Goals:**
- **OIDC sign-in** (ADR-0029) — a separate change that consumes this one as a
  prerequisite.
- **Full ACME client integration** — lego/autocert, challenge-mode selection
  (HTTP-01 / TLS-ALPN-01), and public reachability handling are deferred; this
  slice ships the facade + fake only (D-C4).
- **Network resource changes** — management-plane binding and plane membership
  stay as specified by ADR-0014.
- **Data-plane transport security** — TLS covers the management plane only
  (ADR-0028); SMB/NFS/NVMe-oF are unaffected.
- **Installer-side CA generation** — built-in CA is generated lazily on first
  reconcile (D-C3), so no installer dependency.

## 3. Decisions

### D-C1: Dedicated `certificates` singleton resource
A new `certificates` kind owned by a new cert controller, cross-referencing
`network.hostname` for SANs.
- *Why:* The D1 convention is one owned resource kind per controller; folding
  TLS into the networking controller would either give it two kinds or make it
  reconcile a foreign kind. A dedicated resource keeps ownership, status writes,
  and admission clean.
- *Alternatives:* TLS fields on the `Network` resource (rejected: bends D1,
  mixes transport security into L3 config); TLS on `Network` with a separate
  controller (rejected: splits desired state and status across two kinds).

### D-C2: SAN sources from the `network` resource
The server certificate's SANs are derived from `network.spec.hostname` (plus
management-plane IPs when the `network` resource declares them).
- *Why:* The management-plane identity is hostname-based; deriving SANs avoids
  duplicating identity in two resources and keeps `certificates` a thin
  transport-security resource.
- *Alternative:* SANs declared directly on the `certificates` spec (rejected:
  two sources of truth for the same identity).

### D-C3: Lazy built-in CA bootstrap on first reconcile
In `builtin` mode, the controller generates the root CA and a server certificate
on the first reconcile if the CA is absent — no installer dependency.
- *Why:* There is no installer yet; lazy bootstrap keeps the slice testable and
  robust to a clean `config/var`, and matches the facade-first house pattern.
- *Alternative:* CA generation at install time (ADR-0028 wording) — deferred
  because the installer does not exist and first-reconcile bootstrap is a
  superset.

### D-C4: ACME behind a provider facade
`acme` mode is expressed in the contract and spec, but implemented as a host
facade with a fake for tests, mirroring restic/nvmet/mdadm. Full lego/autocert,
challenge selection, and reachability are out of slice scope.
- *Why:* ACME requires outbound network, a DNS name, and public reachability —
  none exist in this slice; a facade keeps the resource and status model honest
  without dragging in a heavy dependency.
- *Alternative:* Full lego now (rejected: premature, depends on networking +
  installer + firewall posture).

### D-C5: Fail-closed serving
If no valid serving cert exists at serve time, the daemon refuses plain-HTTP
fallback and the `certificates` status reports degraded with a reason. The
management plane is HTTPS-only per ADR-0028.
- *Why:* ADR-0028 reversed the plain-HTTP baseline; a fail-open fallback
  silently reintroduces it and would also undermine the OIDC `redirect_uri`
  guarantee.
- *Alternative:* Fail-open plain HTTP with degraded status (rejected: violates
  ADR-0028, defeats the OIDC prerequisite).

### D-C6: Hot reload via `tls.Config.GetCertificate`
The API server serves with `tls.Config.GetCertificate` reading a stable cert/key
path with a small cache that is invalidated when the controller rotates the files
— no daemon restart on renewal.
- *Why:* Renewals happen on a cadence; restarting the daemon per rotation
  interrupts management access and complicates the startup sequence (D6).
- *Alternative:* Static `ListenAndServeTLS` with daemon restart (rejected:
  disruptive); controller-driven reload channel (rejected: more moving parts
  than a path watch).

### D-C7: Material on `config/var` with `/etc` symlinks
CA, key, and server cert live under the OS-disk `config/var` partition; `/etc`
symlinks reference them (writable-config convention, ADR-0001).
- *Why:* An immutable root (ADR-0001) cannot hold writable cert state; the
  encrypted OS-disk container keeps the CA key at rest (ADR-0011 pattern).
- *Consequence:* The `manual` import payload (uploaded chain + key) is a
  one-shot transport through the write-only `spec.manual` fields; after a
  successful install the controller consumes it out of the spec store, so the
  private key never persists in desired state (bbolt or versioned snapshots)
  or survives a rollback restore.
- *Alternative:* Material in the spec store (rejected: private keys do not
  belong in desired state).

## 4. Trust-mode reconciliation

The `CertController` reconciles the singleton `certificates` kind (D1) by
dispatching on `spec.mode` (task 3.2):

- **`builtin`** (default): if no CA exists on `config/var`, generate the CA and
  a server certificate signed by it (D-C3). If the server certificate is within
  the renewal window, issue a replacement signed by the same CA. Re-issue when
  the `network` hostname changes (SANs no longer cover it).
- **`manual`**: validate the uploaded chain + key (D-C2/D-C7), install them,
  consume the upload out of the spec store (D-C7: no private key persists in
  desired state), and never renew — an expired installed import degrades
  fail-closed with a reason.
- **`acme`**: obtain/renew through the provider facade (D-C4); the full client
  is deferred.

Every pass computes the serving-certificate status (state, issuer, serial,
expiry, nextRotation, error) and writes it through the store's serialized path
(D4), and publishes the `amberhold.certificates.*` gauges (task 3.4).

## 5. Fail-closed serving and hot reload

The API server is HTTPS-only (ADR-0028). It serves with
`tls.Config.GetCertificate` over a stable cert/key path (D-C6): on each
handshake it loads the on-disk certificate, and when the file content differs
from the cached certificate it replaces the cache (atomic temp+rename on the
controller side, D-C7). If no valid certificate exists, `GetCertificate` returns
an error and the listener refuses the handshake — there is no plain-HTTP
fallback (D-C5); the `certificates` status reports degraded with a reason. The
existing `--tls-cert/--tls-key` flags remain as an override path (they win over
`GetCertificate`) so dev and manual setups keep working (design migration plan).

## 6. Status and observability

The controller writes the declared `CertificatesStatus` shape (state, issuer,
serial, expiry, nextRotation, error) through the store (D4). Metrics
(`amberhold.certificates.expiry`, `amberhold.certificates.next.rotation`,
`amberhold.certificates.state`) are published from the catalog
(`contracts/metrics/catalog.yaml`) through the daemon registry (D8/D10); the
controller never builds providers.

## 7. Risk / trade-offs

- **Fail-closed can brick management access if the cert path breaks** →
  mitigation: `builtin` mode self-heals by regenerating on reconcile;
  manual/acme failures surface a precise degraded reason; rotation happens well
  inside the renewal window.
- **GetCertificate cache invalidation race between rotate and reload** →
  mitigation: atomic file replacement (write temp + rename) at a stable path;
  reload on each handshake when the on-disk cert differs from the cached one.
- **ACME facade defers a real dependency decision** → mitigation: the facade
  interface keeps the resource/spec model stable so the full client drops in
  without contract churn.
- **SAN derivation couples cert controller to `network` resource** → mitigation:
  the coupling is read-only (hostname reference); a missing/unset hostname falls
  back to management-plane IPs, keeping the controller operable
  pre-network-reconcile.

## 8. Migration plan

No existing deployment migrates — this is greenfield management-plane TLS. The
slice lands the `certificates` resource and controller; the daemon's existing
`--tls-cert/--tls-key` flags remain as an override path so dev and manual setups
keep working. Rollback is a pointer revert in the core submodule; the OS-disk
cert material is on `config/var` and does not affect data pools.

## 9. Open questions (implementation-time)

None that would change the specs, approach, or task breakdown. ACME
challenge-mode selection and installer-time CA generation are deliberately
deferred (see Non-Goals) and do not alter the contract surface.
