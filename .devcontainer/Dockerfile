FROM nvidia/cuda:12.8.1-base-ubuntu24.04

ARG DEBIAN_FRONTEND=noninteractive
ARG GO_VERSION=1.24.2
ARG USERNAME=devuser
ARG USER_UID=1000
ARG USER_GID=${USER_UID}

# ── Layer 1: System packages + Python + Node.js + Java + create user ─────────
RUN \
    # System packages first (sudo needed for user setup)
    apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential git curl ca-certificates gnupg \
        unzip zip pkg-config libssl-dev sudo software-properties-common \
    # Remove pre-existing UID/GID 1000 (ubuntu user in base image)
    && if id -u ${USER_UID} >/dev/null 2>&1; then \
        userdel -r "$(getent passwd ${USER_UID} | cut -d: -f1)" 2>/dev/null || true; \
    fi \
    && if getent group ${USER_GID} >/dev/null 2>&1; then \
        groupdel "$(getent group ${USER_GID} | cut -d: -f1)" 2>/dev/null || true; \
    fi \
    && groupadd --gid ${USER_GID} ${USERNAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} -m ${USERNAME} -s /bin/bash \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME} \
    # Python 3.13
    && add-apt-repository -y ppa:deadsnakes/ppa \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        python3.13 python3.13-venv python3.13-dev \
    && update-alternatives --install /usr/bin/python python /usr/bin/python3.13 1 \
    && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.13 1 \
    # Node.js v24 LTS + TypeScript
    && curl -fsSL https://deb.nodesource.com/setup_24.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && npm install -g typescript \
    && npm cache clean --force \
    # Java 25 (Eclipse Temurin JDK)
    && curl -fsSL https://packages.adoptium.net/artifactory/api/gpg/key/public \
        | gpg --dearmor -o /usr/share/keyrings/adoptium.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(. /etc/os-release && echo $VERSION_CODENAME) main" \
        > /etc/apt/sources.list.d/adoptium.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends temurin-25-jdk \
    # Cleanup: purge build-only packages, autoremove, wipe apt cache
    && apt-get purge -y software-properties-common \
    && apt-get autoremove -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# ── Install as devuser (avoids mv + chown layer that doubles size) ───────────
USER ${USERNAME}

# ── Layer 2: uv + Rust + Go ─────────────────────────────────────────────────
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && . "$HOME/.local/bin/env" \
    # Rust (minimal profile: rustc + cargo only, saves ~300 MB)
    && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
        | sh -s -- --default-toolchain stable --profile minimal -y \
    && rm -rf "$HOME/.rustup/tmp" \
    # Go (needs sudo for /usr/local)
    && sudo curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" -o /tmp/go.tar.gz \
    && sudo tar -C /usr/local -xzf /tmp/go.tar.gz \
    && sudo rm -rf /tmp/go.tar.gz /tmp/*

# ── Layer 3: Pre-warm uv cache with PyTorch CUDA 12.8 wheels ────────────────
# Large NVIDIA packages (500MB+) need extended timeout for Docker build network
RUN export PATH="$HOME/.local/bin:$PATH" \
    && UV_REQUEST_TIMEOUT=1200 uv venv /tmp/_pytorch_warmup --python python3.13 \
    && UV_REQUEST_TIMEOUT=1200 uv pip install --python /tmp/_pytorch_warmup/bin/python \
        torch torchvision \
        --index-url https://download.pytorch.org/whl/cu128 \
    && rm -rf /tmp/_pytorch_warmup

# ── Environment (single layer) ───────────────────────────────────────────────
ENV JAVA_HOME="/usr/lib/jvm/temurin-25-jdk-amd64" \
    GOPATH="/home/${USERNAME}/go" \
    CARGO_HOME="/home/${USERNAME}/.cargo" \
    RUSTUP_HOME="/home/${USERNAME}/.rustup" \
    NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=compute,utility \
    PATH="/home/${USERNAME}/.local/bin:/home/${USERNAME}/.cargo/bin:/usr/local/go/bin:/usr/lib/jvm/temurin-25-jdk-amd64/bin:${PATH}"

WORKDIR /workspaces
