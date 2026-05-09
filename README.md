# testinfra

[![Build and Push GHCR Images](https://github.com/peppekerstens/testinfra/actions/workflows/build-images.yml/badge.svg)](https://github.com/peppekerstens/testinfra/actions/workflows/build-images.yml)

Pre-built container images with **PowerShell 7.6.1**, **Pester 5**, and **PSScriptAnalyzer** for multi-distro CI testing of the [peppekerstens Linux PowerShell module collection](https://github.com/peppekerstens/).

Part of the **Linux PowerShell Cmdlet Parity** project — inspired by Evgenij Smirnov's [2025 European PowerShell Summit session](https://www.youtube.com/watch?v=RlzinWYIjBY) and documented in the blog series at [peppekerstens.github.io](https://peppekerstens.github.io/linux-command-wrapping-part-1/).

---

## What it does

Each image is a thin layer on top of a standard distro base. It adds:

- **PowerShell 7.6.1** — via the Microsoft APT/RPM repo (Ubuntu, Debian) or direct tarball (Fedora, openSUSE, Arch)
- **Pester 5.2+** — installed `AllUsers` scope via `Install-Module`
- **PSScriptAnalyzer** — installed `AllUsers` scope via `Install-Module`
- **Linux tool set** — the CLI tools exercised by the module tests: `iproute2`, `ping`, `nc`, `ss`, `parted`, `fdisk`, `e2fsprogs`, `cups-client`, `samba-client`, `procps`, `bind-utils`

Images are built and pushed to the GitHub Container Registry (GHCR) automatically on every change to any `Dockerfile.*` or the build workflow. They can also be triggered manually via `workflow_dispatch`.

---

## Images

| Image | Base | Tag | PS install method |
|---|---|---|---|
| `ghcr.io/peppekerstens/pwsh-pester-ubuntu` | `ubuntu:24.04` | `24.04` | Microsoft APT repo |
| `ghcr.io/peppekerstens/pwsh-pester-debian` | `debian:12` | `12` | Microsoft APT repo |
| `ghcr.io/peppekerstens/pwsh-pester-fedora` | `fedora:40` | `40` | Direct tarball v7.6.1 |
| `ghcr.io/peppekerstens/pwsh-pester-opensuse` | `opensuse/tumbleweed` | `tumbleweed` | Direct tarball v7.6.1 |
| `ghcr.io/peppekerstens/pwsh-pester-arch` | `archlinux:latest` | `latest` | Direct tarball v7.6.1 |

> **openSUSE note:** `leap:15.6` ships a glibc too old for the PS 7.6.1 tarball (segfault on startup). `tumbleweed` is used instead — it has a current glibc and rolling packages.
>
> **Fedora / Arch note:** The Microsoft RHEL9 RPM repository is not compatible with these distros. All three use the upstream tarball release directly.

---

## Usage

### Pull an image

```bash
docker pull ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04
```

### Run Pester against a module

Mount the module directory to `/module` — the default `CMD` runs `Invoke-Pester -Output Detailed` from that path.

```bash
# From a module repo root
docker run --rm \
  --cap-add=NET_RAW \
  -v "$(pwd):/module" \
  ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04
```

> `--cap-add=NET_RAW` is required for tests that call `ping`. Without it, rootless containers return "Operation not permitted" even when the `ping` binary is installed.

### Run via docker compose (recommended)

Every module repo contains a `docker-compose.test.yml` that wires up all five distros:

```powershell
# From a module repo root
docker compose -f docker-compose.test.yml up --abort-on-container-exit
```

### Run all modules at once

A workspace-level script is available at the GitHub root (`run-tests-docker.ps1`):

```powershell
# All modules, all distros
.\run-tests-docker.ps1

# Single module, all distros
.\run-tests-docker.ps1 -Module NetTCPIP.Linux

# Single module, specific compose file
.\run-tests-docker.ps1 -Module NetTCPIP.Linux -Compose docker-compose.test.yml
```

---

## Podman

`podman compose` works as a drop-in replacement for `docker compose`. No licensing concern.

On **Windows**: install Podman directly in WSL2 (`sudo apt install podman`) rather than using `podman machine init` — the machine init downloads a large VM image from `quay.io` and is prone to TLS timeout on slow connections.

WSL2 Podman also requires `nftables` for container networking:

```bash
sudo apt install nftables
sudo nft list ruleset   # verify
```

---

## Tool set per distro

The tool set installed in each image is chosen to match what the module tests actually call. The `netcat` package name varies by distro — this is one of the things that made a shared base image impractical.

| Tool | Ubuntu / Debian | Fedora | openSUSE | Arch |
|---|---|---|---|---|
| `ip`, `ss` | `iproute2` | `iproute` | `iproute2` | `iproute2` |
| `ping` | `iputils-ping` | `iputils` | `iputils` | `iputils` |
| `nc` (netcat) | `netcat-openbsd` | `nmap-ncat` | `netcat-openbsd` | `openbsd-netcat` |
| `ps`, `kill` | `procps` | `procps-ng` | `procps` | `procps-ng` |
| `dig`, `host` | `dnsutils` | `bind-utils` | `bind-utils` | `bind` |
| `parted`, `fdisk` | `parted`, `fdisk` | `parted` | `parted` | `parted` |
| `mkfs.*` | `e2fsprogs`, `dosfstools` | `e2fsprogs`, `dosfstools` | `e2fsprogs`, `dosfstools` | `e2fsprogs`, `dosfstools` |
| `smbstatus` | `samba-common-bin` | `samba-client` | `samba-client` | `samba` |
| `lpstat` | `cups-client` | `cups-client` | `cups-client` | `cups` |
| .NET globalization | (transitive) | (transitive) | `libicu` (explicit) | (transitive) |

> **openSUSE** requires `libicu` to be installed explicitly — other distros pull it in as a transitive dependency of some other package. Without it, PowerShell / .NET throws a globalization exception at startup.

---

## GitHub Actions integration

Each module repo's `.github/workflows/pester.yml` references these images via the `container:` key:

```yaml
jobs:
  pester-linux:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        distro:
          - { image: 'ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04',       slug: ubuntu-24.04 }
          - { image: 'ghcr.io/peppekerstens/pwsh-pester-debian:12',          slug: debian-12 }
          - { image: 'ghcr.io/peppekerstens/pwsh-pester-fedora:40',          slug: fedora-40 }
          - { image: 'ghcr.io/peppekerstens/pwsh-pester-opensuse:tumbleweed',slug: opensuse-tumbleweed }
          - { image: 'ghcr.io/peppekerstens/pwsh-pester-arch:latest',        slug: arch-latest }
    container:
      image: ${{ matrix.distro.image }}
      options: --cap-add=NET_RAW
    steps:
      - uses: actions/checkout@v4
      - run: pwsh -NoProfile -Command "Invoke-Pester -Path ./<ModuleName> -Output Detailed"
      - uses: actions/upload-artifact@v4
        with:
          name: pester-${{ matrix.distro.slug }}
          path: results.xml
```

> Artifact names use the `slug` field — colons and slashes in image tags are not valid in artifact names.

---

## Why not a single shared base image?

The tool set needed by the tests differs enough between distros that a shared base would either pull in too many packages (bloating all images) or still require per-distro layers. The five separate `Dockerfile.*` files are straightforward and transparent — each distro's quirks are visible in plain sight rather than hidden in a shared base layer.

---

## Lessons learned

**`ping` needs `--cap-add=NET_RAW`.** Rootless containers drop the `CAP_NET_RAW` capability required for ICMP raw sockets. The `ping` binary may be installed and executable, but it will return "Operation not permitted" unless you add this capability at `docker run` or `cap_add` in compose.

**openSUSE Tumbleweed, not Leap.** `leap:15.6` ships glibc 2.31; PS 7.6.1 requires a newer glibc and segfaults silently at startup. Tumbleweed is the rolling release and has current glibc.

**Tarball install for Fedora, openSUSE, Arch.** The Microsoft RHEL9 `.rpm` package is built for RHEL/CentOS. It installs on Fedora 40 but the runtime crashes on incompatible system libraries. The upstream tarball from GitHub releases works cleanly on all three.

**`nftables` required for Podman networking in WSL2.** Podman's `netavark` network backend calls `nft` at container startup. If `nftables` is absent, every `podman run` fails with `netavark: nftables error: unable to execute "nft"`.

---

## Version history

| Version | Notes |
|---|---|
| 0.1.0 | Initial release. 5 Dockerfiles; GHCR build workflow; `build-images.yml` push on Dockerfile change. |
| 0.1.1 | openSUSE base changed `leap:15.6` → `tumbleweed`; added `gzip`, `libicu`. Fedora/openSUSE/Arch PS install changed to direct tarball. Added `iputils-ping`, `procps`, and distro-specific `netcat` packages to all images. |

---

## License

GPL-3.0 — see [LICENSE](LICENSE).
