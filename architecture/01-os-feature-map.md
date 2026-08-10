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
│   │  declarative desired-state → core daemon reconciles       │
└──────────────────────┬───────────────────────────────────────┘
┌──────────────────────▼───────────────────────────────────────┐
│  DATA PLANE  (host features)                                  │
│   ZFS (data only) ─► pools · datasets · snapshots · zvols    │
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
| 2 | ZFS storage | Pools, datasets/properties, snapshots, zvols (data-only) |
| 3 | SMB + NFS shares | Share a dataset over the network; share auth via linked users |
| 4 | NVMe-oF shares | nvmet kernel target exporting zvols as block devices to remote hosts |
| 5 | App workloads | Full docker-compose on containerd; ZFS-backed volumes; images on a configured dataset |
| 6 | API + CLI | First-class HTTP API (OpenAPI); thin CLI client |
| 7 | RBAC + users | NAS user DB; roles per capability; optional link to system/SMB users |
| 8 | Web-UI | Management console over the API; no direct host access |
| 9 | Observability | Prometheus metrics for every subsystem |

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
  and CLI become thin clients; drift detection and rollback fall out of the model.
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
  ZFS-backed (dataset/zvol per volume).
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
┌───────────┐  ┌───────────┐
│  Web-UI   │  │  CLI      │          thin clients
└─────┬─────┘  └─────┬─────┘
      └──────┬───────┘
             │ HTTPS + RBAC admission
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
      │        ├─ NFS controller      │ /etc/exports
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
