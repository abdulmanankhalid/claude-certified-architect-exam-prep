# Messages API Basics

## What you will learn

- The shape of a Messages API request and response
- Why the API is stateless and what that means for conversations
- The role of `system`, `max_tokens`, and `stop_reason`
- How to control output with `temperature` and how to stream responses

## Explanation

The Messages API (`POST /v1/messages`) is the core way to talk to Claude. Almost every other feature (tool use, caching, agents) is built on it.

### A minimal request

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY

response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1024,
    system="You are a concise assistant for cloud architects.",
    messages=[
        {"role": "user", "content": "Explain eventual consistency in two sentences."}
    ],
)
print(response.content[0].text)
```

Key fields:

- `model`: which Claude model to use.
- `max_tokens`: a required hard cap on output tokens. It is a limit, not a target.
- `system`: top-level instructions (role, tone, rules). It is a separate parameter, not a message with role `system`.
- `messages`: a list of turns with role `user` or `assistant`. Roles should alternate, and the conversation starts with a user turn.

### The response

The response contains:

- `content`: a list of content blocks (for example `text`, and `tool_use` when tools are involved). Always treat it as a list.
- `stop_reason`: why generation stopped.
  - `end_turn`: Claude finished naturally.
  - `max_tokens`: output was cut off by your limit.
  - `stop_sequence`: a custom stop string was produced.
  - `tool_use`: Claude wants you to run a tool.
- `usage`: `input_tokens` and `output_tokens`, which drive cost.

### Statelessness

The API remembers nothing between calls. To continue a conversation, you resend the full history every time:

```python
history = [{"role": "user", "content": "My name is Sam."}]
r = client.messages.create(model="claude-sonnet-5-5", max_tokens=200, messages=history)
history.append({"role": "assistant", "content": r.content[0].text})
history.append({"role": "user", "content": "What is my name?"})
r2 = client.messages.create(model="claude-sonnet-5-5", max_tokens=200, messages=history)
```

Consequences: input tokens grow each turn, you pay for resent history (prompt caching helps), and you must stay inside the context window.

### Prefilling and sampling

- `temperature` (0 to 1) controls randomness. Lower is more deterministic, good for extraction and classification. Even at 0 outputs are not guaranteed identical.
- Some newer models restrict or ignore certain sampling parameters, so check the current docs for the model you use.

### Streaming

Set `stream=True` (or use the SDK stream helper) to receive tokens as server-sent events. This improves perceived latency for user-facing apps. Total generation time is about the same; time to first token is much better. Long requests should stream to avoid timeouts.

### Multimodal content

`content` can be a string or a list of blocks, so a user turn can mix `text` and `image` blocks.

## Common exam traps

- Putting the system prompt inside `messages` with role `system`. It is a top-level `system` parameter.
- Assuming the API keeps conversation state. It does not; you resend history.
- Forgetting `max_tokens` is required, or assuming Claude will fill it.
- Not checking `stop_reason`. A `max_tokens` stop means a truncated answer (for example broken JSON).
- Assuming `response.content` is a string. It is a list of blocks.
- Believing `temperature=0` guarantees identical output.

## Practice questions

1. A chatbot forgets the user's name on the second request. What is the most likely cause?
2. Your JSON output is occasionally invalid and cut off mid-object. Which field do you check, and what do you change?
3. Where do you put instructions such as "Always answer in French"?

<details>
<summary>Answers</summary>

1. The app only sent the latest message. The API is stateless, so the full history (user and assistant turns) must be sent each call.
2. Check `stop_reason`. If it is `max_tokens`, raise `max_tokens` (or shorten the requested output) and handle truncation in code.
3. In the top-level `system` parameter.

</details>

## Key takeaways

- Messages API = `model` + `max_tokens` + `messages` (+ optional `system`).
- It is stateless: you manage and resend the conversation.
- Responses are lists of content blocks; always inspect `stop_reason`.
- Use streaming for responsiveness; tune `temperature` for determinism.
