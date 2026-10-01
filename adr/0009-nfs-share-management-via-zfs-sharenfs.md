# ADR-0009: NFS share management via ZFS `sharenfs` properties

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.1, decision 1
  (consequence), `scope-missing-plane` D1; ADR-0001, ADR-0002

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

## Generated values always include `insecure` (amendment)

The generated `sharenfs` value is an explicit option list rather than the opaque
`on`:

- no hosts: `rw=*,crossmnt,no_subtree_check,insecure`
- hosts: `rw=<h1>:<h2>,crossmnt,no_subtree_check,insecure`

The list preserves the `sharenfs=on` defaults (`crossmnt`, `no_subtree_check`)
and adds `insecure`, which waives mountd's privileged-source-port requirement. A
client whose NFS requests originate from an unprivileged source port can
therefore mount the export. This matters for the `qemu`/slirp dev harness: slirp
`hostfwd` cannot preserve a privileged source port, so a `secure` export is
refused with `MNT3ERR_ACCES` even though the export is live.

**Recorded security trade-off:** the privileged-port requirement no longer
applies to Amberhold's exports. Host/IP grants remain the access gate — only the
allow-listed clients may mount — so the waiver does not widen *which* clients may
access a share. There is deliberately no `secure`/`insecure` per-share toggle in
v1; a future option can revisit it. This supersedes the original decision's
implicit reliance on `sharenfs` defaults.
