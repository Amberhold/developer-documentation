# Development environment

> How to build, run, and test Amberhold locally. The OS image, the live-ISO
> installer, and the boot harness all run on a single Debian 13 (trixie) amd64
> host with nested virtualization; the retired split build-VM harness
> (`os-image-installer` D26 → `linux-amd64-dev-harness` D1) is not supported.

## Host requirements

The dev host is a Debian 13 (trixie) amd64 machine (physical or VM) with
**nested virtualization** enabled:

| Requirement | Notes |
|-------------|-------|
| Debian 13 (trixie) amd64 | The image build and boot tooling are Linux-only. |
| Nested virtualization | `kvm_intel`/`kvm_amd` with `nested=Y`; `/dev/kvm` present. `systemd-detect-virt` should report `kvm` inside the guest if the host is itself virtualized. |
| `kvm` group access | Provided by the provisioning script; on a VM the group often starts empty. |
| Resources | 4 vCPU and ~15 GiB RAM build and boot comfortably; the sparse 140 GiB scenario disks need ~130 GiB free on `/`. |

If `/dev/kvm` is missing, enable nested virtualization on the host (the
hypervisor, not Amberhold) before provisioning. There is no software-emulation
(TCG) fallback — the harness fails loudly instead of booting slowly.

## Provisioning

One idempotent script installs everything the build and harness need:

```sh
sudo infra/tests/provision-dev-vm.sh
```

It installs the apt build/boot tooling (`mkosi`, `rauc`, `squashfs-tools`,
`dosfstools`, `mtools`, `parted`, `kpartx`, `cryptsetup`, `mdadm`,
`systemd-container`, `qemu-system-x86`, `qemu-utils`, `ovmf`,
`build-essential`, `jq`, `rsync`, `lsof`, `expect`, `smbclient`, `cifs-utils`,
`nfs-common`), the pinned Go toolchain from `go.dev` (apt's Go predates the
repos' `go 1.27` directive), Caddy from the official Cloudsmith repository (for
`tests/frontdoor-check.sh`), the Web-UI e2e browser (Playwright Chromium plus
its system libraries), and adds the invoking user to the `kvm` group.

Re-running it makes no destructive change and reports the environment as ready.
After the first run, **log out and back in** (or run the harness through
`sg kvm -c ...`) so the new `kvm` group membership applies to your shell. The
script reports `/dev/kvm` accessibility at the end.

!!! note "Caddy signing key"
    Cloudsmith signs the Caddy repository with a subkey whose expiry has
    passed; Debian trixie's Sequoia apt verifier rejects it. The provisioning
    script therefore marks that official repository `[trusted=yes]` and still
    imports the key for provenance.

Check the toolchain at any time with:

```sh
qemu-system-x86_64 --version
go version          # go1.27.x
caddy version
mkosi --version
rauc --version
```

## Repos

Amberhold is a multi-repo workspace. From this meta-repo:

```sh
git submodule update --init --recursive
```

| Repo | Language | Inner loop |
|------|----------|------------|
| `core` | Go | `cd core && go build ./... && go vet ./... && go test ./...` |
| `cli` | Go | `cd cli && go build ./... && go vet ./...` |
| `infra/installer` | Go | `cd infra/installer && go test ./...` |
| `web-ui` | TypeScript (Bun) | see below |
| `docs` | Markdown (Zensical) | `cd docs && uvx zensical build --clean` |

The `contracts` repo is schemas only; `web-ui` vendors the generated contract
and its `drift-check` fails if the vendored copy is stale.

## Web-UI toolchain

Bun is pinned in `web-ui/.bun-version`. Install dependencies and run the suite:

```sh
cd web-ui
bun install --frozen-lockfile
bun run build          # drift-check + typegen + formgen + tsc + vite build
bun run test           # vitest
bun run test:e2e       # Playwright (mock mode; Chromium is provisioned)
```

`bun run test:e2e` starts the Vite dev server in mock mode by default; point it
at a real core with `AMBERHOLD_E2E_BASE`/`AMBERHOLD_DEV_API` to exercise the
front door.

## OS image build

The image is built with `mkosi` on this host (the same toolchain CI uses). The
harness boots a built installer **live image** (and its kernel/initrd) staged
under `$SHARED/build-output`, so a single helper builds and stages everything:

```sh
sudo infra/tests/build-harness-images.sh
```

It builds the `core` daemon, the installer binary, and the Web-UI bundle;
generates a dev signing PKI and the harness repo CA; builds the OS image
(`amberhold.raw`); turns its rootfs into the squashfs slot and signs the initial
rauc bundle; builds the installer-only live image; stages
`$SHARED/build-output/{amberhold-installer.raw,.vmlinuz,.initrd}`; and publishes
the initial bundle into `$SHARED/repo` for the update scenarios.

The underlying steps, if you need them individually — the explicit `:/build`
target is required, because the bake reads the staged tree at
`/work/src/build`, while a bare `--build-sources` lands it at `/work/src`:

```sh
cd core && go build -o amberhold-core ./cmd/amberhold-core && cd ..
cd web-ui && bun install --frozen-lockfile && bun run build && cd ..

# OS image (stage core binary, web-ui/dist, plus rauc/ca.cert.pem and
# repo-ca/repo-ca.crt)
infra/os-image/stage-build-sources.sh "$SHARED/build" \
    core/amberhold-core web-ui/dist
mkosi -C infra/os-image/mkosi --build-sources "$SHARED/build:/build" build

# Installer live image (stage installer/, rauc/ incl. ca.cert.pem, and the
# signed bundle/amberhold.raucb)
mkosi -C infra/os-image/mkosi-live --build-sources "$SHARED/build-live:/build" build
```

The amd64 OS image lands at `/var/tmp/amberhold-build/amberhold.raw` and the
live image at `/var/tmp/amberhold-build-live/amberhold-installer.raw`. The ZFS
kernel module is compiled at build time (DKMS) in both images — the long pole.

## Dev harness

The harness (`infra/tests/harness.sh`) builds nothing itself — it boots the
already-built images with qemu and runs scenarios against the guest. It is a
single-machine Linux/amd64 harness:

- `qemu-system-x86_64 -accel kvm -machine q35 -cpu host`
- OVMF (x86_64 UEFI) firmware: read-only code pflash
  (`/usr/share/OVMF/OVMF_CODE_4M.fd`) plus a writable vars store copied from
  `OVMF_VARS_4M.fd`
- slirp networking with the management front door, the update repo, and the
  SMB/NFS share listeners forwarded to unprivileged host ports

```sh
# One scenario, or several
infra/tests/harness.sh boot-smoke
infra/tests/harness.sh boot-smoke full-matrix core-vs-real-host installer-iso share-mount
```

The share and core-vs-real-host scenarios mount the guest's shares or update
repo from the host; host-side mounts need root, so pre-authenticate with
`sudo -v` first.

The harness is arch-parameterized (`AMBERHOLD_ARCH`, default `amd64`; only
amd64 is implemented) and its firmware, KVM device, and share ports are all
overridable through `AMBERHOLD_*` environment variables. A dry run prints the
resolved qemu argv without launching a guest:

```sh
AMBERHOLD_DRY_RUN=1 infra/tests/harness.sh boot-smoke
```

See `docs/architecture/13-os-image-installer.md` §4a for the harness design and
the scenario-by-scenario assertions.
