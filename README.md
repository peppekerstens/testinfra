# testinfra

Pre-built Docker images with **PowerShell 7** and **Pester 5** for
multi-distro CI testing of the `peppekerstens` Linux PowerShell modules.

## Images

| Image | Base | Tag |
|---|---|---|
| `ghcr.io/peppekerstens/pwsh-pester-ubuntu` | `ubuntu:24.04` | `24.04` |
| `ghcr.io/peppekerstens/pwsh-pester-debian` | `debian:12` | `12` |
| `ghcr.io/peppekerstens/pwsh-pester-fedora` | `fedora:40` | `40` |
| `ghcr.io/peppekerstens/pwsh-pester-opensuse` | `opensuse/leap:15.6` | `leap` |
| `ghcr.io/peppekerstens/pwsh-pester-arch` | `archlinux:latest` | `latest` |

Images are built and pushed to GHCR automatically on changes to any
`Dockerfile.*` or the build workflow. Trigger manually via
`workflow_dispatch`.

## Usage

### Run tests locally with Docker

```powershell
# from any module repo root
docker compose -f docker-compose.test.yml up
```

### Run all modules

```powershell
# from C:\Users\peppe\OneDrive\GitHub\
.\run-tests-docker.ps1
```

### Run a single module

```powershell
.\run-tests-docker.ps1 -Module NetTCPIP.Linux
```

## Podman

`podman compose` works as a drop-in replacement for `docker compose`.
On Windows: `podman machine init` (WSL2 backend) required first.
