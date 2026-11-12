# Module 1 — Working with the API

**Lessons:** accessing the API · making a request · multi-turn conversations · chatbot exercise · system prompts · system prompt exercise · temperature · streaming · controlling model output · structured data · structured data exercise · quiz

## Key concepts

| Concept | What it is |
|---|---|
| Messages / turns | Alternating `user` / `assistant` messages. The API is **stateless**: you resend the full history on every call |
| System prompt | Role, rules and context, separate from the user turns |
| Temperature | Randomness (0–1). Low = deterministic/factual, high = creative/varied |
| Streaming | Tokens returned incrementally as they are generated |
| Prefill | You write the beginning of the assistant turn to steer format/content |
| Stop sequences | Generation stops when the model emits a given string |
| Structured data | Clean JSON/code without explanatory text: prefill (e.g. `` ```json ``) + stop sequence (`` ``` ``) |

## Two ways to call Claude on Bedrock

| | `InvokeModel` | `Converse` |
|---|---|---|
| Body format | Anthropic-native (Messages API + `"anthropic_version": "bedrock-2023-05-31"`) | Unified, same shape for every model |
| Change model without changing code | ❌ | ✅ |
| Streaming | `InvokeModelWithResponseStream` | `ConverseStream` |
| Model-specific features | Full access (raw payload) | Via `additionalModelRequestFields` |

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="eu-west-1")
MODEL_ID = "eu.anthropic.claude-..."   # inference profile ID

def chat(messages, system=None, temperature=1.0, stop_sequences=None):
    params = {
        "modelId": MODEL_ID,
        "messages": messages,
        "inferenceConfig": {"maxTokens": 1000, "temperature": temperature},
    }
    if system:
        params["system"] = [{"text": system}]
    if stop_sequences:
        params["inferenceConfig"]["stopSequences"] = stop_sequences
    resp = client.converse(**params)
    return resp["output"]["message"]["content"][0]["text"], resp

# Multi-turn: keep and resend the history
messages = [{"role": "user", "content": [{"text": "Hi, what's a good name for a dog?"}]}]
text, resp = chat(messages)
messages.append({"role": "assistant", "content": [{"text": text}]})
messages.append({"role": "user", "content": [{"text": "Something shorter"}]})

# Structured data: prefill + stop sequence
messages = [
    {"role": "user", "content": [{"text": "Generate 3 AWS CLI commands as JSON"}]},
    {"role": "assistant", "content": [{"text": "```json"}]},   # prefill
]
text, _ = chat(messages, stop_sequences=["```"])

# Streaming
stream = client.converse_stream(modelId=MODEL_ID, messages=messages[:1])
for event in stream["stream"]:
    if "contentBlockDelta" in event:
        print(event["contentBlockDelta"]["delta"]["text"], end="")
```

Response fields worth checking:

- `resp["usage"]` → `inputTokens`, `outputTokens` (cost).
- `resp["stopReason"]` → `end_turn` · `max_tokens` (truncated) · `stop_sequence` · `tool_use`.

## Common pitfalls

- Forgetting to append the assistant reply to the history → the model "forgets" the conversation.
- `stopReason = max_tokens` → truncated answer. Raise `maxTokens` or shorten the request; temperature has nothing to do with it.
- Streaming improves **time-to-first-token** (perceived latency), not total cost.
- Prefill is not supported by every model/mode (e.g. not with extended thinking) → check before relying on it.

---

[← Back to README](README.md)
