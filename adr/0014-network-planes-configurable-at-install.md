# ADR-0014: Network planes configurable at install

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 10, §3.7; ADR-0007

## Context

Feature 10 (networking) was a single line: management IP, hostname, NTP, and where
API/UI/shares bind. But API/UI, file/block shares, and app workloads all need a
bind surface, and whether one LAN serves everything or planes are split across
NICs/VLANs is a hardware-and-network-dependent choice that cannot be fixed in the
image — the same shape of problem ADR-0007 solved for storage.

## Decision

Interface roles (management plane vs data plane) are assigned at install time,
mirroring ADR-0007's storage-role assignment. The management plane carries API/UI
and DNS/hostname/NTP; the data plane carries SMB/NFS/NVMe-oF share traffic. App
workloads (ADR-0004) are general compose containers, not thin API clients — they
publish ports on the management plane by default, and their port publishing
follows plane membership like any other share.

Each plane's interface also gets its IP configuration (DHCP or static) in the
same install step, mirroring the management IP (resolve-control-plane-gaps D11).

## Alternatives considered

- **Fixed single LAN for everything**: simplest, but cannot adapt to hardware with
  separate NICs or a segmented network.
- **Fixed split mgmt/data planes**: forces the topology on installs that don't need it.

## Consequences

- The installer performs an interface-role assignment step alongside storage roles
  (feature 12).
- A networking controller reconciles interface, IP, hostname, NTP, and plane
  membership from the spec store (ADR-0002).
- Share and app binding follows plane membership; the default is one management
  LAN for everything, matching the v1 baseline.
- Data-plane interfaces are addressed at install, not left without configuration;
  the installer captures IP/DHCP per interface alongside role assignment.
