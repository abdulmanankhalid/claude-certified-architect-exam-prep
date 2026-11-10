# Context Window Management: Budgeting, Trimming, and Compacting Long Conversations

## What you will learn

- What counts toward the context window and why output tokens matter too
- How to measure usage with token counting instead of guessing
- Strategies for long-running work: trimming, summarizing (compaction), clearing old tool results, and external memory
- How these strategies interact with prompt caching
- The common traps the exam likes to test

## Explanation

### The context window is a shared budget

The context window is the maximum number of tokens Claude can consider in one request. Everything counts: the system prompt, tool definitions, every earlier message, tool results, and the response Claude generates. The Messages API is stateless (see lesson 001), so you resend the whole history every turn, and the input grows with each turn.

```text
input tokens (system + tools + history + new message) + output tokens <= context window
```

`max_tokens` caps only the output. If the input leaves too little room, the request fails or the response is cut short (`stop_reason: "max_tokens"`).

### Measure, don't guess

Characters and words are poor proxies for tokens, and different models can tokenize the same text differently. Use the token counting endpoint to check a request before sending it:

```python
count = client.messages.count_tokens(
    model=MODEL,
    system=SYSTEM_PROMPT,
    tools=TOOLS,
    messages=messages,
)
print(count.input_tokens)
```

After a real call, `response.usage` reports the actual `input_tokens` and `output_tokens` (and cache fields, see lesson 003). Log these so you can see growth per turn.

### Bigger window is not always better

Even when content fits, a very full context has costs:

- **Cost and latency** grow with input size.
- **Focus** can degrade: relevant facts buried among stale tool output are easier to miss.
- **Stale data** (an old file version, a superseded plan) can mislead the model.

So the goal is not "fill the window" but "keep the right tokens in it."

### Strategies for long-running work

| Strategy | What it does | Trade-off |
|---|---|---|
| **Trimming** | Drop the oldest turns | Simple, but loses facts; keep tool_use/tool_result pairs intact |
| **Summarizing (compaction)** | Ask Claude to summarize older history, then replace it with the summary | Keeps key decisions; summary is lossy |
| **Clearing old tool results** | Remove bulky tool outputs (file dumps, search results) that are no longer needed | Cheap and targeted; Claude may need to re-fetch |
| **External memory** | Write notes or state to files or a database; read back only when needed | Survives across sessions and context resets; needs a read/write tool |
| **Subagents** | Delegate noisy work to a fresh context and return only a short result (lesson 009) | Extra calls, but keeps the main context clean |
| **Retrieval (RAG)** | Fetch only relevant snippets instead of loading whole documents | Depends on retrieval quality |

Claude Code applies the same ideas: it compacts the conversation when it grows large (and you can trigger it with `/compact`), and `CLAUDE.md` (lesson 006) is the durable memory that is reloaded each session.

### A simple compaction sketch

```python
def maybe_compact(messages, limit=150_000, keep_last=6):
    used = client.messages.count_tokens(
        model=MODEL, messages=messages
    ).input_tokens
    if used < limit:
        return messages

    old, recent = messages[:-keep_last], messages[-keep_last:]
    summary = client.messages.create(
        model=MODEL,
        max_tokens=1500,
        messages=old + [{
            "role": "user",
            "content": "Summarize the work so far: goals, decisions, "
                       "open tasks, and key facts. Be concise.",
        }],
    ).content[0].text

    # A summary note replaces the old turns; recent turns stay verbatim.
    return [{"role": "user", "content": f"Summary of earlier work:\n{summary}"},
            {"role": "assistant", "content": "Understood. Continuing."}] + recent
```

Two details matter: split at a point that does not separate a `tool_use` block from its `tool_result`, and make sure the history still starts with a `user` message and alternates sensibly.

### Interaction with prompt caching

Caching (lesson 003) works on an exact prefix. Editing or trimming the *early* part of the history invalidates the cache from that point onward, so your next request pays full input price once. Practical consequences:

- Compact **rarely and in bulk**, not a little every turn.
- Keep the stable parts (system prompt, tool definitions) first and unchanged.
- Append new content at the end where possible so the cached prefix stays valid.

### Putting long documents in the prompt

For large inputs, place the long documents near the top and your question or instructions at the end. Wrap documents in clear tags (for example `<document>` with a source name) so Claude can refer to them, and for extraction tasks ask it to quote the relevant passage before answering. This tends to improve accuracy on long inputs.

## Common exam traps

1. **"The model remembers earlier turns."** The API is stateless; history is only what you resend. Memory is your responsibility.
2. **Forgetting output tokens.** Input plus output must fit the window; a huge `max_tokens` reduces usable input space.
3. **Estimating tokens by character count** instead of using token counting or `usage`.
4. **Trimming that orphans a tool result.** A `tool_result` without its matching `tool_use` (or vice versa) causes an API error.
5. **Assuming a larger window removes the need for management.** Cost, latency, and focus still suffer.
6. **Compacting every turn** and thereby destroying cache hits.
7. **Treating a summary as lossless.** Exact values (IDs, error messages, numbers) should go to external memory or be preserved verbatim.

## Practice questions

**Q1.** A chat app resends the full history each turn. Costs rise steadily and, after many turns, requests start failing with a context-length error. Name two changes that address both problems.

**Q2.** You trim the oldest messages from a tool-using agent's history, and the next request returns a validation error. What is the most likely cause, and how do you fix it?

**Q3.** An agent summarizes and rewrites the start of its history after every single turn, and its cache hit rate drops to near zero. Why, and what should it do instead?

<details>
<summary>Answers</summary>

**A1.** Any two of: compact older turns into a summary; clear bulky, no-longer-needed tool results; move durable facts to external memory and retrieve them on demand; delegate noisy subtasks to subagents. These shrink the input per turn, which lowers cost and keeps requests under the limit. Also monitor usage with token counting.

**A2.** The trim separated a `tool_use` block from its matching `tool_result` (or left the history starting with an invalid message). Trim at turn boundaries so each tool_use/tool_result pair stays together, and ensure the history begins with a `user` message.

**A3.** Prompt caching matches an exact prefix, so changing early content invalidates the cache from that point on every turn. Compact in bulk only when a threshold is reached, keep the system prompt and tools stable at the front, and append new content at the end.

</details>

## Key takeaways

- The window holds system prompt, tools, history, tool results, and the output; all of it is budgeted.
- Use `count_tokens` and `response.usage` to measure rather than estimate.
- Manage context actively: trim, compact, clear old tool results, use external memory, retrieval, or subagents.
- Keep tool_use/tool_result pairs together when editing history.
- Compact rarely and in bulk to protect prompt caching.
- Put long documents first, the question last, and use tags to organize them.
