# The Agentic Loop: Running Claude Until the Job Is Done

## What you will learn

- What an agent is: a model calling tools in a loop, driven by `stop_reason`
- How to write a safe manual agent loop (and when to use an SDK instead)
- How to bound a loop: max turns, token budgets, and timeouts
- How to handle tool errors and keep context from growing without limit
- When a single call or fixed workflow is better than an agent

## Explanation

### An agent is a loop

In lesson 002 you saw one tool round trip. An **agent** repeats it: Claude
decides, calls a tool, sees the result, decides again, until it has an answer.
The loop is controlled by `stop_reason`:

- `tool_use`: Claude wants tools run. Run them, append results, call again.
- `end_turn`: Claude is finished. Return the answer.
- `max_tokens`: the output was cut off. Not a normal finish.
- `pause_turn`: a server-side tool turn paused. Send the conversation back to continue.

### A minimal manual loop

```python
messages = [{"role": "user", "content": task}]
for turn in range(MAX_TURNS):                      # hard bound
    resp = client.messages.create(
        model=MODEL, max_tokens=2048, tools=tools, messages=messages
    )
    messages.append({"role": "assistant", "content": resp.content})

    if resp.stop_reason != "tool_use":
        break                                      # end_turn, max_tokens, ...

    results = []
    for block in resp.content:
        if block.type == "tool_use":
            try:
                out, is_err = run_tool(block.name, block.input), False
            except Exception as e:
                out, is_err = str(e), True         # let Claude see the failure
            results.append({"type": "tool_result",
                            "tool_use_id": block.id,
                            "content": out,
                            "is_error": is_err})
    messages.append({"role": "user", "content": results})
```

Key points:

- Append the **whole** assistant content (including `tool_use` blocks) before the results.
- Every `tool_use` block needs a matching `tool_result` with the same id, all in one user message.
- Return errors with `is_error: true` so Claude can retry or change approach rather than crash the loop.

### Bounding the loop

Never trust the model alone to stop. Combine:

- **Max turns** (as above) to stop runaway loops.
- **Token or cost budget** summed from `resp.usage`.
- **Wall-clock timeout** for tools and the whole run.
- **Loop detection**: same tool and same input repeated means stop or intervene.

### Managing context

Each turn resends the full history, so cost and latency grow. Mitigations:

- Prompt caching on the stable prefix (tools, system prompt), see lesson 003.
- Truncate or summarize large tool outputs before returning them.
- Clear or compact old tool results when the history gets long.
- Delegate noisy subtasks to a subagent that returns only a summary.

### Loop yourself or use an SDK?

The Agent SDK and the SDK tool runner implement this loop for you, including
tool execution, permissions, and context handling. Write the loop by hand when
you need custom control (approvals, logging, special stop logic). Otherwise
prefer the SDK.

### Agent vs workflow

If the steps are known in advance (extract, then classify, then store), use a
fixed **workflow** of separate calls. It is cheaper, faster, and more
predictable. Use an agent when the path depends on what is discovered along
the way. Start with the simplest design that works.

## Common exam traps

- **Stopping on text instead of `stop_reason`.** Claude may emit text and also a `tool_use` block. Check `stop_reason`.
- **Dropping the assistant turn** or sending results without the matching `tool_use` ids: the API returns an error.
- **No iteration cap.** A loop without a bound can run up cost indefinitely.
- **Raising on tool failure** instead of returning `is_error` results.
- **Treating `max_tokens` as success.** The answer may be truncated; raise the limit or continue.
- **Choosing an agent for a fixed pipeline.** A workflow is usually the better answer.

## Practice questions

1. Your loop ends whenever the response contains a text block, and users report half-finished tasks. What is the bug?
2. A tool throws a timeout. Should the loop abort? What should you return?
3. A pipeline always runs: parse invoice, validate totals, write to database. Agent or workflow?

<details>
<summary>Answers</summary>

1. Claude can include text alongside `tool_use` blocks (for example "Let me check that"). The loop must continue while `stop_reason == "tool_use"` and stop only on other reasons such as `end_turn`.
2. No. Return a `tool_result` with `is_error: true` and a short message. Claude can retry, try another tool, or explain the limitation. Keep the max-turn cap so retries are bounded.
3. A fixed workflow. The steps and order are known, so separate deterministic calls are cheaper, faster, and easier to test. An agent adds unneeded variability.

</details>

## Key takeaways

- An agent is a loop: call, run requested tools, append results, repeat until `stop_reason` is not `tool_use`.
- Always bound the loop with max turns, budgets, and timeouts.
- Return tool errors with `is_error: true`; do not crash.
- Control context growth with caching, truncation, and subagents.
- Prefer a fixed workflow when the steps are known; use an agent when they are not.
