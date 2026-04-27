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

ARG BUILD_BASE_IMAGE=nvidia/cuda:${CUDA_VERSION}-devel-ubuntu${UBUNTU_VERSION}
ARG FINAL_BASE_IMAGE=nvidia/cuda:${CUDA_VERSION}-base-ubuntu${UBUNTU_VERSION}

ARG VLLM_VERSION=""
ARG LLAMA_SWAP_VERSION=latest

#################### BUILDER STAGE ####################
FROM ${BUILD_BASE_IMAGE} AS builder

ARG PYTHON_VERSION
ARG VLLM_VERSION
ARG LLAMA_SWAP_VERSION

ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
        curl ca-certificates git \
        python3-pip python3-dev \
    && rm -rf /var/lib/apt/lists/*

# Install uv for fast Python package management
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && $HOME/.local/bin/uv venv /opt/venv --python ${PYTHON_VERSION} \
    && ln -s /opt/venv/bin/python3 /usr/bin/python3 \
    && ln -s /opt/venv/bin/pip /usr/bin/pip

ENV PATH="/opt/venv/bin:/root/.local/bin:$PATH"
ENV VIRTUAL_ENV="/opt/venv"

# Install vLLM (latest or specific version)
RUN --mount=type=cache,target=/root/.cache/uv \
    if [ -n "${VLLM_VERSION}" ]; then \
        echo "Installing vLLM ${VLLM_VERSION}" \
        && uv pip install vllm==${VLLM_VERSION}; \
    else \
        echo "Installing latest vLLM" \
        && uv pip install vllm; \
    fi

# Download llama-swap release binary
RUN mkdir -p /install/bin \
    && ARCH=$(uname -m) \
    && case "$ARCH" in \
        x86_64) ARCH="amd64" ;; \
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
    && echo "${LS_VER}" > /install/llama-swap-version

#################### RUNTIME STAGE ####################
FROM ${FINAL_BASE_IMAGE} AS runtime

ARG CUDA_VERSION

ENV DEBIAN_FRONTEND=noninteractive
ENV LD_LIBRARY_PATH="/usr/local/cuda/lib64:${LD_LIBRARY_PATH}"
ENV PATH="/usr/local/bin:${PATH}"
ENV VIRTUAL_ENV="/opt/venv"
ENV PATH="/opt/venv/bin:${PATH}"

# Install runtime dependencies
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
        libgomp1 \
        python3 \
        curl \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy Python virtual environment with vLLM
COPY --from=builder /opt/venv /opt/venv

# Copy llama-swap binary
COPY --from=builder /install/bin/llama-swap /usr/local/bin/
COPY --from=builder /install/llama-swap-version /tmp/

# CUDA stub drivers for container compatibility
COPY --from=builder /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so || true
COPY --from=builder /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1 || true

RUN ldconfig

# Create directories
RUN mkdir -p /etc/llama-swap/config \
    && mkdir -p /models \
    && mkdir -p /app

# Copy example config
COPY config.example.yaml /etc/llama-swap/config/config.yaml

# Version tracking
RUN echo "vllm: $(python3 -c 'import vllm; print(vllm.__version__)' 2>/dev/null || echo 'unknown')" > /versions.txt \
    && echo "llama-swap: $(cat /tmp/llama-swap-version)" >> /versions.txt \
    && echo "cuda: ${CUDA_VERSION}" >> /versions.txt \
    && echo "build_timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> /versions.txt

WORKDIR /models

EXPOSE 8080

ENTRYPOINT ["llama-swap"]
CMD ["-config", "/etc/llama-swap/config/config.yaml", "-listen", "0.0.0.0:8080"]
