# ADR-0028: TLS on the management plane

- Status: accepted
- Date: 2026-08-16
- Deciders: Amberhold design (discovery phase)
- References: `docs/architecture/01-os-feature-map.md` feature 10, §6;
  ADR-0001, ADR-0006, ADR-0014; `scope-missing-plane` decision D6

## Context

The v1 baseline serves the API/UI over plain HTTP on the management LAN; TLS is
on the feature-map deferred list, recorded as design decision D6 in the
`scope-missing-plane` change. OIDC authentication (ADR-0029) is a common
enterprise/NAS expectation, but its authorization-code flow requires an HTTPS
`redirect_uri` — a plain-HTTP management LAN cannot host it. TLS therefore becomes
a prerequisite for OIDC and must be pulled into v1.

## Decision

The management plane serves the API and Web-UI over HTTPS in v1. This reverses the
deferred plain-HTTP baseline (`scope-missing-plane` D6) and removes TLS from the
feature-map deferred list.

Certificate provisioning supports three trust modes, selectable at configuration
time:

- **Built-in CA (default)**: the NAS generates a CA and a server certificate at
  install; the CA is the trust anchor that users import into their browsers or
  devices. This matches the baked-in update-image trust anchor (ADR-0001,
  ADR-0006) and keeps the default path free of external dependencies.
- **ACME**: for deployments that expose the management plane publicly; the NAS
  obtains and renews certificates from an ACME provider.
- **Manual cert import**: for enterprise deployments with an existing PKI; the
  admin imports the server certificate and its trust anchor.

The trust anchor has a defined lifecycle: certificates are renewed per the
configured mode, rotation replaces the serving certificate and (where applicable)
the CA, and a CA compromise is handled by replacing the trust anchor and re-issuing
server certificates. The management-plane identity is reconciled from the spec
store like any other resource (ADR-0002, ADR-0017).

## Alternatives considered

- **TLS only on the auth endpoints**: rejected — the UI serves the API from the
  same origin and IdP redirect URIs are per-origin, so a partial-HTTPS plane does
  not satisfy the OIDC requirement and leaves the rest of the management traffic
  unencrypted.
- **Forcing a single certificate mode**: rejected — NAS deployments range from
  air-gapped LAN appliances to publicly reachable hosts; one mode cannot cover
  both, mirroring the install-time flexibility of ADR-0014.

## Consequences

- The management plane (feature 10) is HTTPS in v1; the plain-HTTP baseline and the
  TLS deferral are removed from the feature-map deferred list.
- ADR-0029 (OIDC) depends on this decision; its `redirect_uri` is HTTPS.
- Certificate and CA state follow the writable-config convention: generated files
  on the OS-disk `config/var` partition with `/etc` symlinks (ADR-0001), owned by a
  networking/cert controller.
- Management-plane binding and plane membership remain as specified by ADR-0014.
- Data-plane traffic (SMB/NFS/NVMe-oF) is unaffected — TLS covers the management
  plane only.
