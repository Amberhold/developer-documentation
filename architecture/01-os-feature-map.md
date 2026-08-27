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
│   · OS-disk LUKS2 encryption, unlocked at boot (ADR-0031)     │
└───────────────────────────────────────────────────────────────┘
```

| # | Feature | Minimal scope |
|---|---------|---------------|
| 0 | OS image | Debian base, read-only squashfs root, A/B dual-slot, writable state off-root |
| 1 | System updates | Product feature: trigger, progress, reboot, rollback via UI/API |
| 2 | ZFS storage | Pools, datasets/properties, snapshots (scheduled with retention, shared `Schedule` resource, ADR-0022), zvols (data-only), native encryption (keyfile on OS-disk partition, ADR-0015 — keyfiles now live inside the OS-disk LUKS container and are protected at rest by external unlock factors (YubiKey FIDO2 / USB keyfile / recovery passphrase, ADR-0031); OS-disk loss still loses the keyfile, so encrypted datasets remain unrecoverable), dataset quotas (`quota`/`refquota`; user/group quotas deferred) |
| 3 | SMB + NFS shares | Share a dataset over the network; share auth via linked users |
| 4 | NVMe-oF shares | nvmet kernel target exporting zvols as block devices to remote hosts |
| 5 | App workloads | Full docker-compose on containerd (opt-in); ZFS-backed volumes; images on a configured (opt-in) dataset |
| 6 | API + CLI | First-class HTTP API (OpenAPI); thin CLI client; auth via sessions + API tokens (ADR-0020), with OIDC as an additional sign-in option for Web-UI sessions (ADR-0029) |
| 7 | RBAC + users | NAS user DB; roles per capability; optional link to system/SMB users; Argon2id password hashing (ADR-0020); federated principals get roles from IdP `groups` claims while NAS-local principals keep DB-stored roles (ADR-0029) |
| 8 | Web-UI | Management console over the API; no direct host access |
| 9 | Observability | OpenTelemetry-instrumented metrics (default Prometheus export) for every subsystem; optional OTLP export of metrics/traces/logs; reconcile-loop tracing (off by default); direct OTEL log export with journald as the durable store |
| 10 | Networking | Two planes, roles assigned at install (ADR-0014): management plane carries API/UI + DNS/hostname/NTP with a per-interface IP (DHCP or static); data plane carries SMB/NFS/NVMe-oF share traffic with per-interface IP at install. The management plane serves API/UI over HTTPS in v1 (ADR-0028); v1 default: one management LAN for everything |
| 11 | Pool storage | Disk inventory; pool/vdev creation; disk replacement |
| 12 | Installer | First-boot: assign storage layout roles (data + optional app, decision 7); import existing pools with role reassignment; admin bootstrap (admin user + password set at install, ADR-0020); assign network plane roles + per-interface IP (ADR-0014); OS-disk sizing check for slots + spec store + config/var, 128–256 GB floor (ADR-0011); create the OS-disk LUKS2 container and enroll initial unlock factors (USB keyfile + YubiKey FIDO2 + recovery passphrase) and the unlock policy before first boot (ADR-0031) |
| 13 | Logging + audit | System logs via journald, forwarded to the OS-disk `config/var` partition; audit trail of admin actions (tagged journald entries), rotation + retention (ADR-0016). Core components log directly through the OTEL pipeline with journald as a parallel durable exporter (ADR-0027); third-party daemon logs (samba, containerd, kernel, sshd) remain journald-only |
| 14 | Disk health | Scrub schedule (shared `Schedule` resource, ADR-0022); SMART monitoring via `core` smartctl polling — per-disk status + Prometheus gauges, thresholds spec-declared (ADR-0021) |
| 15 | Off-site backup | restic backup of snapshots of opted-in datasets to a remote repository; event-driven per-snapshot ingestion (restic dedup, no new `Schedule` consumer, ADR-0030); per-dataset opt-in via `amberhold:backup` ZFS user property (app-images excluded); dataset sources mounted read-only, zvols via `zfs send` stream; password co-located on the OS-disk spec-store partition, auto-loaded (ADR-0015 pattern); recovery boundary = data-pool loss only, D1 posture unchanged; restic pinned in the image (ADR-0001/0006 pattern) |

### Deferred areas (future paths, out of v1 scope)

These are deliberately not v1 features. They are recorded here so they read as
explicit decisions (design decision D6 in the `scope-missing-plane` change), not
silent gaps:

- **Replication** — `zfs send/receive` to a remote ZFS host is out of scope for v1.
  ZFS snapshots (in scope, feature 2) are the primitive a future replication
  feature builds on. Backups are now feature 15 (off-site restic archive,
  ADR-0030), distinct from replication: restic is file/stream-level, replication
  would be block-level to a remote ZFS sink.
- **Firewall** — external network exposure hardening is out of scope for v1; the
  management network binds per feature 10. The management plane itself serves
  HTTPS in v1 (ADR-0028).
- **Alerting** — no push-based alerting (email/webhook) in v1; status is surfaced
  via API/UI and Prometheus metrics (feature 9).
- **AD/LDAP join** — no directory-service integration in v1; identity is the
  NAS-local user DB (ADR-0005). OIDC (ADR-0029) is an identity *provider* for the
  Web-UI, distinct from directory sync — this deferral stands.
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
  spec (ADR-0002).

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
  dedicated identity-controller UID that owns its dataset (ADR-0004, ADR-0005).

### 3.5 Identity: separate NAS DB with optional link
- **Decision**: RBAC identity is a NAS-local user DB (API/UI/CLI). A separate,
  optional link materializes only where POSIX/SMB ownership is required — and is
  auto-created when a share grants access.
- **Alternatives considered**: pure system accounts (couples RBAC to host accounts);
  fully separate (share auth disconnected from users).
- **Implications**: "create a share user" ⇒ NAS user + optional system UID + optional
  samba passdb entry. UIDs are also allocated per app stack (ADR-0004, ADR-0005).

### 3.6 Updates: a product feature, not just image mechanics
- **Decision**: A/B is the mechanism; system updates (trigger, progress, reboot,
  rollback) are exposed through the UI/API.
- **Implications**: an update controller with a reboot in the middle of the flow;
  progress reporting; boot-fail and manual rollback paths.

### 3.7 Storage layout: data (+ optional app) configurable at install, system state fixed on the OS disk
- **Decision**: the roles of datasets (`app/images` when the apps feature is in
  use, and `data`) are assigned at install time, not fixed. System config state
  is *not* a pool role: the spec store + keyfiles and the `config/var` partition
  (logs/audit, generated daemon config fragments) live on the OS disk
  (ADR-0011, ADR-0013, ADR-0016). With host encryption (ADR-0031), the OS disk
  is a single LUKS2 container over slots + spec store + keyfiles + `config/var`,
  unlocked at boot by an external factor; only the ESP stays plain.
- **Implications**: an install-time assignment step for the data (and optional
  app) roles; everything else points at the assigned datasets. Pools are
  data-only; encryption keyfiles live on the OS disk inside the LUKS container
  (ADR-0015, ADR-0031).

### 3.8 Metrics: every subsystem observable
- **Decision**: OpenTelemetry SDK is the instrumentation layer; Prometheus is the
  always-on default exporter serving `/metrics` (ADR-0008, amended). Metric names
  move to the `amberhold.*` catalog with a documented Prometheus translation,
  declared in `contracts` and enforced by OTEL Views. OTLP export is additive per
  configured target; a singleton `Telemetry` resource (ADR-0025) carries export,
  sampling, and log-level configuration. Reconcile passes are traced with a root
  `reconcile` span and per-controller children, off by default (ADR-0026). Core
  logs flow through the OTEL pipeline with journald as the durable store
  (ADR-0027).
- **Implication**: each controller exposes counters/gauges/histograms via its own
  OTEL meter; no subsystem is unobservable. Traces and off-box log forwarding
  require a configured OTLP target; `/metrics` and journald never depend on one.
- **Cross-references**: feature 9 (Observability), feature 13 (Logging + audit),
  ADR-0016 (journald), ADR-0025 (telemetry resource), ADR-0026 (tracing),
  ADR-0027 (log export).

## 4. Architecture skeleton

```
┌─────────────────┐  ┌───────────┐
│  Web-UI         │  │  CLI      │          thin clients
│  (static server │  └─────┬─────┘
│   in image)     │        │
└────────┬────────┘        │
          └───────┬─────────┘
                  │ HTTPS + auth (sessions/tokens) + RBAC admission
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
            │        ├─ Backup controller  │ restic shell-out, per-snapshot ingestion (ADR-0030)
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
   OS disk (fixed)      → ESP (plain) + LUKS2 container: slots + spec store
                          + keyfiles + config/var (ADR-0011, ADR-0031)
```
```

## 6. Open questions (resolved in the architecture phase)

These were open in the discovery phase and are now resolved by ADRs 0013–0022,
with the update-image bullet also drawing on the earlier ADR-0001/0006. They
refine the design but do not change the feature map or the earlier decisions.

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
  (ADR-0001, ADR-0006, ADR-0011).
- API resource model / OpenAPI shape → feature-map-aligned resources with
  desired-state CRUD + status, declared in `contracts/` (ADR-0019).
- Authentication → sessions for the UI + API tokens for CLI/automation, Argon2id
  hashing, enforced at admission (ADR-0020); OIDC sign-in for Web-UI sessions adds
  a federated path with roles from IdP claims and TLS as a prerequisite
  (ADR-0028, ADR-0029).
- Disk health → core polls SMART via smartctl; status + Prometheus gauges,
  thresholds spec-declared (ADR-0021).
- Snapshot/scrub scheduling → one shared `Schedule` resource (ADR-0022).
- Off-site backup → feature 15: restic archive of snapshots of opted-in
  datasets; the backup story spans ADR-0022 (snapshot primitive) and ADR-0030
  (off-site archive).
- Spec-store schema evolution → versioned store with forward migration, rollback
  via snapshot restore (ADR-0013).

> Resolved by the `scope-missing-plane` change (no longer open):
> - NFS share management → ZFS `sharenfs` properties, not `/etc/exports`
>   (ADR-0009, consequence of ADR-0001).
> - Compose volume backing (dataset vs zvol) → ZFS dataset per volume by default,
>   zvols as an explicit escape hatch (ADR-0004).
> - Rollback sourcing → composite rollback: spec-store revert + ZFS snapshot
>   rollback (ADR-0002).

> Additional decisions captured during the architecture phase (see ADRs):
> - Network planes configurable at install, with per-interface IP (ADR-0014).
> - Encryption keyfiles on the OS-disk spec store partition, headless unlock;
>   OS-disk partitions are plain filesystems (ADR-0015, ADR-0011).
> - Logging + audit via journald, forwarded to the OS-disk `config/var` partition,
>   rotation + retention, no snapshots (ADR-0016, ADR-0011).
> - NVMe-oF NQN allowlist per export (ADR-0003).
> - NFSv4 `idmapd` for UID mapping (ADR-0005).
> - Update images signed, verified pre-boot (ADR-0006).
> - `config` role removed: system config state lives on OS-disk partitions; only
>   `data` + optional `app` roles remain (ADR-0007).
> - OS-disk layout: ESP / slots A+B / spec store + keyfiles / `config/var`; sizing
>   check at install with a 128–256 GB floor (ADR-0011).
> - Whole-OS-disk LUKS2 encryption unlocked at boot by external factors
>   (YubiKey FIDO2 / USB keyfile / recovery passphrase) with a per-mode unlock
>   policy; ESP stays plain with per-slot kernel+initramfs (ADR-0031).
> - App identity via dedicated UID per stack (ADR-0004, ADR-0005).
> - Imperative ops as action endpoints within the declarative model (ADR-0002).
> - RO-root writable-config convention: generated files on `config/var` partition,
>   `/etc` symlinks (ADR-0001).
> - Management-plane TLS with built-in-CA / ACME / manual-import trust modes,
>   reversing the deferred plain-HTTP baseline (ADR-0028).
> - OIDC authentication for the Web-UI: single IdP, authorization-code + PKCE,
>   claims-as-roles for federated principals, JIT NAS account (ADR-0029).
