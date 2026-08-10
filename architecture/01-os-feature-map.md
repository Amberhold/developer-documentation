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
| 2 | ZFS storage | Pools, datasets/properties, snapshots (scheduled with retention, shared `Schedule` resource, ADR-0022), zvols (data-only), native encryption (keyfile on OS-disk partition, ADR-0015), dataset quotas (`quota`/`refquota`; user/group quotas deferred) |
| 3 | SMB + NFS shares | Share a dataset over the network; share auth via linked users |
| 4 | NVMe-oF shares | nvmet kernel target exporting zvols as block devices to remote hosts |
| 5 | App workloads | Full docker-compose on containerd (opt-in); ZFS-backed volumes; images on a configured (opt-in) dataset |
| 6 | API + CLI | First-class HTTP API (OpenAPI); thin CLI client; auth via sessions + API tokens (ADR-0020) |
| 7 | RBAC + users | NAS user DB; roles per capability; optional link to system/SMB users; Argon2id password hashing (ADR-0020) |
| 8 | Web-UI | Management console over the API; no direct host access |
| 9 | Observability | Prometheus metrics for every subsystem |
| 10 | Networking | Management network: management IP (DHCP or static), hostname, NTP; where API/UI/shares bind. Interface roles (mgmt vs data plane) configurable at install (ADR-0014) |
| 11 | Pool storage | Disk inventory; pool/vdev creation; disk replacement |
| 12 | Installer | First-boot: assign storage layout roles (data + optional app, decision 7); import existing pools with role reassignment; admin bootstrap (admin user + password set at install, ADR-0020); assign network plane roles + per-interface IP (ADR-0014); OS-disk sizing check for slots + spec store + config/var, 128–256 GB floor (ADR-0011) |
| 13 | Logging + audit | System logs via journald, forwarded to the OS-disk `config/var` partition; audit trail of admin actions (tagged journald entries), rotation + retention (ADR-0016) |
| 14 | Disk health | Scrub schedule (shared `Schedule` resource, ADR-0022); SMART monitoring via `core` smartctl polling — per-disk status + Prometheus gauges, thresholds spec-declared (ADR-0021) |

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
- **User/group quotas** — dataset quotas are in v1 (feature 2); `userquota`/
  `groupquota` are deferred with no user-identity pressure.

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
  - Writable state (spec store, keyfiles, generated configs, logs/audit, app
    images, user data) must live on the OS-disk partitions (ADR-0011) or
    configured app/data datasets (ADR-0007), never on `/`.
  - A/B = two slots; bootloader selects active slot; rollback = fall to the other.

### 3.2 Control plane: declarative desired-state API
- **Decision**: the API is declarative desired-state, reconciled by the core daemon.
- **Alternatives considered**: imperative actions (simpler now, but drift/rollback are
  hard); hybrid.
- **Rationale**: idempotent, auditable, natural fit for a "system of record" NAS; UI
  and CLI become thin clients; drift detection falls out of the model and rollback is
  composite (spec-store revert + ZFS snapshot rollback, ADR-0002).
- **Implications**: core daemon is a reconciler; spec store is the source of truth;
  RBAC is enforced at API admission, not in the UI. Imperative operations (disk
  replace, scrub now, rollback triggers) are action endpoints that do not mutate
  spec (ADR-0002 amendment).

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
  data participates in the snapshot/replication story. Each stack runs as a
  dedicated identity-controller UID that owns its dataset (ADR-0004/0005
  amendment).

### 3.5 Identity: separate NAS DB with optional link
- **Decision**: RBAC identity is a NAS-local user DB (API/UI/CLI). A separate,
  optional link materializes only where POSIX/SMB ownership is required — and is
  auto-created when a share grants access.
- **Alternatives considered**: pure system accounts (couples RBAC to host accounts);
  fully separate (share auth disconnected from users).
- **Implications**: "create a share user" ⇒ NAS user + optional system UID + optional
  samba passdb entry. UIDs are also allocated per app stack (ADR-0004/0005
  amendment).

### 3.6 Updates: a product feature, not just image mechanics
- **Decision**: A/B is the mechanism; system updates (trigger, progress, reboot,
  rollback) are exposed through the UI/API.
- **Implications**: an update controller with a reboot in the middle of the flow;
  progress reporting; boot-fail and manual rollback paths.

### 3.7 Storage layout: data (+ optional app) configurable at install, system state fixed on the OS disk
- **Decision**: the roles of datasets (`app/images` when the apps feature is in
  use, and `data`) are assigned at install time, not fixed. System config state
  is *not* a pool role: the spec store + keyfiles and the `config/var` partition
  (logs/audit, generated daemon config fragments) live on the OS disk as plain
  partitions (ADR-0011, ADR-0013, ADR-0016).
- **Implications**: an install-time assignment step for the data (and optional
  app) roles; everything else points at the assigned datasets. Pools are
  data-only; encryption keyfiles live on the OS disk (ADR-0015).

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
                  │ HTTP + auth (sessions/tokens) + RBAC admission
           ┌──────▼───────────────┐
           │       API            │   contracts/ (OpenAPI)
           └──────┬───────────────┘
                  │ desired-state CRUD + action endpoints (declarative + ops)
            ┌──────▼───────────────────────────────┐
            │  core daemon                          │
            │   spec store (source of truth)        │   on the OS disk, not a
            │   └─ reconciler loop (wanted≠actual)  │   pool (ADR-0013);
            │        ├─ ZFS controller      │ zpool/zfs (go-zfs)    per-controller
            │        ├─ SMB controller      │ samba + smbpasswd     loops over a
            │        ├─ NFS controller      │ zfs set sharenfs (ADR-0009) shared event
            │        ├─ NVMe-oF controller  │ configfs (nvmet, NQN allowlist) source
            │        ├─ Apps controller     │ nerdctl compose → containerd  (ADR-0017)
            │        ├─ Identity controller │ user DB ↔ UID ↔ samba (app UIDs too)
            │        ├─ Disk-health ctlr    │ smartctl + scrub via Schedule (ADR-0021)
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
   app dataset (opt-in) → images, compose state
   data pools           → shares, zvols, app volume datasets (user-chosen)
   OS disk (fixed)      → slots + spec store + keyfiles + config/var (ADR-0011)
```
```

## 6. Open questions (resolved in the architecture phase)

These were open in the discovery phase and are now resolved by ADRs 0013–0022.
They refine the design but do not change the feature map or the earlier decisions.

- Spec store format and location → persistent OS-disk partition, versioned/snapshotted
  (ADR-0013).
- Reconcile loop granularity → per-controller loops over a shared event source
  (ADR-0017).
- compose → containerd translation approach → nerdctl compose wrapper, version pinned
  in the image (ADR-0004 consequence).
- RBAC capability taxonomy → fixed roles with a capability map: admin, storage-admin,
  share-admin, app-admin, auditor, read-only (ADR-0018).
- Update image format and slot/bootloader specifics → standard A/B tooling (rauc /
  ostree / ABRoot candidates), signed images with a baked-in trust anchor
  (ADR-0001 amendment, ADR-0006 amendment, ADR-0013).
- API resource model / OpenAPI shape → feature-map-aligned resources with
  desired-state CRUD + status, declared in `contracts/` (ADR-0019).
- Authentication → sessions for the UI + API tokens for CLI/automation, Argon2id
  hashing, enforced at admission (ADR-0020).
- Disk health → core polls SMART via smartctl; status + Prometheus gauges,
  thresholds spec-declared (ADR-0021).
- Snapshot/scrub scheduling → one shared `Schedule` resource (ADR-0022).
- Spec-store schema evolution → versioned store with forward migration, rollback
  via snapshot restore (ADR-0013).

> Resolved by the `scope-missing-plane` change (no longer open):
> - NFS share management → ZFS `sharenfs` properties, not `/etc/exports`
>   (ADR-0009, §3.2 consequence of ADR-0001).
> - Compose volume backing (dataset vs zvol) → ZFS dataset per volume by default,
>   zvols as an explicit escape hatch (ADR-0004).
> - Rollback sourcing → composite rollback: spec-store revert + ZFS snapshot
>   rollback (ADR-0002).

> Additional decisions captured during the architecture phase (see ADRs):
> - Network planes configurable at install, with per-interface IP (ADR-0014).
> - Encryption keyfiles on the OS-disk spec store partition, headless unlock;
>   OS-disk partitions are plain filesystems (ADR-0015 amendment, ADR-0011).
> - Logging + audit via journald, forwarded to the OS-disk `config/var` partition,
>   rotation + retention, no snapshots (ADR-0016, ADR-0011).
> - NVMe-oF NQN allowlist per export (ADR-0003 amendment).
> - NFSv4 `idmapd` for UID mapping (ADR-0005 amendment).
> - Update images signed, verified pre-boot (ADR-0006 amendment).
> - `config` role removed: system config state lives on OS-disk partitions; only
>   `data` + optional `app` roles remain (ADR-0007 amendment).
> - OS-disk layout: ESP / slots A+B / spec store + keyfiles / `config/var`; sizing
>   check at install with a 128–256 GB floor (ADR-0011 amendment).
> - App identity via dedicated UID per stack (ADR-0004/0005 amendment).
> - Imperative ops as action endpoints within the declarative model (ADR-0002
>   amendment).
> - RO-root writable-config convention: generated files on `config/var` partition,
>   `/etc` symlinks (ADR-0001 amendment).
