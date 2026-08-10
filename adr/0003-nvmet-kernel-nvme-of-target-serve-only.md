# ADR-0003: nvmet kernel NVMe-oF target (serve-only)

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.3, decision 3
- Amendments: access control — each block export carries an explicit NQN
  allowlist; `allow_any_host` is never enabled.

## Context

The NAS must export zvols as block devices to remote hosts over NVMe-oF. Two
target implementations exist: the kernel `nvmet` target (configfs) or a userspace
SPDK target.

## Decision

Use the kernel `nvmet` target, driven by the daemon through configfs. Amberhold
serves zvols only; it does not consume remote NVMe-oF (serve-only scope).

## Alternatives considered

- **SPDK userspace target**: richer feature set (RDMA etc.) but heavier and not
  needed for the serve-only scope.

## Consequences

- The `nvme-target` kernel module must be baked into the image (ADR-0001).
- The block-shares controller writes nvmet configfs entries to define targets,
  subsystems, and namespaces backed by zvols.
- Because the system is serve-only, A/B boot never interacts with remote block
  devices; there is no client-side story to design.
