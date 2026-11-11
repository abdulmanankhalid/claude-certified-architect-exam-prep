# Extended Thinking: Giving Claude a Reasoning Budget

## What you will learn

- What extended thinking is and how it differs from asking for step-by-step reasoning in the prompt
- How to enable it with `thinking` and `budget_tokens`, and how the budget relates to `max_tokens`
- How thinking blocks appear in responses, streaming, and multi-turn conversations
- Why thinking blocks must be passed back unchanged during tool use
- When thinking is worth its cost, and which settings conflict with it

## Explanation

**Extended thinking** lets Claude produce an internal reasoning phase before its final answer. The reasoning arrives as separate `thinking` content blocks, ahead of the normal `text` block. Lesson 012 covered prompt-based reasoning (`<thinking>` tags you ask for). Extended thinking is the API feature: the output is returned in a dedicated block type.

### Enabling it

```python
response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[{"role": "user", "content": "Plan a zero-downtime database migration."}],
)

for block in response.content:
    if block.type == "thinking":
        print("REASONING:", block.thinking)
    elif block.type == "text":
        print("ANSWER:", block.text)
```

Key rules:

- `budget_tokens` is the maximum Claude may spend thinking. It is a ceiling, not a guarantee, and Claude may use less.
- `budget_tokens` must be lower than `max_tokens`, because thinking tokens count toward the output limit. Leave room for the answer itself.
- There is a minimum budget (about 1,024 tokens). Start small and raise it only if evals improve.
- Thinking tokens are **billed as output tokens**.
- Check the current docs for model-specific behavior (some newer models offer adaptive or effort-based thinking controls). The exam concept stays the same: you trade tokens and latency for reasoning quality.

### Response shape

```json
{
  "content": [
    {"type": "thinking", "thinking": "First, consider...", "signature": "EuYBCkQY..."},
    {"type": "text", "text": "Here is the plan..."}
  ],
  "stop_reason": "end_turn"
}
```

The `signature` lets the API verify the thinking block was produced by Claude and not edited. Some thinking may come back as an encrypted `redacted_thinking` block. You pass those back as-is too.

### Streaming

Thinking arrives as `thinking_delta` events, followed by `text_delta` events for the answer. With large budgets, use streaming: long non-streamed requests risk timeouts, and users see progress sooner.

### Thinking with tool use

This is the most tested detail. When Claude thinks and then calls a tool, your next request must include the **assistant turn exactly as returned**, thinking block included, followed by your `tool_result`:

```python
messages.append({"role": "assistant", "content": response.content})  # unchanged
messages.append({"role": "user", "content": [tool_result]})
```

If you drop or modify the thinking block mid tool loop, the API returns an error or reasoning continuity is lost. Do not edit, reorder, or strip it until the tool loop for that turn is complete.

### Constraints and interactions

- **Sampling:** thinking is not compatible with modified `temperature` or `top_k`, and `top_p` is restricted. Leave them at defaults.
- **Forced tool choice:** forcing a specific tool (or "any") with `tool_choice` is not allowed with thinking. Only `auto` or `none` work. This matters for the structured-output pattern from lesson 012: use `auto` plus a clear instruction, or skip thinking for that call.
- **Prompt caching:** changing the thinking budget invalidates cached message prefixes (system prompts and tool definitions stay cached). Keep the budget stable across calls that share a cache (lesson 003).
- **Prefilling** the assistant turn is not supported with thinking.

### When to use it

Good fits: multi-step math, complex coding and debugging, planning, analysis with many constraints, agent steps where a wrong decision is expensive.

Poor fits: classification, extraction, simple lookups, latency-sensitive chat. The extra output tokens cost money and time (lesson 011). Decide with an eval (lesson 007): run the same test set with and without thinking, and compare accuracy against cost.

### Prompting tips

- Give high-level guidance ("think carefully about edge cases") rather than scripting each reasoning step.
- Do not paste the thinking back into the prompt as examples.
- Do not treat raw thinking as a verified explanation. It is useful for debugging, not a guarantee of faithful reasoning.

## Common exam traps

1. **Setting `budget_tokens` >= `max_tokens`.** Invalid. The budget is part of the output limit.
2. **Thinking is free.** It is billed as output tokens.
3. **Removing the thinking block during a tool loop.** Pass the assistant content back unchanged.
4. **Combining thinking with a forced `tool_choice`.** Not supported; only `auto` or `none`.
5. **Treating a larger budget as always better.** Returns diminish, and cost and latency rise. Measure.
6. **Assuming thinking replaces evals or validation.** It improves reasoning but does not guarantee correctness.

## Practice questions

<details>
<summary>Question 1: A request sets <code>max_tokens=4000</code> and <code>budget_tokens=6000</code>. What happens, and how do you fix it?</summary>

The request is rejected, since the thinking budget must be smaller than `max_tokens` (thinking counts toward output). Raise `max_tokens` above the budget, leaving room for the final answer (for example `max_tokens=10000`, `budget_tokens=6000`), or lower the budget.

</details>

<details>
<summary>Question 2: Your agent uses extended thinking and tools. After a <code>tool_use</code> stop, a developer rebuilds the assistant message from only the <code>text</code> and <code>tool_use</code> blocks. Requests start failing. Why?</summary>

The thinking block (with its signature) was dropped. During a tool-use turn you must send back the full assistant content exactly as received, including `thinking` or `redacted_thinking` blocks, followed by the `tool_result`. Fix it by appending `response.content` unchanged.

</details>

<details>
<summary>Question 3: A team wants to enable thinking on a high-volume ticket-classification endpoint, and also force a classification tool with <code>tool_choice</code>. What do you advise?</summary>

Advise against it. Classification is a simple task where thinking adds cost and latency with little gain, and forced tool choice is incompatible with thinking. Keep thinking off and use the forced tool for reliable structured output. Only enable thinking if an eval shows a meaningful accuracy gain on hard cases, perhaps routed to a separate path.

</details>

## Key takeaways

- Extended thinking is a separate reasoning phase returned as `thinking` blocks, enabled with `thinking={"type": "enabled", "budget_tokens": N}`.
- The budget must be below `max_tokens`, and thinking tokens are billed as output.
- In tool loops, return thinking blocks unchanged with the tool result.
- Not compatible with forced `tool_choice`, prefilling, or custom temperature; changing the budget breaks message-level cache hits.
- Use it for hard reasoning tasks, stream large budgets, and justify it with evals.
