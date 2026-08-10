# NAS Documentation

- [Architecture](architecture/01-os-feature-map.md) — target feature map and architecture decisions (discovery-phase capture).
- [ADR-0001](adr/0001-read-only-squashfs-root-ab-boot.md) — read-only squashfs root with A/B boot (amended: `core` baked into the image; no host-file writes under `/`)
- [ADR-0002](adr/0002-declarative-desired-state-api-reconciler-core.md) — declarative desired-state API with reconciler core (amended: composite rollback)
- [ADR-0003](adr/0003-nvmet-kernel-nvme-of-target-serve-only.md) — nvmet kernel NVMe-oF target (serve-only)
- [ADR-0004](adr/0004-zfs-backed-container-volumes-full-docker-compose.md) — ZFS-backed container volumes with full docker-compose (amended: dataset-per-volume default)
- [ADR-0005](adr/0005-identity-model-separate-nas-db.md) — identity model (separate NAS DB with optional system/SMB link) (amended: NFS model — host-based access, UID alignment to the NAS user)
- [ADR-0006](adr/0006-updates-as-a-product-feature.md) — updates as a product feature (amended: repo-channel-only delivery)
- [ADR-0007](adr/0007-configurable-storage-layout-assigned-at-install.md) — configurable storage layout assigned at install time
- [ADR-0008](adr/0008-prometheus-metrics-for-every-subsystem.md) — Prometheus metrics for every subsystem (amended: scrape-only hosting, no embedded server)
- [ADR-0009](adr/0009-nfs-share-management-via-zfs-sharenfs.md) — NFS share management via ZFS `sharenfs` properties
- [ADR-0010](adr/0010-supported-multi-protocol-access-combos.md) — supported multi-protocol concurrent access combos (amended: app volumes are XOR with network shares)
- [ADR-0011](adr/0011-dedicated-os-disk-ab-slots.md) — dedicated OS disk for the A/B squashfs slots
- [ADR-0012](adr/0012-web-ui-static-server-in-image.md) — Web-UI as a separate static server baked into the image
