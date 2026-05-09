# testinfra

[![Build and Push GHCR Images](https://github.com/peppekerstens/testinfra/actions/workflows/build-images.yml/badge.svg)](https://github.com/peppekerstens/testinfra/actions/workflows/build-images.yml)

Pre-built container images with **PowerShell 7.6.1**, **Pester 5**, and **PSScriptAnalyzer** for multi-distro testing of the [peppekerstens Linux PowerShell module collection](https://github.com/peppekerstens/).

Part of the **Linux PowerShell Cmdlet Parity** project — inspired by Evgenij Smirnov's [2025 European PowerShell Summit session](https://www.youtube.com/watch?v=RlzinWYIjBY) and documented in the blog series at [peppekerstens.github.io](https://peppekerstens.github.io/linux-command-wrapping-part-1/).

---

## Primary use: local test-bed via Podman

The main reason these images exist is **local iteration**. GitHub Actions CI is the safety net; the images are how you get fast feedback while actively developing a module.

The workflow is:

1. Build your module DLL (or `.psm1`) on your machine.
2. Mount the repo into a container.
3. Run Pester inside that container.
4. Fix, rebuild, repeat — without pushing to GitHub.

You do not need Docker Desktop. **Podman** is the recommended runtime — it is free, rootless-capable, and available in every Linux package manager.

### On WSL2 (Windows development machine)

Install Podman inside your WSL2 distro:

```bash
sudo apt install podman        # Ubuntu / Debian WSL2
```

Podman's `netavark` network backend needs `nftables`:

```bash
sudo apt install nftables
```

Pull an image (must run as root inside WSL2 — rootless Podman requires
`newuidmap` which is not present in the default WSL2 userland):

```bash
sudo podman pull ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04
```

Run Pester against a module directory (mount the repo root to `/module`):

```bash
sudo podman run --rm --privileged \
  -v "/mnt/c/Users/you/GitHub/MyModule:/module" \
  ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04 \
  pwsh -NoProfile -Command '
    $config = New-PesterConfiguration
    $config.Run.Path = "tests/MyModule.Tests.ps1"
    $config.Run.Exit = $false
    $config.Output.Verbosity = "Detailed"
    Invoke-Pester -Configuration $config
  '
```

> `--privileged` is required for write-operation tests that call `useradd`,
> `usermod`, `ip addr add`, or `systemctl`. Read-only tests work with just
> `--cap-add=NET_RAW`.

### On Ubuntu / Debian natively

```bash
sudo apt install podman nftables

sudo podman pull ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04

sudo podman run --rm --privileged \
  -v "$(pwd):/module" \
  ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04 \
  pwsh -NoProfile -Command '
    $config = New-PesterConfiguration
    $config.Run.Path = "tests/MyModule.Tests.ps1"
    $config.Run.Exit = $false
    $config.Output.Verbosity = "Detailed"
    Invoke-Pester -Configuration $config
  '
```

### Running all five distros in sequence

```bash
for IMAGE in \
  ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04 \
  ghcr.io/peppekerstens/pwsh-pester-debian:12 \
  ghcr.io/peppekerstens/pwsh-pester-fedora:40 \
  ghcr.io/peppekerstens/pwsh-pester-opensuse:tumbleweed \
  ghcr.io/peppekerstens/pwsh-pester-arch:latest
do
  echo "=== $IMAGE ==="
  sudo podman run --rm --privileged \
    -v "$(pwd):/module" \
    "$IMAGE" \
    pwsh -NoProfile -Command '
      $config = New-PesterConfiguration
      $config.Run.Path = "tests/MyModule.Tests.ps1"
      $config.Run.Exit = $false
      $config.Output.Verbosity = "Normal"
      Invoke-Pester -Configuration $config
    '
done
```

### Pre-built DLL vs build inside container

The containers do not have `dotnet` installed. Build your DLL on the host
(Windows or WSL2) first, then mount the whole repo. Pester finds the DLL via
the relative path in `BeforeAll`.

If you need a clean build-and-test loop, install the .NET 8 SDK in WSL2
(`sudo apt install dotnet-sdk-8.0`) and run `dotnet build` there before
starting the container.

---

## Secondary use: GitHub Actions CI

Each module repo's `.github/workflows/pester.yml` uses the same images via
the `container:` key, so local and CI runs are identical environments.

```yaml
jobs:
  pester:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        container:
          - ghcr.io/peppekerstens/pwsh-pester-ubuntu:24.04
          - ghcr.io/peppekerstens/pwsh-pester-debian:12
          - ghcr.io/peppekerstens/pwsh-pester-fedora:40
          - ghcr.io/peppekerstens/pwsh-pester-opensuse:tumbleweed
          - ghcr.io/peppekerstens/pwsh-pester-arch:latest
    container:
      image: ${{ matrix.container }}
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
      options: --privileged
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'
      - name: Build
        run: dotnet build src/MyModule/MyModule.csproj --configuration Release
      - name: Run Pester tests
        shell: pwsh
        run: |
          $config = New-PesterConfiguration
          $config.Run.Path = 'tests/MyModule.Tests/MyModule.Tests.ps1'
          $config.Run.Exit = $true
          $config.Output.Verbosity = 'Detailed'
          Invoke-Pester -Configuration $config
```

Key points that differ from the local invocation:
- `actions/setup-dotnet` installs the SDK inside the container at job runtime.
- `shell: pwsh` is required so `$PSScriptRoot` is set correctly in the `run:` block (the runner writes the script to a temporary `.ps1` file and executes it).
- `$config.Run.Exit = $true` makes Pester exit non-zero on failure, which fails the step.
- `credentials:` block is required because the images live on GHCR (a private registry relative to the runner).

---

## Images

| Image | Base | Tag | PS install method |
|---|---|---|---|
| `ghcr.io/peppekerstens/pwsh-pester-ubuntu` | `ubuntu:24.04` | `24.04` | Microsoft APT repo |
| `ghcr.io/peppekerstens/pwsh-pester-debian` | `debian:12` | `12` | Microsoft APT repo |
| `ghcr.io/peppekerstens/pwsh-pester-fedora` | `fedora:40` | `40` | Direct tarball v7.6.1 |
| `ghcr.io/peppekerstens/pwsh-pester-opensuse` | `opensuse/tumbleweed` | `tumbleweed` | Direct tarball v7.6.1 |
| `ghcr.io/peppekerstens/pwsh-pester-arch` | `archlinux:latest` | `latest` | Direct tarball v7.6.1 |

> **openSUSE note:** `leap:15.6` ships a glibc too old for the PS 7.6.1 tarball (segfault on startup). `tumbleweed` is used instead.
>
> **Fedora / Arch note:** The Microsoft RHEL9 RPM is not compatible with these distros. The upstream tarball from GitHub releases works cleanly on all three.

Images are rebuilt and pushed to GHCR automatically on every `Dockerfile.*` change via the `build-images.yml` workflow, and can be triggered manually via `workflow_dispatch`.

---

## Tool set per distro

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

> **openSUSE** requires `libicu` explicitly — other distros pull it in transitively. Without it, PowerShell throws a globalization exception at startup.

---

## Lessons learned

**`--privileged` vs `--cap-add=NET_RAW`.** Read-only tests need `--cap-add=NET_RAW` for `ping`. Write tests (user/group creation, interface configuration, systemd interaction) need `--privileged`. Using `--privileged` for everything is the safe default in a local test environment.

**`sudo podman` in WSL2.** Rootless Podman requires `newuidmap`, which is not in the default WSL2 userland. Running as root sidesteps the issue cleanly. The containers themselves are also root inside, which matches GHA container behaviour.

**`shell: pwsh` in GHA is mandatory.** Using `run: pwsh -Command "..."` leaves `$PSScriptRoot` empty, which breaks any test that uses it to locate its own DLL or helper files. `shell: pwsh` causes the runner to write the script to a temp file and set `$PSScriptRoot` correctly.

**`& id -u` in `BeforeDiscovery` is fragile.** On some distros it triggers unexpected errors inside Pester discovery. Use `/proc/self/status` instead:
```powershell
$script:isRoot = $IsLinux -and (
    [System.IO.File]::ReadAllText('/proc/self/status') -match '(?m)^Uid:\s+(\d+)' -and
    $Matches[1] -eq '0')
```

**openSUSE Tumbleweed, not Leap.** `leap:15.6` ships glibc 2.31; PS 7.6.1 requires a newer glibc and segfaults silently at startup.

**Tarball install for Fedora, openSUSE, Arch.** The Microsoft RHEL9 `.rpm` is built for RHEL/CentOS. The upstream tarball from GitHub releases works cleanly on all three.

**`nftables` required for Podman networking in WSL2.** Podman's `netavark` network backend calls `nft` at container startup. If `nftables` is absent, every `podman run` fails with `netavark: nftables error: unable to execute "nft"`.

---

## Version history

| Version | Notes |
|---|---|
| 0.1.0 | Initial release. 5 Dockerfiles; GHCR build workflow. |
| 0.1.1 | openSUSE base `leap:15.6` → `tumbleweed`; added `gzip`, `libicu`. Fedora/openSUSE/Arch PS install changed to direct tarball. Added `iputils-ping`, `procps`, distro-specific `netcat` to all images. |
| 0.2.0 | README rewritten to lead with local podman/WSL2 test-bed usage. GHA integration section updated with `shell: pwsh`, `credentials:`, and `--privileged` guidance. Lessons-learned section expanded with `$PSScriptRoot` and `id -u` pitfalls. |

---

## License

GPL-3.0 — see [LICENSE](LICENSE).
