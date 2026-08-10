# Amberhold OS — Feature Map & Architecture

> Discovery-phase capture. Established via the `os-feature-map` openspec change.
> This is the anchor for per-subsystem design work; each capability below gets
> its own openspec change and spec before implementation.

## 1. Purpose

Agree on the target feature inventory of the Amberhold NAS OS at minimal detail,
record the architecture decisions that shape it, and sketch the high-level
architecture (a declarative reconciler) that subsequent changes build on.

## 2. Feature map

```
┌──────────────────────────────────────────────────────────────┐
│  CONTROL PLANE  (management)                                  │
│  API + CLI      Web-UI       RBAC/Users      Metrics          │
│  Networking (mgmt IP)   Logging + audit                       │
│  Installer (first boot)                                       │
│   │  declarative desired-state → core daemon reconciles       │
└──────────────────────┬───────────────────────────────────────┘
┌──────────────────────▼───────────────────────────────────────┐
│  DATA PLANE  (host features)                                  │
│   disks ─► pool/vdev creation, replacement, scrub, SMART      │
│   ZFS (data only) ─► datasets · snapshots · zvols            │
│        │                             │                        │
│   SMB/NFS ┘ (share a dataset)   NVMe-oF ┘ (nvmet target,     │
│                                     serve zvols, no client)   │
│   containerd/compose ──► full compose, ZFS-backed volumes     │
└──────────────────────┬───────────────────────────────────────┘
┌──────────────────────▼───────────────────────────────────────┐
│  OS IMAGE                                                     │
│   Debian · squashfs RO root (not ZFS) · writable config/state │
│   · A/B dual-slot · updates as a product feature              │
└───────────────────────────────────────────────────────────────┘
```

| # | Feature | Minimal scope |
|---|---------|---------------|
| 0 | OS image | Debian base, read-only squashfs root, A/B dual-slot, writable state off-root |
| 1 | System updates | Product feature: trigger, progress, reboot, rollback via UI/API |
| 2 | ZFS storage | Pools, datasets/properties, snapshots (scheduled with retention), zvols (data-only), native encryption |
| 3 | SMB + NFS shares | Share a dataset over the network; share auth via linked users |
| 4 | NVMe-oF shares | nvmet kernel target exporting zvols as block devices to remote hosts |
| 5 | App workloads | Full docker-compose on containerd; ZFS-backed volumes; images on a configured dataset |
| 6 | API + CLI | First-class HTTP API (OpenAPI); thin CLI client |
| 7 | RBAC + users | NAS user DB; roles per capability; optional link to system/SMB users |
| 8 | Web-UI | Management console over the API; no direct host access |
| 9 | Observability | Prometheus metrics for every subsystem |
| 10 | Networking | Management network: management IP (DHCP or static), hostname, NTP; where API/UI/shares bind |
| 11 | Pool storage | Disk inventory; pool/vdev creation; disk replacement |
| 12 | Installer | First-boot: assign storage layout roles (decision 7); import existing pools with role reassignment; admin bootstrap |
| 13 | Logging + audit | System logs and audit log of admin actions; retention |
| 14 | Disk health | Scrub schedule; SMART health monitoring |

### Deferred areas (future paths, out of v1 scope)

These are deliberately not v1 features. They are recorded here so they read as
explicit decisions (design decision D6 in the `scope-missing-plane` change), not
silent gaps:

- **Backup / replication** — out of scope for v1. ZFS snapshots (in scope,
  feature 2) are the primitive a future replication feature builds on.
- **TLS / firewall** — external network exposure hardening is out of scope for v1;
  the management network binds per feature 10. In v1 the API/UI serve plain HTTP
  on the management LAN (no TLS, no certificate management); the skeleton diagram
  reflects this baseline.
- **Alerting** — no push-based alerting (email/webhook) in v1; status is surfaced
  via API/UI and Prometheus metrics (feature 9).
- **AD/LDAP join** — no directory-service integration in v1; identity is the
  NAS-local user DB (ADR-0005).

## 3. Architecture decisions

### 3.1 OS image: read-only squashfs root, A/B boot (not ZFS-on-root)
- **Decision**: root is an immutable squashfs image; ZFS is reserved for data pools.
- **Alternatives considered**: ZFS-on-root (rich snapshots of OS state, but complex
  bootloader/kernel-module story and couples boot to pool state); plain RO partition.
- **Rationale**: simplest A/B story; root cannot drift; everything user-facing lives
  in data pools.
- **Implications**:
  - No post-deploy package installs — kernel modules (ZFS, `nvme-target`), samba,
    containerd must all be baked into the image. Image build is a first-class feature.
  - Writable state (configs, spec store, container images) must live on configured
    datasets, not on `/`.
  - A/B = two slots; bootloader selects active slot; rollback = fall to the other.

### 3.2 Control plane: declarative desired-state API
- **Decision**: the API is declarative desired-state, reconciled by the core daemon.
- **Alternatives considered**: imperative actions (simpler now, but drift/rollback are
  hard); hybrid.
- **Rationale**: idempotent, auditable, natural fit for a "system of record" NAS; UI
  and CLI become thin clients; drift detection falls out of the model and rollback is
  composite (spec-store revert + ZFS snapshot rollback, ADR-0002).
- **Implications**: core daemon is a reconciler; spec store is the source of truth;
  RBAC is enforced at API admission, not in the UI.

### 3.3 NVMe-oF: nvmet kernel target, serve-only
- **Decision**: kernel `nvmet` target, controlled by the daemon via configfs; we serve
  zvols, we do not consume remote NVMe-oF.
- **Alternatives considered**: SPDK userspace target (richer — RDMA etc. — but heavier
  and not needed for the serve-only scope).
- **Implications**: `nvme-target` kernel module in the image; block-shares controller
  writes configfs; no client-side story, so A/B boot never interacts with remote block.

### 3.4 App workloads: full docker-compose, ZFS-backed volumes
- **Decision**: full docker-compose semantics on containerd; container volumes are
  ZFS-backed — a dataset per volume by default (zvol as explicit escape hatch,
  ADR-0004).
- **Alternatives considered**: reduced/own schema (fragments ecosystem compatibility);
  plain-directory volumes (no snapshots/replication).
- **Implications**: images and compose state live on a configured app dataset; app
  data participates in the snapshot/replication story.

### 3.5 Identity: separate NAS DB with optional link
- **Decision**: RBAC identity is a NAS-local user DB (API/UI/CLI). A separate,
  optional link materializes only where POSIX/SMB ownership is required — and is
  auto-created when a share grants access.
- **Alternatives considered**: pure system accounts (couples RBAC to host accounts);
  fully separate (share auth disconnected from users).
- **Implications**: "create a share user" ⇒ NAS user + optional system UID + optional
  samba passdb entry.

### 3.6 Updates: a product feature, not just image mechanics
- **Decision**: A/B is the mechanism; system updates (trigger, progress, reboot,
  rollback) are exposed through the UI/API.
- **Implications**: an update controller with a reboot in the middle of the flow;
  progress reporting; boot-fail and manual rollback paths.

### 3.7 Storage layout: configurable at install
- **Decision**: the roles of datasets (system/config, app/images, data) are assigned
  at install time, not fixed.
- **Implications**: an install-time assignment step; everything else points at the
  assigned datasets.

### 3.8 Metrics: every subsystem observable
- **Decision**: Prometheus metrics for pools, shares, apps, API, updates.
- **Implication**: each controller exposes counters/gauges; no subsystem is
  unobservable.

## 4. Architecture skeleton

```
┌─────────────────┐  ┌───────────┐
│  Web-UI         │  │  CLI      │          thin clients
│  (static server │  └─────┬─────┘
│   in image)     │        │
└────────┬────────┘        │
         └───────┬─────────┘
                 │ HTTP + RBAC admission
          ┌──────▼───────────────┐
          │       API            │   contracts/ (OpenAPI)
          └──────┬───────────────┘
                 │ desired-state CRUD (declarative)
          ┌──────▼───────────────────────────────┐
          │  core daemon                          │
          │   spec store (source of truth)        │   persisted on a
          │   └─ reconciler loop (wanted≠actual)  │   configured dataset
          │        ├─ ZFS controller      │ zpool/zfs (go-zfs)
          │        ├─ SMB controller      │ samba + smbpasswd
          │        ├─ NFS controller      │ zfs set sharenfs (ADR-0009)
          │        ├─ NVMe-oF controller  │ configfs (nvmet)
          │        ├─ Apps controller     │ nerdctl compose → containerd
          │        ├─ Identity controller │ user DB ↔ UID ↔ samba
          │        └─ Update controller   │ A/B slot swap + reboot
          └──────────────────────────────────────┘
```

## 5. Key flows

### 5.1 Identity linkage
```
NAS user (API/UI/CLI)
   │  share granted
   ├──► system account + UID   ──► POSIX owner on the dataset
   └──► samba passdb entry     ──► SMB auth
```

### 5.2 A/B update
```
[slot A active] ─ deploy to slot B ─▶ [B staged] ─▶ set bootenv, reboot
   ▲                                                   │
   └──────── manual rollback / boot-fail ──────────────┘
```

### 5.3 Configurable layout
```
installer → assign roles:
   system dataset   → configs, spec store
   app dataset      → images, compose state
   data pools       → shares, zvols (user-chosen)
```

## 6. Open questions (deferred to the architecture phase)

These are deliberately not resolved here; they refine the design but do not change
the feature map or the decisions above. They will be resolved in per-subsystem
changes as the architecture is detailed.

- Spec store format and location (files vs database; which dataset).
- Reconcile loop granularity (per-controller loops vs a single global pass).
- compose → containerd translation approach (nerdctl compose wrapper vs own engine).
- RBAC capability taxonomy (which roles and capabilities).
- Update image format and slot/bootloader specifics.
- API resource model / OpenAPI shape.

> Resolved by the `scope-missing-plane` change (no longer open):
> - NFS share management → ZFS `sharenfs` properties, not `/etc/exports`
>   (ADR-0009, §3.2 consequence of ADR-0001).
> - Compose volume backing (dataset vs zvol) → ZFS dataset per volume by default,
>   zvols as an explicit escape hatch (ADR-0004).
> - Rollback sourcing → composite rollback: spec-store revert + ZFS snapshot
>   rollback (ADR-0002).
