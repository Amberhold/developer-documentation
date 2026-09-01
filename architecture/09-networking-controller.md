# Networking Controller — Network Planes, Hostname, DNS, and NTP

> Discovery-phase design. Authored from the `networking-controller` openspec
> change. The decisions D-N1–D-N7 here fix how the `NetworkController` reconciles
> the singleton `network` resource (ADR-0014) so the host's hostname, DNS, NTP,
> and per-plane interface bindings are configured via systemd-networkd, with
> status and metrics, and so the plane membership is consumable by the
> certificates, file-share, block-share, and app controllers for binding.
> ADR-0014 fixes the *decision* (two planes, roles + per-interface IP assigned at
> install); this document fixes the *internal mechanics*, built on the
> framework-first runtime in `docs/architecture/02-core-daemon.md` (D1–D10,
> ADR-0031). The feature map (feature 10 networking) is in
> `docs/architecture/01-os-feature-map.md`.

## 1. Purpose

Networking is the last declared control-plane capability with zero
implementation: the `network` singleton, its OpenAPI contract
(`Network`/`InterfaceBinding`/`InterfaceStatus` in `contracts/openapi/v1.yaml`),
the RBAC capabilities (`network:read`/`network:write`), and the metrics
(`amberhold.network.interface.link`/`.reconciled` in `contracts/metrics/
catalog.yaml`) are all declared, but no controller reconciles the resource. The
`certificates` controller already reads the `network` resource for SAN
derivation (`deriveSANs`) and today falls back to `localhost` because no
controller populates it. This slice delivers the networking substrate the rest
of the control plane was designed around: a `NetHost` facade pinned to
systemd-networkd, and the `NetworkController` reconciling the singleton with an
apply-and-verify fail-closed strategy (ADR-0014, ADR-0013). It follows the same
singleton-controller and thin-facade conventions as the certificates (ADR-0028),
storage (ADR-0024/0021), and scheduling (ADR-0022) slices.

## 2. Goals / Non-Goals

**Goals:**
- Reconcile hostname, DNS, NTP, and the management/data plane interface bindings
  from the `network` singleton via a systemd-networkd-backed `NetHost` facade
  (D-N1/D-N2).
- Apply-and-verify fail-closed config (D-N3): a binding is written, verified
  (link up + address present) before `ready`, and the prior working config is
  restored on verify failure — never strand the management plane.
- Surface per-interface status (link state, plane, address) and the catalog
  `amberhold.network.interface.link`/`.reconciled` gauges (D4, D10).
- Make plane membership readable by consumers (D-N4/D-N5) so SMB/NFS/NVMe-oF
  bind to the data plane and app compose ports to the management plane.
- Make the cert controller's SAN derivation read real reconciled state (D-N6).
- Keep the single-LAN v1 default behavior-preserving: the binding sweep is inert
  when no data plane is declared.
- Architecture doc lands in the same change (D-N7).

**Non-Goals:**
- Firewall / traffic rules (deferred per the feature map).
- Consuming remote network shares; we only bind host interfaces.
- WiFi/wwan, VLANs, bonds, bridges — a single plain L2 interface per plane in v1.
- Runtime plane reassignment after install (roles assigned at install, ADR-0014);
  spec edits converge existing bindings only.
- Anything that would lock the box out without a verified, reversible config
  (guarded by D-N3 apply-and-verify).

## 3. Decisions

### D-N1: systemd-networkd is the host mechanism
`.network` files land under the writable config partition (`config/var/etc/
systemd/network/`, referenced via `/etc/systemd/network` symlinks — the RO-root /
config-var convention, ADR-0001/ADR-0013), reloaded with `networkctl reload` and
applied per-interface with `networkctl reconfigure <iface>`. Hostname uses
`hostnamectl set-hostname`, DNS `systemd-resolved` (`resolvectl dns/domain`), and
NTP the `systemd-timesyncd` config.
- *Why:* Declarative per-interface files map 1:1 to an `InterfaceBinding`; the
  appliance image already runs systemd for journald (ADR-0008/0013), so it is
  already present; files survive reboot; no extra runtime dependency.
- *Alternatives:* NetworkManager/nmcli (heavier, higher-level — overkill for two
  static planes); iproute2 + ad-hoc (no reboot persistence, hand-rolled).

### D-N2: NetHost facade is the only host access path
`core/internal/networking/` exposes a narrow `NetHost` interface (interface
list/status with link state + plane + address, apply binding, verify link,
hostname, DNS, NTP) with an in-memory fake for tests — mirroring `storage/host`,
`shares`, `backup/restic`, and `blockshares`. The controller never shells out
directly.
- *Why:* Consistent with every prior slice; enables deterministic tests and a
  simulated multi-plane fixture.

### D-N3: Apply-and-verify fail-closed
Each interface binding is written, then verified (link up + address present)
before the resource reports `ready`; on verify failure the prior `.network`
state is restored and status reports `degraded`/`error` with a reason. Hostname/
DNS/NTP apply-and-verify similarly: probe before commit.
- *Why:* The management plane carries the API that runs the reconcile; a bad
  static IP must never be silently committed. This is the one controller where
  "host first, then spec" (D3 ordering) is deliberately reversed — host
  last-valid, then report.
- *Alternatives:* apply-only-with-degrade (simpler but can strand the box);
  keepalive watchdog (stronger but complex; deferred — apply-and-verify plus the
  single-LAN default make lockout improbable in v1).

### D-N4: Plane membership resolved from the store, not injected state
Consumers read plane membership by listing the `Network` resource from the store
(exactly how `deriveSANs` works today). A small shared resolver
(`internal/networking/Planes(store)`) returns `{management, data}` interface
lists; consumers call it and pass the result into their backends.
- *Why:* No new event plumbing; the single source of truth stays the spec store;
  works when the network resource is absent (returns empty → consumers keep
  today's single-LAN behavior).
- *Alternatives:* pushing plane membership into each consumer's spec (couples
  unrelated resources); an injected binding at app composition (loses
  per-resource freshness).

### D-N5: Consumer binding is backend-level, behavior-preserving by default
- **SMB**: `renderConfig` gains the data-plane interface list; when non-empty it
  adds `interfaces = <list>` and `bind interfaces only = yes` to `[global]`.
- **NFS**: `sharenfs` default grants resolve against data-plane subnets when
  declared; the existing host/IP allowlist machinery is reused.
- **NVMe-oF**: `BindPort`/`PortBound` scoped to data-plane interfaces when
  declared (D-B4).
- **Apps**: compose port-publishing override binds published ports to
  management-plane host IPs by default.
All four are no-ops when no data plane is declared — the single-LAN default is
unchanged.
- *Why:* Binding is a property of how the *backend* renders host state, so it
  lives at the backend/override layer, not in the resource specs.

### D-N6: Cert coupling becomes real, ordering matters
`deriveSANs` already reads `Network`; this change makes that read meaningful. The
daemon seeds `network` before `certificates` and registers the network controller
before certs, so on first reconcile certs see the real hostname/IPs rather than
`defaultSANs`.
- *Why:* Closes the latent dependency the cert slice already declared without
  changing its contract.

### D-N7: Architecture doc lands in the same change
This document (`docs/architecture/09-networking-controller.md`) documents
D-N1–D-N6 and is wired into the `zensical.toml` nav (per AGENTS.md, verified with
`zensical build --clean`).

## 4. Reconciliation

The `NetworkController` owns the singleton `network` kind (D1) and converges the
desired hostname, DNS, NTP, and plane bindings against the `NetHost` facade
(task 3.2). Each pass:
- **Hostname/DNS/NTP**: apply when the host differs from the desired state;
  probe-before-commit (D-N3) and idempotent no-op when converged (D9).
- **Per-plane interface binding**: for each declared binding, write the
  systemd-networkd `.network` file under `config/var`, apply it with
  `networkctl reconfigure`, then verify the link is up with the desired address.
  On verify failure, restore the prior config and report `degraded`/`error` with
  a reason (D-N3).
- **Status**: write `status.state` (ready/degraded/error) and
  `status.interfaces[]` (per-interface plane, address, link state) through the
  store's serialized path (D4).
- **Metrics**: publish the `amberhold.network.interface.link` and
  `.reconciled` gauges (labels `interface`, `plane`) from the catalog (D8/D10);
  the controller never builds providers.

## 5. Plane membership for consumers

The `Planes(store)` resolver (D-N4) returns `{management, data}` interface lists
from the `network` resource, empty when the resource is absent. Consumer
controllers resolve membership each pass and pass it into their backends:
- `certificates` reads the hostname + management-plane IPs for SANs (D-N6).
- `shares` SMB/NFS backends bind to the data plane (D-N5).
- `blockshares` scopes `BindPort`/`PortBound` to the data plane (D-N5).
- `apps` binds published compose ports to the management plane (D-N5).

## 6. Status and observability

The controller writes the declared `NetworkStatus` shape (state, interfaces[]
with plane/address/link) through the store (D4). Metrics
(`amberhold.network.interface.link`, `amberhold.network.interface.reconciled`)
are published from the catalog (`contracts/metrics/catalog.yaml`) through the
daemon registry (D8/D10); the controller never builds providers and emits no
unregistered network metric names (ADR-0008).

## 7. Risk / trade-offs

- **Management-plane lockout** → apply-and-verify (D-N3): a binding must verify
  before commit; failed verifies restore prior config. Residual risk: verify
  succeeds but a later dependency (DHCP lease loss, link flap) strands the plane
  — mitigated by the single-LAN default and the data plane never carrying
  API/UI.
- **systemd-networkd vs. existing host tools** → the appliance image already runs
  systemd for journald; `.network` files are declarative and reboot-persistent.
  Trade-off: NetworkManager-style auto-config (WiFi/wwan) is out of scope, in the
  non-goals.
- **Binding sweep inert in single-LAN default** → the four consumer deltas are
  untestable against real multi-plane hardware in most CI; the fake `NetHost` +
  a simulated two-plane fixture covers the logic, and the inert default means the
  sweep cannot regress single-LAN behavior.
- **Cert SAN staleness** → cert re-issue is driven by the existing hostname-
  change reconciliation; network reconcile precedes certs at startup (D-N6).
- **Plane resolver drift** → consumers read live from the store each reconcile,
  so membership changes converge on the next pass with no cache to go stale.

## 8. Migration plan

No data migration: the `network` resource is new; `seedNetwork` creates the
singleton on startup (mirroring `seedCertificates`), defaulting to an empty spec
(single LAN) so existing installs converge to current behavior. Rollback: revert
the change; `core` keeps serving with the previous network state since the
controller never destroys config it cannot verify (D-N3). Cert SANs fall back to
`defaultSANs` as today.

## 9. Open questions (implementation-time)

None that would change the specs, approach, or task breakdown. systemd-networkd
file-naming conventions, exact `resolvectl`/`timesyncd` invocations, and compose
port-override mechanics are implementation details resolved during the coding
task (design open questions).
