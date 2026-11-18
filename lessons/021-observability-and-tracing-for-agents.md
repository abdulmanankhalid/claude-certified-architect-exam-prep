# Observability for Claude Applications: Tracing, Logging, and Debugging Agents

## What you will learn

- Why agents need *traces* (not just request logs) to be debugged
- What to record for every model call and every tool call
- How to correlate steps with trace IDs and turn production traces into eval cases
- How to handle privacy, sampling, and cost of logging
- Which metrics and alerts matter in production

## Explanation

### Why agents are hard to debug

A single user request to an agent may trigger 5 to 30 model calls, several tool calls, and retries (see lesson 004 and lesson 016). When the final answer is wrong, the bug is usually in an *earlier step*: a bad tool argument, a truncated tool result, a stale context. A log line saying "request failed" or "answer returned" does not tell you where.

An **observability** setup lets you answer: *what did the model see, what did it decide, what did each tool return, how long and how much did it cost?*

### The trace model

- **Trace**: one end-to-end user task (one `trace_id`).
- **Span**: one step inside it: a model call, a tool call, a retrieval, a guardrail check.
- Spans nest: an orchestrator span contains subagent spans (lesson 009).

```
trace 7f3a  "Refund order 1182"                 4.2s  $0.031
├─ span model_call #1   stop_reason=tool_use     1.1s  in=2,400 out=120
├─ span tool lookup_order {id:1182}  ok          0.2s
├─ span model_call #2   stop_reason=tool_use     0.9s  in=2,900 out=95
├─ span tool issue_refund {id:1182} ERROR 409    0.3s
└─ span model_call #3   stop_reason=end_turn     1.7s  in=3,100 out=210
```

Reading this, you instantly see the refund tool returned a 409 and the model's last answer should be checked against that.

### What to record per model call

```python
resp = client.messages.create(...)
log.info({
    "trace_id": trace_id,
    "span": "model_call",
    "model": resp.model,
    "request_id": resp._request_id,      # from the request-id header
    "stop_reason": resp.stop_reason,
    "input_tokens": resp.usage.input_tokens,
    "output_tokens": resp.usage.output_tokens,
    "cache_read": resp.usage.cache_read_input_tokens,
    "cache_write": resp.usage.cache_creation_input_tokens,
    "latency_ms": elapsed_ms,
    "prompt_version": "support-v14",
})
```

- **Prompt version**: lets you tie a regression to a prompt change.
- **Cache fields**: confirm prompt caching (lesson 003) is actually hitting.
- **stop_reason**: `max_tokens` means truncation; `tool_use` means the loop continues; `refusal` and `end_turn` are terminal.

### What to record per tool call

- Tool name, arguments, result (or a truncated/hashed version), error flag, latency
- The `tool_use_id`, which links the call to its `tool_result`
- Whether a human approved it (lesson 020)

### Full transcripts for replay

Keep the complete `messages` array (system prompt, user turns, assistant turns, tool results) for sampled or failed traces. With the transcript you can **replay** the exact input against a new prompt or model. Replay turns production failures into regression cases for your eval suite (lesson 007).

### Production metrics and alerts

| Metric | Why it matters |
|---|---|
| Task success rate (graded sample) | The true quality signal |
| Loop iterations per task (p50, p95) | Rising = agent flailing or tools unclear (lesson 019) |
| Tool error rate per tool | Finds a broken or confusing tool |
| `max_tokens` stop rate | Truncated answers |
| Cache hit rate | Cost and latency (lesson 011) |
| 429/529 rate and retries | Capacity problems (lesson 016) |
| Cost per task | Budget control |

Set a hard **iteration cap and token/cost budget** per trace and alert when it is hit. Runaway loops are visible in traces long before they show up on the bill.

### Privacy, sampling, and cost of logging

- Prompts and tool results often contain **PII or secrets**. Redact or hash before storing, set retention limits, and restrict access.
- Logging everything at high volume is expensive. Always keep metadata (tokens, latency, stop reason); keep full transcripts for **errors, low-graded runs, and a random sample**.
- Never log API keys. Anthropic keys must stay out of traces and prompts.

### In Claude Code and the Agent SDK

Hooks (lesson 006) such as `PreToolUse` and `PostToolUse` are a natural place to emit tool spans, audit logs, and approvals. Headless mode (lesson 013) can emit structured JSON output for CI logs. Subagents have separate contexts, so trace them as child spans or you lose visibility into what they did.

## Common exam traps

1. **"Log only the final answer."** Insufficient: you cannot locate which step failed. Trace each step.
2. **Treating `request-id` as optional.** It is what support needs to investigate a specific call.
3. **Logging raw prompts with PII forever.** Redaction and retention are part of the correct design.
4. **Assuming the model's own explanation is a reliable log.** The model's stated reasoning is not a substitute for recorded tool inputs and outputs.
5. **Monitoring only latency and error codes.** An agent can return HTTP 200 with a wrong answer; you also need quality grading on sampled traces.
6. **Losing subagent activity.** If subagent calls are not linked to the parent trace, multi-agent failures are undebuggable.

## Practice questions

1. An agent sometimes gives wrong refund amounts, but the API never returns errors. What observability change helps most?
2. Your cache hit rate suddenly drops to near zero after a deploy. Which logged fields would you inspect first, and what is a likely cause?
3. Your team wants to store all prompts and tool results indefinitely for debugging. What concerns should the architect raise, and what is a better design?

<details>
<summary>Answers</summary>

1. Add step-level tracing: log each tool call's arguments and results with a shared `trace_id`, and grade a sample of completed traces for correctness. HTTP success says nothing about answer quality, and the bug is probably in a tool argument or tool result earlier in the loop.
2. Inspect `cache_read_input_tokens` and `cache_creation_input_tokens` plus the `prompt_version`. A likely cause is that the deploy changed content before the cache breakpoint (for example a timestamp or reordered tools in the prefix), so the prefix no longer matches (lesson 003).
3. Prompts and results can contain PII and secrets, so indefinite retention is a privacy and security risk and is costly. Better: redact or hash sensitive fields, set retention limits and access controls, always keep lightweight metadata, and keep full transcripts only for errors, low-quality runs, and a random sample.

</details>

## Key takeaways

- Debug agents with **traces**: one trace per task, one span per model call or tool call.
- Record model, request id, stop reason, token and cache usage, latency, and prompt version on every call; record arguments, results, and errors on every tool call.
- Store transcripts for failures and samples, and replay them as eval cases.
- Monitor quality, loop length, tool errors, cache hits, and cost, not just HTTP status.
- Redact sensitive data, limit retention, and cap iterations and budget per trace.
