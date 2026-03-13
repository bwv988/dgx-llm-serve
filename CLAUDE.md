# AGENTS.md / CLAUDE.md

This file is the common guide for **all AI coding agents** working on this repository.
(Examples: Claude Code, Codex, Cursor, Cline, etc.)

## Basic Principles (Common to All Agents)

- Make minimal, targeted changes; avoid unrelated refactoring
- Do not blindly revert user-created changes
- Destructive operations (`rm -rf`, `git reset --hard`, etc.) only when explicitly requested
- After implementation, run verification commands where possible and report results
- When uncertain, avoid over-speculation and ask for short, specific clarification
- For new dependencies or external access, explicitly state the reason and obtain agreement

## Project Overview

Docker Compose managed monorepo for LLM inference backends optimized for NVIDIA DGX Spark OEM machines (TensorRT-LLM, vLLM, NVIDIA NIM).

## Directory Structure

```text
dgx-llm-serve/
├── backends/
│   ├── trtllm/    # TensorRT-LLM (Qwen3-FP4, Nemotron-NVFP4)
│   ├── vllm/      # vLLM (Qwen3-Coder, Qwen3.5, Nemotron, Nemotron-VL)
│   └── nim/       # NVIDIA NIM (DGX Spark)
├── artifacts/     # Benchmark results (aiperf output)
├── docs/          # Common documentation
└── scripts/       # Utility scripts
```

## Key Commands

### Starting Servers

```bash
# TensorRT-LLM (profile selection)
cd backends/trtllm && docker compose --profile qwen up
cd backends/trtllm && docker compose --profile nemotron up
cd backends/trtllm && docker compose --profile multi up

# vLLM (profile selection)
cd backends/vllm && docker compose --profile qwen up
cd backends/vllm && docker compose --profile qwen35 up
cd backends/vllm && docker compose --profile nemotron up
cd backends/vllm && docker compose --profile nemotron-vl up
cd backends/vllm && docker compose --profile multi up

# NVIDIA NIM
cd backends/nim && docker compose up
```

### Health Check

```bash
curl http://localhost:8000/health
```

### API Test

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "<MODEL_NAME>", "messages": [{"role": "user", "content": "Hello"}]}'

# MODEL_NAME examples:
#   TRT-LLM:  nvidia/Qwen3-30B-A3B-FP4, nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4
#   vLLM:     Qwen/Qwen3-Coder-30B-A3B-Instruct, Qwen/Qwen3.5-35B-A3B-FP8
#             nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4, nvidia/NVIDIA-Nemotron-Nano-12B-v2-VL-NVFP4-QAD
#   NIM:      Qwen/Qwen3-32B
```

### Benchmark

```bash
# Run benchmark with aiperf (output to artifacts/)
uv run scripts/benchmark.py --model <MODEL_NAME>
```

## Backend-Specific Notes

### TensorRT-LLM
- Image: `1.3.0rc4` (ARM64 support)
- Nemotron: `--backend _autodeploy` + `compile_backend: torch-cudagraph` (Mamba SSM compatible)
- multi profile: KV cache limit required in `qwen_multi.yaml` (OOM without it)
- SM120 `cudaErrorIllegalInstruction`: Fixed in 1.3.0rc3+. Qwen's `compile_backend: torch-cudagraph` removed
- Thinking mode: Enabled by default. Add `/no_think` to system prompt to disable
- Need to strip `` tags on client side

### vLLM
- Qwen3.5-35B-A3B-FP8: `qwen35` profile. Uses `vllm/vllm-openai:v0.17.1-cu130` (NGC 26.01 doesn't support `qwen3_5_moe`). `--reasoning-parser qwen3` separates thinking into `reasoning_content`. `--language-model-only` disables vision encoder (text-only mode). SM 12.1 selects TRITON Fp8 MoE backend automatically
- Tool calling support (Qwen3-Coder)
- Internal prompt inspection: Use `echo: true` parameter
- Config params: `--gpu-memory-utilization 0.9`, `--max-model-len 32768`
- multi profile: Qwen (25.11) + Nemotron (26.01) use different images. Tool calling disabled in multi
- Forward Compat constraint: Driver 580 limits 26.01 (CUDA 13.1) containers to 1 at a time. Cannot run 2 26.01 containers

### NIM
- Model: `qwen/qwen3-32b-dgx-spark:1.1.0-variant`
- Model included in container image (no host mount needed)
- NGC API key authentication required (`NGC_API_KEY` environment variable)
- Workspace volume mount not supported (NIM internal management)

## Environment Requirements

- Target: DGX Spark OEM (GB10 SoC, ARM64, 128 GiB integrated memory)
- NVIDIA GPU + nvidia-container-toolkit
- Docker + Docker Compose
- Python (managed with uv, for benchmark scripts)
- Model weights: Located in `~/model_weights/` (except NIM)

## Related Documentation

- `docs/thinking-mode.md`: Qwen3 Thinking mode
- `docs/tool-calling.md`: vLLM tool calling guide