# Qwen3 Thinking Mode

The Qwen3 model includes a Thinking mode that outputs the "thinking process" during inference.

## Overview

- **Default**: Enabled (Thinking mode)
- **Output format**: Thinking process is included in the response with `
thought process is separated into the `reasoning_content` field
- **Supported backends**: TensorRT-LLM, vLLM (Qwen3 family models)

## Thinking Mode (Enabled)

Default behavior. The model outputs the thinking process before giving the answer.

### Request Example

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Qwen3-30B-A3B-FP4",
    "messages": [{"role": "user", "content": "What is 2+2?"}],
    "max_tokens": 256
  }'
```

### Response Example

```
thought process is separated into the `reasoning_content` field
The user is asking for a simple arithmetic calculation.
2 + 2 = 4
</think>
The answer is 4.
```

## Non-thinking Mode (Disabled)

Adding `/no_think` to the system prompt makes the model output only the answer without the thinking process.

### Request Example

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Qwen3-30B-A3B-FP4",
    "messages": [
      {"role": "system", "content": "/no_think"},
      {"role": "user", "content": "What is 2+2?"}
    ],
    "max_tokens": 128
  }'
```

### Response Example

```
The answer is 4.
```

## Client-Side Processing

When using Thinking mode, you need to remove the `
tags from the response.

### Python Example

```python
import re

def remove_thinking(response: str) -> str:
    """Remove `
 tags from response."""
    return re.sub(r'`thought` tags are removed, client-side implementation is required