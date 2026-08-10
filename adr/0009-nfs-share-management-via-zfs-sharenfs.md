# ADR-0009: NFS share management via ZFS `sharenfs` properties

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.2, decision 2 (D1);
  ADR-0001, ADR-0002

## Context

The architecture skeleton originally had the NFS controller write `/etc/exports`.
That is impossible on the read-only squashfs root (ADR-0001): no subsystem may
write host files under `/`. The NFS controller needs a way to express shares that
survives the immutable root and keeps share state consistent.

## Decision

NFS shares are managed via ZFS `sharenfs` properties on datasets
(`zfs set sharenfs=...`). Shares are expressed as dataset properties, mirroring
the `sharesmb` model for SMB. The share controller treats ZFS as the single
source of share state; no `/etc/exports` writer exists.

## Alternatives considered

- **Bind-mount a writable exports file** over `/etc/exports`: keeps classic
  semantics but splits share state into two stores and fights the RO root.
- **Writable `/etc/exports` via overlay**: violates RO-root immutability
  (ADR-0001).

## Consequences

- The NFS controller drives `zfs set sharenfs`; the skeleton diagram is corrected
  accordingly.
- Share state lives in ZFS dataset properties, consistent with `sharesmb` and
  observable like any dataset property.
- `sharenfs` semantics vary slightly across ZFS versions; the ZFS userland is
  pinned in the image (ADR-0001) and share semantics are verified in the
  `file-shares` design.
