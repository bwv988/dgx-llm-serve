# vLLM Backend

vLLM-based LLM serving environment.

## Supported Models

| Profile | Model | Features |
|---------|-------|----------|
| qwen | Qwen3-Coder-30B-A3B-Instruct | Tool calling support |
| qwen35 | Qwen3.5-35B-A3B-FP8 | Gated DeltaNet + MoE, FP8 quantization, thinking mode (reasoning_content separation), text-only mode (vision disabled) |
| nemotron | NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4 | Fast inference |
| nemotron-vl | NVIDIA-Nemotron-Nano-12B-v2-VL | Multimodal (image support) |
| multi | Qwen3-Coder + Nemotron simultaneous startup | Single port with OpenResty proxy |

## Startup

```bash
# Qwen3-Coder (tool calling support)
docker compose --profile qwen up

# Qwen3.5 (thinking mode support)
docker compose --profile qwen35 up

# Nemotron
docker compose --profile nemotron up

# Nemotron-VL (multimodal)
docker compose --profile nemotron-vl up

# Multi-model (Qwen3-Coder + Nemotron simultaneous startup)
docker compose --profile multi up
```

## Multi-Model Startup

The `multi` profile integrates 2 models on port 8000 via OpenResty proxy.
Automatic routing based on the `model` field in the request body.

| Model | GPU Memory | Routing Condition |
|-------|-----------|-------------------|
| Qwen3-Coder-30B-A3B (bf16) | 50% | `model` contains "qwen" |
| Nemotron-30B-A3B (NVFP4) | 25% | `model` contains "nemotron" |

```bash
# List available models
curl http://localhost:8000/v1/models

# Qwen3-Coder
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-Coder-30B-A3B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'

# Nemotron
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4", "messages": [{"role": "user", "content": "Hello"}]}'
```

### Troubleshooting

- **503 unhealthy**: Wait for both backends to finish starting (first start takes several minutes)
- **404 unknown model**: Specify the correct model name containing "qwen" or "nemotron" in the `model` field
- **GPU OOM**: Adjust `--gpu-memory-utilization` for each service
- **Tool calling**: Tool calling (with `--tool-call-parser`) for Qwen3-Coder is disabled in multi profile. After updating to driver 590+, both services can use the 26.01 image to enable it

### Forward Compat Constraint (Driver 580)

Driver 580 natively supports CUDA 13.0.2, and CUDA 13.1 (26.01) can only run **1 container at a time** via Forward Compat.

- The multi profile uses the Qwen (25.11 / CUDA 13.0.2) + Nemotron (26.01 / CUDA 13.1) combination to avoid this constraint
- 26.01 × 2 containers (e.g., Qwen3-FP4 + Nemotron) is **not possible** because Nemotron's flashinfer CUTLASS backend initialization fails
- With driver 590+ update, CUDA 13.1 will be natively supported and this constraint is expected to be resolved

## Configuration Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `--gpu-memory-utilization` | 0.9 | GPU memory utilization |
| `--max-model-len` | 32768 | Maximum context length |
| `--max-num-seqs` | 4 | Maximum concurrent sequence count |
| `--tensor-parallel-size` | 1 | Tensor parallel size |

## Tool Calling

To use tool calling with Qwen3-Coder:

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-Coder-30B-A3B-Instruct",
    "messages": [{"role": "user", "content": "What is the weather in Tokyo?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather information",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string"}
          },
          "required": ["location"]
        }
      }
    }],
    "tool_choice": "auto"
  }'
```

See [Tool Calling Guide](../../docs/tool-calling.md) for details.

## Environment Requirements

- NVIDIA GPU + nvidia-container-toolkit
- Model weights: Located in `~/model_weights/`