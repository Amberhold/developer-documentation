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
│   · OS-disk LUKS2 encryption, unlocked at boot (ADR-0011)     │
└───────────────────────────────────────────────────────────────┘
```

| # | Feature | Minimal scope |
|---|---------|---------------|
| 0 | OS image | Debian base, read-only squashfs root, A/B dual-slot, writable state off-root |
| 1 | System updates | Product feature: trigger, progress, reboot, rollback via UI/API |
| 2 | ZFS storage | Pools, datasets/properties, snapshots (scheduled with retention, shared `Schedule` resource, ADR-0022), zvols (data-only), native encryption (keyfile on OS-disk partition, ADR-0011 — keyfiles now live inside the OS-disk LUKS container and are protected at rest by external unlock factors (YubiKey FIDO2 / USB keyfile / recovery passphrase, ADR-0011); OS-disk loss still loses the keyfile, so encrypted datasets remain unrecoverable), dataset quotas (`quota`/`refquota`; user/group quotas deferred) |
| 3 | SMB + NFS shares | Share a dataset over the network; share auth via linked users |
| 4 | NVMe-oF shares | nvmet kernel target exporting zvols as block devices to remote hosts |
| 5 | App workloads | Full docker-compose on containerd (opt-in); ZFS-backed volumes; images on a configured (opt-in) dataset |
| 6 | API + CLI | First-class HTTP API (OpenAPI); thin CLI client; auth via sessions + API tokens (ADR-0020), with OIDC as an additional sign-in option for Web-UI sessions (ADR-0029) |
| 7 | RBAC + users | NAS user DB; roles per capability; optional link to system/SMB users; Argon2id password hashing (ADR-0020); federated principals get roles from IdP `groups` claims while NAS-local principals keep DB-stored roles (ADR-0029) |
| 8 | Web-UI | Management console over the API; no direct host access |
| 9 | Observability | OpenTelemetry-instrumented metrics (default Prometheus export) for every subsystem; optional OTLP export of metrics/traces/logs; reconcile-loop tracing (off by default); direct OTEL log export with journald as the durable store |
| 10 | Networking | Two planes, roles assigned at install (ADR-0014): management plane carries API/UI + DNS/hostname/NTP with a per-interface IP (DHCP or static); data plane carries SMB/NFS/NVMe-oF share traffic with per-interface IP at install. The management plane serves API/UI over HTTPS in v1 (ADR-0028); v1 default: one management LAN for everything |
| 11 | Pool storage | Disk inventory; pool/vdev creation; disk replacement |
| 12 | Installer | First-boot: assign storage layout roles (data + optional app, decision 7); import existing pools with role reassignment; admin bootstrap (admin user + password set at install, ADR-0020); assign network plane roles + per-interface IP (ADR-0014); OS-disk sizing check for slots + spec store + config/var, 128–256 GB floor (ADR-0011); create the OS-disk LUKS2 container and enroll initial unlock factors (USB keyfile + YubiKey FIDO2 + recovery passphrase) and the unlock policy before first boot (ADR-0011) |
| 13 | Logging + audit | System logs via journald, forwarded to the OS-disk `config/var` partition; audit trail of admin actions (tagged journald entries), rotation + retention (ADR-0013). Core components log directly through the OTEL pipeline with journald as a parallel durable exporter (ADR-0008); third-party daemon logs (samba, containerd, kernel, sshd) remain journald-only |
| 14 | Disk health | Scrub schedule (shared `Schedule` resource, ADR-0022); SMART monitoring via `core` smartctl polling — per-disk status + Prometheus gauges, thresholds spec-declared (ADR-0021) |
| 15 | Off-site backup | restic backup of snapshots of opted-in datasets to a remote repository; event-driven per-snapshot ingestion (restic dedup, no new `Schedule` consumer, ADR-0030); per-dataset opt-in via `amberhold:backup` ZFS user property (app-images excluded); dataset sources mounted read-only, zvols via `zfs send` stream; password co-located on the OS-disk spec-store partition, auto-loaded (ADR-0011 pattern); recovery boundary = data-pool loss only, D1 posture unchanged; restic pinned in the image (ADR-0001/0006 pattern) |

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

Each decision below is a short index entry. The full decision record — context,
alternatives considered, and consequences — lives in the ADR it points to (the
decision-of-record). This section exists to navigate, not to re-derive.

| Decision | In a nutshell | Decision-of-record |
|----------|---------------|--------------------|
| OS image | Immutable squashfs root; ZFS reserved for data pools; A/B dual-slot boot | [ADR-0001](adr/0001-read-only-squashfs-root-ab-boot.md) |
| Control plane | Declarative desired-state API reconciled by the core daemon; action endpoints for imperative ops | [ADR-0002](adr/0002-declarative-desired-state-api-reconciler-core.md) |
| NVMe-oF | Kernel `nvmet` target, serve-only; we export zvols, never consume remote | [ADR-0003](adr/0003-nvmet-kernel-nvme-of-target-serve-only.md) |
| App workloads | Full docker-compose on containerd; ZFS-backed volumes (dataset per volume) | [ADR-0004](adr/0004-zfs-backed-container-volumes-full-docker-compose.md) |
| Identity | NAS-local user DB; optional system/SMB link materialized where ownership is needed | [ADR-0005](adr/0005-identity-model-separate-nas-db.md) |
| Updates | A/B is the mechanism; updates (trigger, progress, reboot, rollback) are a product feature | [ADR-0006](adr/0006-updates-as-a-product-feature.md) |
| Storage layout | Data + optional app roles assigned at install; system config state fixed on the OS disk | [ADR-0007](adr/0007-configurable-storage-layout-assigned-at-install.md) |
| Observability | OpenTelemetry for all signals; Prometheus always-on; one `Telemetry` resource; tracing off by default; journald durable store | [ADR-0008](adr/0008-observability-otel.md) |
| OS disk (layout + encryption) | Dedicated OS disk; LUKS2-sealed slots + spec store + keyfiles + `config/var`; external-factor boot unlock; data keys inside | [ADR-0011](adr/0011-os-disk-layout-encryption.md) |
| OS disk (writable state) | Spec store + `config/var` on OS-disk partitions; journald logging/audit; versioned spec store | [ADR-0013](adr/0013-os-disk-writable-state.md) |
| Networking | Two planes, roles + per-interface IP assigned at install | [ADR-0014](adr/0014-network-planes-configurable-at-install.md) |
| Reconcile | Per-controller reconcile loops over a shared event source | [ADR-0017](adr/0017-per-controller-reconcile-loops.md) |
| Core daemon | Framework-first controller runtime (D1–D10) + startup sequence + action routing | [ADR-0031](adr/0031-core-daemon-controller-runtime-and-startup-sequence.md) |
| RBAC | Fixed roles with a capability map, enforced at API admission | [ADR-0018](adr/0018-rbac-fixed-roles-capability-map.md) |
| API model | Feature-map-aligned API resource model; shared `Schedule` resource | [ADR-0019](adr/0019-feature-map-aligned-api-resource-model.md) |
| Authentication | Sessions for UI + API tokens for CLI; Argon2id | [ADR-0020](adr/0020-authentication-session-token-argon2.md) |
| Disk health | Core polls SMART via smartctl; status + Prometheus gauges; scrub via `Schedule` | [ADR-0021](adr/0021-disk-health-smart-scrub-schedule.md) |
| Scheduling | Shared `Schedule` resource for snapshots and scrubs | [ADR-0022](adr/0022-shared-schedule-resource.md) |
| SMB | samba config on `config/var`; per-user access control | [ADR-0023](adr/0023-smb-share-management-mechanism.md) |
| Pool topology | Single pool; mirror/raidz1/raidz2; spares; disk replacement flow | [ADR-0024](adr/0024-pool-vdev-topology-and-disk-replacement.md) |
| TLS | HTTPS on the management plane; built-in CA / ACME / manual-import trust | [ADR-0028](adr/0028-tls-management-plane.md) |
| OIDC | OIDC sign-in for Web-UI sessions; claims-as-roles; JIT NAS account | [ADR-0029](adr/0029-oidc-authentication.md) |
| Off-site backup | restic archive of opted-in snapshots; event-driven per-snapshot ingestion | [ADR-0030](adr/0030-offsite-backup-zfs-snapshots-restic.md) |

### 3.9 Cross-cutting relationships

The feature map (above) and the skeleton (below) reference ADRs in place. Two
consolidations are worth noting: observability (metrics, tracing, logs, and the
`Telemetry` resource) is a single decision ([ADR-0008](adr/0008-observability-otel.md)),
and the OS disk is a single story — physical layout + encryption
([ADR-0011](adr/0011-os-disk-layout-encryption.md)) plus the writable state that
lives on it ([ADR-0013](adr/0013-os-disk-writable-state.md)).

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
            │       API            │   contracts/ (openapi/v1.yaml +
            │                       │   metrics/catalog.yaml, ADR-0019)
           └──────┬───────────────┘
                  │ desired-state CRUD + action endpoints (declarative + ops)
            ┌──────▼───────────────────────────────┐
            │  core daemon                          │
             │   spec store (source of truth)        │   on the OS disk, not a
             │   └─ reconciler loop (wanted≠actual)  │   pool (ADR-0013);
             │        ├─ Storage ctlr     │ zpool/zfs/smartctl  per-controller
             │        │                   │   shell-outs (ADR-0024)
             │        ├─ FileShare ctlr   │ SMB backend → samba+tdbsam (ADR-0023)  loops over a
             │        │                   │ NFS backend → zfs set sharenfs (ADR-0009) shared event
             │        ├─ NVMe-oF controller  │ configfs (nvmet, NQN allowlist) source
             │        ├─ Apps controller     │ nerdctl compose → containerd  (ADR-0017)
             │        ├─ Identity service  │ UID ledger → spec store + tdbsam (app UIDs too)
             │        ├─ Disk-health ctlr    │ smartctl + scrub via Schedule (ADR-0021)
             │        ├─ Backup controller  │ restic shell-out, per-snapshot ingestion (ADR-0030)
             │        └─ Update controller   │ A/B slot swap + reboot
            └──────────────────────────────────────┘
    framework-first controller runtime: D1–D10 mechanics, startup sequence,
    action routing → docs/architecture/02-core-daemon.md (ADR-0031)
    storage data-plane anchor: host facade + Disk/Pool controllers →
    docs/architecture/03-storage-controller.md (ADR-0024, ADR-0021, ADR-0011)
    shares slice: one FileShare controller + SMB/NFS mechanism backends →
    docs/architecture/04-shares-controller.md (ADR-0023, ADR-0009, ADR-0005)
```

## 5. Key flows

### 5.1 Identity linkage
```
NAS user (API/UI/CLI)
   │  share granted
   ├──► allocated UID (identity service) ──► POSIX owner on the dataset
   └──► samba tdbsam entry          ──► SMB auth
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
                          + keyfiles + config/var (ADR-0011)
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
  desired-state CRUD + status, declared in `contracts/openapi/v1.yaml` with the
  metric catalog in `contracts/metrics/catalog.yaml` (ADR-0019).
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
> - Encryption keyfiles on the OS-disk spec store partition, headless unlock,
>   now inside the OS-disk LUKS container and protected at rest by external
>   unlock factors (ADR-0011).
> - Logging + audit via journald, forwarded to the OS-disk `config/var` partition,
>   rotation + retention, no snapshots (ADR-0013).
> - NVMe-oF NQN allowlist per export (ADR-0003).
> - NFSv4 `idmapd` for UID mapping (ADR-0005).
> - Update images signed, verified pre-boot (ADR-0006).
> - `config` role removed: system config state lives on OS-disk partitions; only
>   `data` + optional `app` roles remain (ADR-0007).
> - OS-disk layout: ESP / slots A+B / spec store + keyfiles / `config/var`; sizing
>   check at install with a 128–256 GB floor (ADR-0011).
> - Whole-OS-disk LUKS2 encryption unlocked at boot by external factors
>   (YubiKey FIDO2 / USB keyfile / recovery passphrase) with a per-mode unlock
>   policy; ESP stays plain with per-slot kernel+initramfs (ADR-0011).
> - App identity via dedicated UID per stack (ADR-0004, ADR-0005).
> - Imperative ops as action endpoints within the declarative model (ADR-0002).
> - RO-root writable-config convention: generated files on `config/var` partition,
>   `/etc` symlinks (ADR-0001).
> - Management-plane TLS with built-in-CA / ACME / manual-import trust modes,
>   reversing the deferred plain-HTTP baseline (ADR-0028).
> - OIDC authentication for the Web-UI: single IdP, authorization-code + PKCE,
>   claims-as-roles for federated principals, JIT NAS account (ADR-0029).
