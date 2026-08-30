<!-- Source brand asset: copy of logo/logo_with_text.png from the ai-harness meta-repo. Re-copy whenever the kit changes. -->

![Amberhold](images/logo-with-text.png)

# Amberhold

*A NAS operating system for self-hosted storage appliances — every capability managed through a web UI and a first-class API.*

Amberhold is a NAS operating system that puts the whole appliance behind one management surface. NVMe-oF volumes, ZFS pools, container workloads, identity and access control, and observability are all driven declaratively through a web UI and a contract-first API — no SSH required for day-to-day operations.

## Capabilities

- **NVMe-oF volumes** — high-performance block storage over NVMe over Fabrics
- **ZFS** — pooled storage with snapshots, replication, encryption, and compression
- **containerd** — run apps as Docker Compose workloads on the appliance
- **RBAC** — fine-grained roles and permissions for users and groups
- **Web UI + API** — a full management surface, contract-first
- **Prometheus metrics** — built-in observability endpoint for all subsystems

## Documentation map

We are in a discovery phase: the system is designed here, in this documentation, before any code is written. All design and architecture work lives in the `docs` repo, captured as architecture docs and architecture decision records (ADRs).

### Architecture

- [Feature map & architecture](architecture/01-os-feature-map.md) — target feature map and architecture decisions (discovery-phase capture).
- [Core daemon](architecture/02-core-daemon.md) — framework-first controller runtime (D1–D10), startup sequence, action routing (ADR-0031).
- [Storage controller](architecture/03-storage-controller.md) — the data-plane anchor: host facade, Disk/Pool controllers, scrub action (ADR-0024, ADR-0021, ADR-0011).

### Architecture decision records

- [ADR-0001](adr/0001-read-only-squashfs-root-ab-boot.md) — read-only squashfs root with A/B boot (`core` in the image, no host-file writes under `/`, RO-root writable-config convention, standard A/B tooling)
- [ADR-0002](adr/0002-declarative-desired-state-api-reconciler-core.md) — declarative desired-state API with reconciler core (composite rollback; imperative action endpoints)
- [ADR-0003](adr/0003-nvmet-kernel-nvme-of-target-serve-only.md) — nvmet kernel NVMe-oF target (serve-only, NQN allowlist)
- [ADR-0004](adr/0004-zfs-backed-container-volumes-full-docker-compose.md) — ZFS-backed container volumes with full docker-compose (dataset-per-volume default; dedicated app UID per stack; opt-in app role)
- [ADR-0005](adr/0005-identity-model-separate-nas-db.md) — identity model (separate NAS DB with optional system/SMB link; NFS host-based access with UID alignment via `idmapd`; app UID allocation)
- [ADR-0006](adr/0006-updates-as-a-product-feature.md) — updates as a product feature (repo-channel-only delivery; signed images)
- [ADR-0007](adr/0007-configurable-storage-layout-assigned-at-install.md) — configurable storage layout assigned at install time (`config` role removed — system state on OS-disk partitions; remaining roles `data` + optional `app`)
- [ADR-0008](adr/0008-observability-otel.md) — observability: OpenTelemetry for all signals (Prometheus always-on exporter; OTLP additive; one `Telemetry` resource; reconcile-loop tracing off by default; journald durable log store)
- [ADR-0009](adr/0009-nfs-share-management-via-zfs-sharenfs.md) — NFS share management via ZFS `sharenfs` properties
- [ADR-0010](adr/0010-supported-multi-protocol-access-combos.md) — supported multi-protocol concurrent access combos (app volumes are XOR with network shares; per-path grants with UID alignment)
- [ADR-0011](adr/0011-os-disk-layout-encryption.md) — OS disk: layout, encryption, and data-encryption keys (dedicated OS disk; LUKS2-sealed slots + spec store + keyfiles + `config/var`; external-factor boot unlock; keyfiles inside the container)
- [ADR-0012](adr/0012-web-ui-static-server-in-image.md) — Web-UI as a separate static server baked into the image
- [ADR-0013](adr/0013-os-disk-writable-state.md) — OS disk writable state: spec store + `config/var` (boot reads spec before pool import; versioned spec store; journald logging + audit with rotation/retention)
- [ADR-0014](adr/0014-network-planes-configurable-at-install.md) — network planes configurable at install (per-interface IP assigned at install)
- [ADR-0017](adr/0017-per-controller-reconcile-loops.md) — per-controller reconcile loops over a shared event source (no hard ordering, idempotent retries)
- [ADR-0018](adr/0018-rbac-fixed-roles-capability-map.md) — RBAC fixed roles with a capability map
- [ADR-0019](adr/0019-feature-map-aligned-api-resource-model.md) — feature-map-aligned API resource model (with `Schedule` resource)
- [ADR-0020](adr/0020-authentication-session-token-argon2.md) — authentication: sessions for the UI, API tokens for CLI/automation, Argon2id
- [ADR-0021](adr/0021-disk-health-smart-scrub-schedule.md) — disk health: core polls SMART via smartctl, metrics + status
- [ADR-0022](adr/0022-shared-schedule-resource.md) — shared `Schedule` resource for snapshots and scrubs
- [ADR-0023](adr/0023-smb-share-management-mechanism.md) — SMB share management via samba config on the `config/var` partition, per-user access control (no `sharesmb`)
- [ADR-0024](adr/0024-pool-vdev-topology-and-disk-replacement.md) — pool/vdev topology (single pool, mirror/raidz1/raidz2, spares, replacement flow)
- [ADR-0028](adr/0028-tls-management-plane.md) — TLS on the management plane (HTTPS in v1; built-in CA / ACME / manual import trust modes)
- [ADR-0029](adr/0029-oidc-authentication.md) — OIDC authentication for the Web-UI (single IdP, authorization-code + PKCE, claims-as-roles, JIT NAS account)
- [ADR-0030](adr/0030-offsite-backup-zfs-snapshots-restic.md) — off-site backup of ZFS snapshots with restic (event-driven per-snapshot ingestion, per-dataset opt-in, data-pool-loss recovery boundary)
- [ADR-0031](adr/0031-core-daemon-controller-runtime-and-startup-sequence.md) — core daemon controller runtime and startup sequence (framework-first runtime, D1–D10)