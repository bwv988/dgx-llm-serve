# vLLM Tool Calling Guide

A comprehensive guide for configuring tool calling (Function Calling) with vLLM and debugging internal prompts.

## Enabling Tool Calling

### vLLM Server Startup Command

```bash
vllm serve Qwen/Qwen3-Coder-8B-Instruct \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder
```

### Docker Compose Configuration Example

```yaml
services:
  vllm:
    image: nvcr.io/nvidia/vllm:26.01-py3
    command: >
      vllm serve Qwen/Qwen3-Coder-8B-Instruct
      --enable-auto-tool-choice
      --tool-call-parser hermes
```

---

## How to Inspect Internal Prompts

### Method 1: echo=true Parameter (Recommended)

By adding `echo: true` to the API request, each token in the internal prompt is returned in the `prompt_logprobs` field.

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-Coder-30B-A3B-Instruct",
    "messages": [{"role": "user", "content": "What is the weather in Tokyo?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string", "description": "City name"}
          },
          "required": ["location"]
        }
      }
    }],
    "tool_choice": "auto",
    "echo": true
  }' > /tmp/echo_response.json
```

Format and display the internal prompt:

```bash
python3 -c "
import json
with open('/tmp/echo_response.json') as f:
    data = json.load(f)
tokens = []
for item in data['prompt_logprobs']:
    if item is None:
        continue
    for key, value in item.items():
        if 'decoded_token' in value:
            tokens.append(value['decoded_token'])
print(''.join(tokens))
"
```

### Method 2: Offline Verification with apply_chat_template

You can verify prompts locally without starting vLLM.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen3-Coder-8B-Instruct")

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What's the weather in Tokyo?"}
]

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather information",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"}
                },
                "required": ["city"]
            }
        }
    }
]

formatted_prompt = tokenizer.apply_chat_template(
    messages,
    tools=tools,
    tokenize=False,
    add_generation_prompt=True
)
print(formatted_prompt)
```

### Method 3: /tokenize Endpoint

```bash
curl -X POST http://localhost:8000/tokenize \
    -H "Content-Type: application/json" \
    -d '{
        "model": "Qwen/Qwen3-Coder-8B-Instruct",
        "messages": [
            {"role": "user", "content": "What is the weather in Tokyo?"}
        ]
    }'
```

### Method 4: Direct chat_template Verification

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen3-Coder-8B-Instruct")
print(tokenizer.chat_template)
```

---

## Method Comparison

| Method | Requires vLLM | Difficulty | Accuracy | Use Case |
|--------|--------------|------------|----------|----------|
| `echo=true` | Yes | Low | High | Runtime debugging |
| `apply_chat_template` | No | Low | High | Development-time verification |
| `/tokenize` API | Yes | Medium | High | Verification via API |
| Direct chat_template check | No | Low | - | Template understanding |

---

## Note: Debug Log Environment Variables

The following environment variables don't support internal prompt output:

| Environment Variable | Result |
|---------------------|--------|
| `VLLM_LOGGING_LEVEL=DEBUG` | Batch execution info, engine state, etc. are output |
| `VLLM_DEBUG_LOG_API_SERVER_RESPONSE=TRUE` | For API response log output |

**Use Method 1 (echo=true) or Method 2 (apply_chat_template) for internal prompt verification.**

---

## Tool Calling Format

Qwen3-Coder outputs tool calls in the following format:

```
`
{"name": "get_weather", "arguments": {"city": "Tokyo"}}
` 
```

---

## References

- [vLLM Tool Calling Documentation](https://docs.vllm.ai/en/latest/features/tool_calling/)
- [vLLM Environment Variables List](https://docs.vllm.ai/en/stable/configuration/env_vars/)
- [Qwen Function Calling Guide](https://qwen.readthedocs.io/en/latest/framework/function_call.html)
- [HuggingFace Chat Templates Documentation](https://huggingface.co/docs/transformers/en/chat_templating)