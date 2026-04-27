# vllm-swap: Unified container for vLLM + llama-swap
# Provides model swapping capability with vLLM inference engine
#
# Build:
#   docker buildx build --platform linux/amd64 -t vllm-swap:latest .
#
# Usage:
#   docker run --gpus all -p 8080:8080 -v /path/to/models:/models vllm-swap:latest
#
# =============================================================================

ARG CUDA_VERSION=13.0.2
ARG PYTHON_VERSION=3.12
ARG UBUNTU_VERSION=22.04

ARG VLLM_VERSION=""
ARG LLAMA_SWAP_VERSION=latest

#################### RUNTIME STAGE ####################
FROM nvidia/cuda:${CUDA_VERSION}-base-ubuntu${UBUNTU_VERSION} AS runtime

ARG CUDA_VERSION
ARG VLLM_VERSION
ARG LLAMA_SWAP_VERSION

ENV DEBIAN_FRONTEND=noninteractive
ENV LD_LIBRARY_PATH="/usr/local/cuda/lib64:${LD_LIBRARY_PATH}"
ENV PATH="/usr/local/bin:${PATH}"

# Install runtime dependencies
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
        libgomp1 \
        python3 \
        python3-pip \
        curl \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && pip3 install --no-cache-dir pip --upgrade

# Install vLLM (use pre-built wheel from PyPI)
RUN if [ -n "${VLLM_VERSION}" ]; then \
        pip3 install --no-cache-dir vllm==${VLLM_VERSION}; \
    else \
        pip3 install --no-cache-dir vllm; \
    fi

# Download and install llama-swap release binary
RUN mkdir -p /install/bin \
    && ARCH=$(uname -m) \
    && case "$ARCH" in \
        x86_64) ARCH="amd64" ;; \
        aarch64|arm64) ARCH="arm64" ;; \
        *) echo "Unsupported architecture: $ARCH" >&2; exit 1 ;; \
    esac \
    && if [ "${LLAMA_SWAP_VERSION}" = "latest" ]; then \
        LS_VER=$(curl -fsSL "https://api.github.com/repos/mostlygeek/llama-swap/releases/latest" \
            | grep '"tag_name"' | head -1 | cut -d'"' -f4 | sed 's/^v//'); \
    else \
        LS_VER="${LLAMA_SWAP_VERSION}"; \
    fi \
    && echo "Downloading llama-swap v${LS_VER}" \
    && curl -fSL -o /tmp/llama-swap.tar.gz \
        "https://github.com/mostlygeek/llama-swap/releases/download/v${LS_VER}/llama-swap_${LS_VER}_linux_${ARCH}.tar.gz" \
    && tar -xzf /tmp/llama-swap.tar.gz -C /install/bin/ \
    && rm /tmp/llama-swap.tar.gz \
    && chmod +x /install/bin/llama-swap \
    && echo "${LS_VER}" > /install/llama-swap-version

# Create directories
RUN mkdir -p /etc/llama-swap/config \
    && mkdir -p /models \
    && mkdir -p /app

# Copy example config
COPY config.example.yaml /etc/llama-swap/config/config.yaml

# Version tracking
RUN echo "vllm: $(python3 -c 'import vllm; print(vllm.__version__)' 2>/dev/null || echo 'unknown')" > /versions.txt \
    && echo "llama-swap: $(cat /install/llama-swap-version 2>/dev/null || echo 'unknown')" >> /versions.txt \
    && echo "cuda: ${CUDA_VERSION}" >> /versions.txt \
    && echo "build_timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> /versions.txt

WORKDIR /models

EXPOSE 8080

ENTRYPOINT ["llama-swap"]
CMD ["-config", "/etc/llama-swap/config/config.yaml", "-listen", "0.0.0.0:8080"]
