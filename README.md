```
POST /v1/chat/completions
Content-Type: application/json
```

The API accepts standard OpenAI-compatible chat messages, including text, images, and audio, and returns generated text, with full support for streaming, function calling, JSON mode, and reasoning/thinking.

### Example Request (cURL)

Below is a complete example of how to interact with the API, demonstrating how to use advanced parameters like `enable_thinking`, `seed`, and `max_tokens`.

```bash
curl -X POST "https://aifoundry.discretal.com/slm/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful AI assistant."
      },
      {
        "role": "user",
        "content": "Explain quantum computing in simple terms."
      }
    ],
    "temperature": 1.0,
    "top_p": 0.95,
    "seed": 42,
    "max_tokens": 2048,
    "stream": false,
    "enable_thinking": true
  }'
```

### Request Body

#### `messages` (required)

An array of message objects forming the conversation. Each message has the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `role` | `string` | ✅ Yes | The role of the message author. One of: `system`, `user`, or `assistant`. |
| `content` | `string` or `array` | ✅ Yes | The content of the message. Can be a plain text string **or** a multimodal content array (see below). |
| `name` | `string` | No | An optional name for the participant. Useful to differentiate multiple users in multi-turn conversations. |

#### System Prompts (The `system` role)

The first message in the `messages` array should typically have the `system` role. This is where you define the model's persona, provide global instructions, and specify output constraints. A highly detailed system prompt is the most effective way to steer the model's behavior.

**Basic Example:**
```json
{"role": "system", "content": "You are a helpful AI assistant."}
```

**Detailed Example:**
```json
{
  "role": "system", 
  "content": "You are an expert Python software engineer. Always write clean, well-documented code. When providing code snippets, include comments explaining complex logic. Do not provide explanations unless explicitly asked. If you don't know the answer, say 'I don't know' instead of hallucinating."
}
```

#### Multimodal Messages (The `user` role)

**Plain text message:**

```json
{"role": "user", "content": "What is the capital of Saudi Arabia?"}
```

**Multimodal message (text + image):**

Use a content array when sending images alongside text. This is how the Vision capabilities of the model are accessed through the chat endpoint.

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "What document is this? Extract all text."},
    {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,/9j/4AAQ..."}}
  ]
}
```

> **Tip:** Image URLs must be Base64-encoded data URIs in the format `data:image/<format>;base64,<data>`. External HTTP URLs are not supported.

---

#### Sampling Parameters

| Parameter | Type | Required | Default | Range | Description |
|-----------|------|----------|---------|-------|-------------|
| `temperature` | `float` | No | `1.0` | `0.0` – `2.0` | Controls randomness. Lower values (e.g., `0.1`) make output more deterministic and focused. Higher values (e.g., `1.5`) make output more creative and diverse. |
| `top_p` | `float` | No | `0.95` | `0.0` – `1.0` | Nucleus sampling: only tokens comprising the top `p` cumulative probability are considered. Use `top_p` **or** `temperature`, not both aggressively. |
| `top_k` | `int` | No | `64` | `≥ 0` | Limits token selection to the top K most probable candidates at each step. Set to `0` to disable. |
| `min_p` | `float` | No | — | `0.0` – `1.0` | Minimum probability threshold relative to the most likely token. Tokens below `min_p × max_prob` are filtered out. More adaptive than `top_k`. |
| `max_tokens` | `int` | No | `2048` | `≥ 1` | Maximum number of tokens the model will generate in its response. Set higher (e.g., `4096`) for long-form outputs, large JSON extractions, or tasks requiring extensive reasoning. |
| `stop` | `string` or `array` | No | — | — | One or more sequences that will cause the model to stop generating. Example: `["\n\n", "END"]`. |
| `seed` | `int` | No | — | — | A fixed random seed for deterministic generation. When set, repeated requests with the same `seed` and parameters will produce identical outputs (best used with `temperature=0`). |

#### Penalty Parameters

| Parameter | Type | Required | Default | Range | Description |
|-----------|------|----------|---------|-------|-------------|
| `presence_penalty` | `float` | No | `0.0` | `-2.0` – `2.0` | Positive values penalize tokens that have already appeared in the text, encouraging the model to talk about new topics. |
| `frequency_penalty` | `float` | No | `0.0` | `-2.0` – `2.0` | Positive values penalize tokens proportionally to how often they have appeared, reducing verbatim repetition. |
| `logit_bias` | `object` | No | — | — | A JSON object mapping token IDs (as strings) to bias values (`-100` to `100`). Use to increase or decrease the likelihood of specific tokens appearing. Example: `{"1234": -100}` bans token 1234. |

#### Output Control

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `stream` | `bool` | No | `false` | If `true`, the response is delivered as a stream of Server-Sent Events (SSE). Each event contains a partial token. The stream ends with `data: [DONE]`. |
| `response_format` | `object` | No | — | Force structured output. See [Response Format](#response-format) section below. |

#### Tools & Function Calling

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `tools` | `array` | No | — | A list of tool definitions the model may call. Each tool has `type` (`"function"`) and a `function` object with `name`, `description`, and `parameters` (JSON Schema). |
| `tool_choice` | `string` or `object` | No | — | Controls tool selection: `"auto"` (model decides), `"none"` (never call), or `{"type": "function", "function": {"name": "my_func"}}` to force a specific tool. |

#### Reasoning / Thinking

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `enable_thinking` | `bool` | No | — | When set to `true`, the model generates an internal chain-of-thought reasoning block before producing its final answer. This typically improves accuracy for complex tasks (math, logic, analysis). |
| `include_reasoning` | `bool` | No | `true` | Controls whether the reasoning content is included in the API response. Set to `false` if you want the model to think internally but only return the clean final answer to the client. The Gateway strips the reasoning server-side. |

---

#### Response Format

To force the model to return valid JSON (free-form structure):

```json
{
  "response_format": { "type": "json_object" }
}
```

To enforce a specific JSON schema (the model's output must conform to this structure):

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "my_schema",
      "schema": {
        "type": "object",
        "properties": {
          "answer": { "type": "string" },
          "confidence": { "type": "number" }
        },
        "required": ["answer", "confidence"]
      }
    }
  }
}
```

### cURL Example — Basic

```bash
curl -X POST "https://aifoundry.discretal.com/slm/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful AI assistant."},
      {"role": "user", "content": "What is the capital of Saudi Arabia?"}
    ],
    "temperature": 0.7,
    "stream": false
  }'
```

### cURL Example — With Thinking

```bash
curl -X POST "https://aifoundry.discretal.com/slm/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -d '{
    "messages": [
      {"role": "user", "content": "Solve: If 3x + 7 = 22, what is x?"}
    ],
    "enable_thinking": true,
    "include_reasoning": false,
    "temperature": 0.7,
    "stream": false
  }'
```

> In this example, the model will reason internally but only the final answer is returned to you.

### cURL Example — Streaming

```bash
curl -X POST "https://aifoundry.discretal.com/slm/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -d '{
    "messages": [
      {"role": "user", "content": "Write a short poem about the desert."}
    ],
    "stream": true
  }'
```

### cURL Example — JSON Mode

```bash
curl -X POST "https://aifoundry.discretal.com/slm/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -d '{
    "messages": [
      {"role": "user", "content": "List 3 planets and their distances from the sun in km."}
    ],
    "response_format": {"type": "json_object"},
    "temperature": 0.3
  }'
```

### Python Example (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(
    api_key="<YOUR_API_KEY>",
    base_url="https://aifoundry.discretal.com/slm/v1"
)

# Non-streaming
response = client.chat.completions.create(
    model="any",  # model is injected server-side, this value is ignored
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing in simple terms."}
    ],
    temperature=0.7
)
print(response.choices[0].message.content)

# Streaming
stream = client.chat.completions.create(
    model="any",
    messages=[{"role": "user", "content": "Tell me a story."}],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### Python Example (AsyncOpenAI SDK)

For high-performance, concurrent applications, you can use the asynchronous client:

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key="<YOUR_API_KEY>",
    base_url="https://aifoundry.discretal.com/slm/v1"
)

async def main():
    # Non-streaming
    response = await client.chat.completions.create(
        model="any",
        messages=[{"role": "user", "content": "Explain quantum computing in simple terms."}],
        temperature=0.7
    )
    print(response.choices[0].message.content)

    # Streaming
    stream = await client.chat.completions.create(
        model="any",
        messages=[{"role": "user", "content": "Tell me a story."}],
        stream=True
    )
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)

if __name__ == "__main__":
    asyncio.run(main())
```

### Response Structure

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1783944215,
  "model": "unsloth/gemma-4-E4B-it-GGUF:UD-Q8_K_XL",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The capital of Saudi Arabia is Riyadh."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 29,
    "completion_tokens": 12,
    "total_tokens": 41
  }
}
```

---
