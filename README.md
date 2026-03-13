# dgx-llm-serve

**NOTE**: All contents in here was auto-translated by GLM 4.7.

LLM inference backend configuration collection for [NVIDIA DGX Spark](https://marketplace.nvidia.com/en-us/enterprise/personal-ai-supercomputers/dgx-spark/) and OEM machines.

> **Note**: This repository is specifically for DGX Spark / OEM machines. Operation on other environments is not expected.

## Target Hardware

- [NVIDIA DGX Spark](https://marketplace.nvidia.com/en-us/enterprise/personal-ai-supercomputers/dgx-spark/)
- OEM machines (e.g., [Lenovo ThinkStation PGX](https://www.lenovo.com/us/en/p/workstations/thinkstation-p-series/lenovo-thinkstation-pgx-sff/30kl0002us))

### Verified Environment

- Lenovo ThinkStation PGX

## Backend List

| Backend | Technology | Supported Models | Features |
|---------|-----------|------------------|----------|
| [trtllm](backends/trtllm/) | TensorRT-LLM | Qwen3-FP4, Nemotron-NVFP4 | Multi-model concurrent deployment |
| [vllm](backends/vllm/) | vLLM | Qwen3-Coder, Nemotron, Nemotron-VL | Tool calling support |
| [nim](backends/nim/) | NVIDIA NIM | Qwen3-32B, Llama-3.1-8B, Nemotron-Nano | NGC managed images |

## Prerequisites

- DGX Spark or OEM machine (GB10 Grace Blackwell)
- Docker + Docker Compose
- NVIDIA Container Toolkit
- Model weights: Configure in `~/model_weights/` (except NIM)

## Quick Start

```bash
# TensorRT-LLM (Qwen3-FP4 standalone)
cd backends/trtllm && docker compose --profile qwen up

# TRT-LLM multi-model (Qwen3-FP4 + Nemotron-NVFP4 simultaneous deployment on single port)
cd backends/trtllm && docker compose --profile multi up

# vLLM (Qwen3-Coder)
cd backends/vllm && docker compose --profile qwen up

# NVIDIA NIM (Qwen3-32B)
cd backends/nim && docker compose up
```

## API Test

OpenAI-compatible APIs are exposed on port 8000 for all backends.

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<MODEL_NAME>",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 128
  }'
```

## Documentation

- [Thinking Mode](docs/thinking-mode.md) - Qwen3 thinking process output
- [Tool Calling](docs/tool-calling.md) - vLLM tool calling configuration and debugging

## Security Notes

This repository is intended for personal use and local execution.

### Default Settings

- **Port binding**: `127.0.0.1:8000` (local host only)
- **API authentication**: None (local execution assumed)

### Access from Other Devices on LAN

Change the port configuration in each `compose.yml`:

```yaml
# Before (local only)
ports:
  - "127.0.0.1:8000:8000"

# After (LAN public)
ports:
  - "8000:8000"
```

**Note**: When exposing to LAN, verify:
- Router blocks external (internet) access to port 8000
- Only trusted devices within the LAN can access

### Note on Remote Code Execution

vLLM / TRT-LLM Nemotron models (`--trust-remote-code` / `--trust_remote_code` flags) allow code execution from HuggingFace:
- Supply chain attack risk exists
- Recommended to verify code in `~/.cache/huggingface` on first model download
