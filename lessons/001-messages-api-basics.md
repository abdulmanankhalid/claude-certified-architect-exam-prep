# The Messages API: Requests, Responses, and the Conversation Loop

## What you will learn

- The required parts of a Messages API request
- How `system`, `user`, and `assistant` roles work, and why the API is stateless
- How to read a response: `content` blocks, `stop_reason`, and `usage`
- How to build a multi-turn conversation and control output length

## Explanation

The Messages API (`POST /v1/messages`) is the core way to talk to Claude. Almost every other feature (tool use, caching, agents) is built on it.

### A minimal request

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment

response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=500,
    system="You are a concise assistant for cloud architects.",
    messages=[{"role": "user", "content": "What is a queue used for?"}],
)
print(response.content[0].text)
```

Required fields: `model`, `max_tokens`, and `messages`. `system` is optional but is the right place for role, tone, and rules.

### Roles

- `user`: input from the human or your application.
- `assistant`: Claude's earlier replies (or text you prefill).
- The system prompt is a **top-level parameter**, not a message with a `system` role.

Messages should alternate between `user` and `assistant`, and the conversation must be sent in order.

### The API is stateless

Claude does not remember earlier calls. To continue a conversation, you resend the full history every time:

```python
history = [{"role": "user", "content": "My name is Sam."}]
r1 = client.messages.create(model="claude-sonnet-5-5", max_tokens=200, messages=history)

history.append({"role": "assistant", "content": r1.content[0].text})
history.append({"role": "user", "content": "What is my name?"})
r2 = client.messages.create(model="claude-sonnet-5-5", max_tokens=200, messages=history)
```

Consequence: input tokens grow each turn, so cost and latency grow too. Later lessons cover caching and context management to reduce this.

### Reading the response

```json
{
  "id": "msg_...",
  "role": "assistant",
  "content": [{"type": "text", "text": "A queue holds..."}],
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 25, "output_tokens": 80}
}
```

- `content` is a **list of blocks** (text now; also `tool_use` and others later). Never assume it is one string.
- `stop_reason` tells you why generation stopped:
  - `end_turn`: Claude finished naturally.
  - `max_tokens`: output was cut off by your limit. The answer may be incomplete.
  - `stop_sequence`: a custom stop string was hit.
  - `tool_use`: Claude wants you to run a tool (next lessons).
- `usage` reports input and output tokens; this is what you are billed on.

### Useful parameters

- `max_tokens`: a hard cap on output, not a target. Claude does not try to fill it.
- `temperature`: lower for extraction/classification, higher for creative work.
- `stop_sequences`: strings that end generation early.
- `stream=True`: receive tokens as they are generated, which improves perceived latency.

## Common exam traps

- **Putting the system prompt in `messages` as a `system` role.** It is a separate top-level field.
- **Assuming the server remembers the conversation.** You must resend history.
- **Treating `max_tokens` as a target length.** It is only a ceiling.
- **Ignoring `stop_reason: max_tokens`.** Truncated output may be silently used as if complete.
- **Reading `response.content` as a string.** It is a list of typed blocks.
- **Believing the model name is optional.** `model`, `max_tokens`, and `messages` are required.

## Practice questions

1. A chatbot forgets the user's name on the second request. Each call sends only the newest user message. What is the cause and the fix?
2. A JSON summary sometimes ends mid-sentence. What field do you check first, and what do you change?
3. Where do you put instructions such as "Always answer in French"?

<details>
<summary>Answers</summary>

1. The API is stateless. Fix it by appending each user and assistant message to a list and sending the whole list on every request.
2. Check `stop_reason`. If it is `max_tokens`, raise `max_tokens` (or ask for shorter output) and handle truncation in code.
3. In the top-level `system` parameter.

</details>

## Key takeaways

- Required: `model`, `max_tokens`, `messages`; `system` is a top-level field.
- The API is stateless; you own the conversation history.
- `content` is a list of blocks; always check `stop_reason` and `usage`.
- `max_tokens` is a ceiling, and growing history raises cost, which motivates caching and context management.
