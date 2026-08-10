# ADR-0010: Supported multi-protocol concurrent access combos

- Status: accepted
- Date: 2026-08-10
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` §3.3, decision 5 (D5);
  ADR-0003, ADR-0009

## Context

One dataset may be exposed over several protocols. Serving it through all of them
at once is a locking and cache-coherency hazard (e.g. NFS clients caching while
SMB clients write, or a block client and a file client sharing the same data).
The supported exposure matrix must be explicit rather than accidental.

## Decision

- **SMB + NFS on a dataset: supported** — the classic multi-protocol file sharing
  combination; both ride on the same dataset properties (`sharesmb` + `sharenfs`).
- **NVMe-oF: dedicated zvols only** — a dataset's data is never exported over
  NVMe-oF; block exports are dedicated zvols (ADR-0003) and are not concurrently
  served over a file protocol.

## Alternatives considered

- **Any protocol on any dataset simultaneously**: maximally flexible, but
  coherency/locking hazards are hard to reason about and support.

## Consequences

- The `file-shares` design encodes SMB+NFS on a dataset; the `block-shares`
  design restricts NVMe-oF exports to zvols.
- The API/UI present the supported/unsupported matrix so users are not surprised
  by a rejected combination.
