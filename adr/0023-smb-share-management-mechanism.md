# ADR-0023: SMB share management — samba config on the config/var partition, per-user access control

- Status: accepted
- Date: 2026-08-11
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 3, §3.5; ADR-0001,
  ADR-0005, ADR-0010, ADR-0011, ADR-0013

## Context

Feature 3 (SMB shares) has no mechanism decision: NFS got one (ADR-0009, via ZFS
`sharenfs`), but SMB only inherits the `sharesmb` analogy, the `/etc/smb.conf`
symlink convention (ADR-0001), and the identity link (ADR-0005). ZFS `sharesmb`
is a thin mechanism: it expresses a dataset as a share but does not carry
per-user access control. Samba's own access control — which users may access
which share — lives in the samba configuration and passdb, not in dataset
properties. The file-shares design therefore needs a decided mechanism for how a
`Share` resource plus a user grant becomes an actual samba configuration on the
read-only root.

## Decision

SMB share management is samba-based, driven by the daemon:

- The SMB controller generates a samba configuration on the OS-disk `config/var`
  partition and symlinks `/etc/smb.conf` to it (the ADR-0001 RO-root convention).
  There is no `sharesmb` mechanism: shares are samba `[share]` sections, because
  samba sections are the only place per-user access control can be expressed.
- A `Share` resource (ADR-0019) grants access to NAS users (ADR-0005). The
  controller materializes each grant as a samba section entry and a `smbpasswd`
  entry for the linked user (ADR-0005 identity link), and reloads samba.
- Per-user access control is expressed in samba share sections
  (`valid users` / `read only` per grant); dataset-level `sharesmb` properties
  are not used, so share state stays single-sourced in the spec store rather than
  split across samba config and ZFS properties.
- NFS sharing of the same dataset (ADR-0010 supported combo) continues to use
  `sharenfs` (ADR-0009); the two mechanisms gate different access paths and are
  kept coherent by the identity controller's UID allocation (ADR-0005).

## Alternatives considered

- **`sharesmb` only (ZFS property, mirroring ADR-0009)**: symmetric with NFS, but
  cannot express per-user access control, which feature 3 requires.
- **`sharesmb` for share creation + samba sections for ACLs**: splits share state
  across two stores and two mechanisms for one feature.
- **Dynamic samba config under `/etc` via overlay**: violates RO-root
  immutability (ADR-0001).

## Consequences

- The file-shares design encodes samba config generation, `smbpasswd` lifecycle,
  and per-grant reload semantics; the symlink target lives on `config/var`
  (ADR-0001, ADR-0011) and is regenerated, not snapshotted (ADR-0013).
- Share state is single-sourced in the spec store; samba config is derived state
  (ADR-0002).
- The SMB controller's generated config is regenerable state on `config/var`,
  consistent with the ADR-0001 convention for daemon configs.
- Per-user SMB grants map 1:1 to the NAS user DB link (ADR-0005); deleting a user
  removes their passdb entry and share-section entries in the same reconcile
  cycle.
