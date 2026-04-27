# vllm-swap

A unified Docker container combining [vLLM](https://github.com/vllm-project/vllm) inference engine with [llama-swap](https://github.com/mostlygeek/llama-swap) model swapping proxy.

## Features

- **vLLM**: High-throughput LLM inference with OpenAI-compatible API
- **llama-swap**: On-demand model loading/unloading to save GPU memory
- **CUDA 13**: Built on NVIDIA CUDA 13.0.2 for latest GPU support
- **Auto-build**: Daily automated builds at 03:00 UTC

## Usage

### Docker Run

```bash
docker run -d \
  --name vllm-swap \
  --gpus all \
  -p 8080:8080 \
  -v /path/to/models:/models \
  -v /path/to/config.yaml:/etc/llama-swap/config/config.yaml \
  sebrinass/vllm-swap:latest
```

### Docker Compose

```yaml
services:
  vllm-swap:
    image: sebrinass/vllm-swap:latest
    container_name: vllm-swap
    runtime: nvidia
    ports:
      - "8080:8080"
    volumes:
      - /path/to/models:/models
      - ./config.yaml:/etc/llama-swap/config/config.yaml
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
```

## Configuration

Edit `config.yaml` to define your models:

```yaml
models:
  "qwen-14b":
    ttl: 600
    cmd: >
      vllm serve /models/Qwen2.5-14B-Instruct
      --port ${PORT}
      --tensor-parallel-size 1
```

## Build Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `CUDA_VERSION` | 13.0.2 | CUDA base image version |
| `VLLM_VERSION` | (latest) | vLLM PyPI version |
| `LLAMA_SWAP_VERSION` | latest | llama-swap release version |

## Architecture

Only `linux/amd64` (x86_64) is supported.
