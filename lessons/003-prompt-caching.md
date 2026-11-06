# Prompt Caching: Cutting Cost and Latency for Repeated Context

## What you will learn

- What prompt caching is and why it reduces cost and latency
- How to mark cache breakpoints with `cache_control`
- Why the cached prefix order (tools, system, messages) matters
- How to read cache metrics in `usage`
- When caching helps and when it does not

## Explanation

Many requests repeat the same large prefix: a long system prompt, tool definitions, a reference document. Without caching, Claude reprocesses that prefix every call (remember from [Lesson 001](001-messages-api-basics.md) that the API is stateless). **Prompt caching** lets the API reuse the processed prefix, so repeated input tokens are cheaper and faster.

### Marking a breakpoint

```python
response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=500,
    system=[
        {"type": "text", "text": "You are a support agent for Acme."},
        {
            "type": "text",
            "text": LONG_POLICY_DOCUMENT,
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": "Can I return an opened item?"}],
)
```

`cache_control` on a block means: "cache everything in the prompt up to and including this block."

### Prefix order and exact matching

The cache covers the prompt prefix in this order: **tools, then system, then messages**. A cache hit needs the prefix to be **identical** up to the breakpoint. Changing anything earlier (a tool definition, one character of the system prompt) invalidates the cache from that point on.

Design rule: put **stable** content first, **volatile** content last. Timestamps, user names, or per-request data placed at the top of the system prompt will break caching every time.

### Reading the results

`usage` reports what happened:

- `cache_creation_input_tokens`: tokens written to the cache (priced higher than normal input).
- `cache_read_input_tokens`: tokens read from the cache (priced much lower than normal input).
- `input_tokens`: the remaining uncached tokens after the last breakpoint.

The first call writes the cache; later calls within the lifetime read it.

### Lifetime and limits

- The default cache lifetime is short (about 5 minutes) and is refreshed each time the entry is used. A longer lifetime option exists at a higher write price. Check current docs for exact numbers and prices.
- There is a **minimum cacheable prompt length** that varies by model; shorter prefixes are silently not cached (no error, just no cache fields).
- You can use several breakpoints (a small maximum number per request), e.g. one after tools, one after the system prompt, one at the end of the conversation so multi-turn chats reuse earlier turns.

### When to use it

Good fit: long system prompts, big documents queried repeatedly, agents with many tool definitions, long multi-turn conversations.
Poor fit: one-off requests, prefixes below the minimum, or traffic so sparse that the cache expires between calls (you pay the write premium and never get reads).

Caching changes **cost and latency only**, not the model's output behavior.

## Common exam traps

- **Thinking caching stores responses.** It caches the processed *input prefix*, not answers. Output tokens are always generated fresh.
- **Putting dynamic data first.** A timestamp at the top of the system prompt defeats the cache.
- **Expecting a hit after a small edit.** Any change before the breakpoint invalidates it.
- **Assuming a cache write is free.** Writes cost more than normal input; savings come from reads.
- **Expecting an error for too-short prompts.** They just are not cached.
- **Confusing it with conversation memory.** Caching does not make the API stateful; you still resend the full history.

## Practice questions

1. An app sends a 6,000-token policy document in the system prompt on every request, with the user's name and current time on the first line. Cache reads are always 0. Why, and what is the fix?
2. In `usage`, you see `cache_creation_input_tokens: 0` and `cache_read_input_tokens: 0` although `cache_control` is set. Name a likely cause.
3. A batch job calls Claude once an hour with a large shared prefix. Is default prompt caching a good fit?

<details>
<summary>Answers</summary>

1. The per-request name and time at the start change the prefix, so it never matches. Move volatile data after the stable document (e.g. into the user message) and put the breakpoint after the policy document.
2. The prefix is below the model's minimum cacheable length, so it was silently not cached. (If the prefix changed every call you would still see creation tokens, not zeros.)
3. Not with the default short lifetime: the entry expires between calls, so you pay write costs without reads. Consider the longer-lifetime option, or accept uncached cost.

</details>

## Key takeaways

- Prompt caching reuses a repeated input prefix to lower cost and latency; outputs are unaffected.
- Mark breakpoints with `cache_control`; the cached prefix order is tools, system, messages.
- Stable content first, volatile content last; any earlier change invalidates the cache.
- Check `cache_creation_input_tokens` and `cache_read_input_tokens` to verify it works.
- Writes cost more than normal input, reads cost much less; it pays off only with reuse inside the cache lifetime.
