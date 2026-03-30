# alpha-dev-container

GPU-accelerated multi-language dev container for VS Code, because Windows.

Container Size: ~14.27GB

## What's Included

| Category | Details |
|---|---|
| **Base** | `nvidia/cuda:12.8.1-base-ubuntu24.04` |
| **GPU** | CUDA 12.8.1 runtime, compatible with RTX 3060 Laptop (Ampere, CC 8.6) |
| **Python** | 3.13 + [uv](https://github.com/astral-sh/uv) (PyTorch CUDA 12.8 wheels pre-cached) |
| **Rust** | Latest stable via rustup (minimal profile) |
| **Go** | 1.24.2 |
| **Node.js** | v24 LTS + npm + TypeScript |
| **Java** | 25 (Eclipse Temurin JDK) |
| **C/C++** | gcc / g++ / make (build-essential) |
| **User** | `devuser` (UID 1000) with passwordless sudo |

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with WSL 2 backend
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- NVIDIA GPU driver installed on the host
- VS Code with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension

## Building the Image

```bash
cd alpha-dev-containers
docker build -t alpha2303/alpha-dev-container -f .devcontainer/Dockerfile .devcontainer/
```

To tag a specific version:

```bash
docker tag alpha2303/alpha-dev-container alpha2303/alpha-dev-container:YYYYMMDD-HHMM
```

## Usage in Any Project

1. Create `.devcontainer/devcontainer.json` in your project root:

```jsonc
{
    "name": "Alpha Dev",
    "image": "alpha2303/alpha-dev-container:20260329-2125",
    "runArgs": ["--gpus", "all"],
    "remoteUser": "devuser",
    "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",
    "workspaceMount": "source=${localWorkspaceFolder},target=/workspaces/${localWorkspaceFolderBasename},type=bind,consistency=cached",
    "hostRequirements": { "gpu": "optional" }
}
```

2. Open the project in VS Code
3. `Ctrl+Shift+P` → **Dev Containers: Reopen in Container**

Your project files are bind-mounted — changes sync both ways in real time.

## PyTorch Setup

PyTorch CUDA 12.8 wheels are pre-cached in uv's global cache. 
This increases container size by ~4GB, but reduces project setup time and saves disk space from duplicate installations.

To use in a project:

```bash
uv init myproject && cd myproject
uv add torch torchvision --index-url https://download.pytorch.org/whl/cu128
```

The first `uv add torch` is instant (cache hit, hard-linked — no download). Multiple projects share the same cached wheels without duplication.

Verify GPU access:

```bash
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

## Pushing to Docker Hub

```bash
docker push alpha2303/alpha-dev-container:20260329-2125
```
