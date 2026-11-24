# Parallel Tool Calls and Streaming Tool Use: Faster Agents Without Breaking the Protocol

## What you will learn

- How Claude can request several tool calls in one turn, and how to return all the results correctly
- When parallel execution is safe and when it is not
- How `disable_parallel_tool_use` and `tool_choice` interact
- How streaming delivers tool calls (`input_json_delta`) and why you must not parse partial JSON
- The common mistakes that cause 400 errors or slow agents

## Explanation

### Parallel tool calls

When a task needs several independent lookups ("weather in Paris, Rome, and Berlin"), Claude may return **one assistant message containing multiple `tool_use` blocks**, each with its own `id`.

```json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {"type": "tool_use", "id": "toolu_A", "name": "get_weather", "input": {"city": "Paris"}},
    {"type": "tool_use", "id": "toolu_B", "name": "get_weather", "input": {"city": "Rome"}}
  ]
}
```

Your reply must be **one user message** that holds a `tool_result` block for every `tool_use` id:

```json
{
  "role": "user",
  "content": [
    {"type": "tool_result", "tool_use_id": "toolu_A", "content": "18C, cloudy"},
    {"type": "tool_result", "tool_use_id": "toolu_B", "content": "24C, sunny"}
  ]
}
```

Rules:

- Every `tool_use` id needs a matching `tool_result`. A missing one gives a 400 error.
- Put all results in a single user message. Sending them as separate consecutive user messages teaches the model to stop making parallel calls and may be rejected.
- In that message, `tool_result` blocks come **first**, before any extra text.
- Results may be in any order, because ids do the matching.

### Running them in parallel yourself

Claude only *requests* the calls. Your code decides how to run them. Independent, read-only calls can run concurrently:

```python
import asyncio

async def run_all(tool_uses):
    async def run(tu):
        try:
            out = await TOOLS[tu.name](**tu.input)
            return {"type": "tool_result", "tool_use_id": tu.id, "content": out}
        except Exception as e:
            return {"type": "tool_result", "tool_use_id": tu.id,
                    "content": f"Error: {e}", "is_error": True}
    return await asyncio.gather(*(run(tu) for tu in tool_uses))
```

Catch errors per call so one failure does not lose the other results.

### When NOT to parallelize

- **Dependent calls**: step 2 needs the output of step 1. Claude will normally sequence these itself, since it cannot fill in an unknown value.
- **Write operations with ordering or conflicts**: two edits to the same file, or "create then update".
- **Rate-limited or side-effecting tools**: you may want to serialize execution in your own code.

You can also turn the behavior off: set `disable_parallel_tool_use: true` inside `tool_choice` (for `auto`, at most one tool; for `any` or `tool`, exactly one).

```python
tool_choice={"type": "auto", "disable_parallel_tool_use": True}
```

### Steering toward parallelism

If Claude calls tools one at a time when it could batch them, add a prompt hint such as "When calls are independent, make them all in the same turn." Also make sure tool results come back in one message, since sending them separately discourages batching.

### Streaming and tool use

With `stream: true`, the response arrives as server-sent events:

1. `message_start`
2. For each content block: `content_block_start`, many `content_block_delta`, then `content_block_stop`
3. `message_delta` (carries `stop_reason` and final usage), then `message_stop`

Text blocks stream `text_delta`. Tool-use blocks stream `input_json_delta` whose `partial_json` is a **fragment of a JSON string**:

```
content_block_start  -> {"type":"tool_use","id":"toolu_A","name":"get_weather","input":{}}
content_block_delta  -> {"partial_json":"{\"ci"}
content_block_delta  -> {"partial_json":"ty\": \"Par"}
content_block_delta  -> {"partial_json":"is\"}"}
content_block_stop
```

Correct handling: accumulate the fragments per content block `index`, and call `json.loads` only after `content_block_stop`. Parallel calls appear as separate blocks with different indexes. Tools are executed only once the stream says `stop_reason: "tool_use"` (or you can start a tool as soon as its own block stops, to save latency).

Streaming does not change the protocol. After the stream ends you append the full assistant message and your `tool_result` message just like the non-streaming loop. SDK helpers (such as the stream's `get_final_message()`) assemble the full message for you.

Also remember: with extended thinking, thinking blocks must be passed back unchanged along with tool results (see lesson 014).

### Latency view

Parallel calls cut **round trips**: three sequential lookups are 3 model calls plus 3 tool waits, while one parallel turn is 1 model call plus the slowest tool. Streaming cuts **perceived latency** (time to first token) but does not reduce total work.

## Common exam traps

- **Splitting results across messages.** All `tool_result` blocks for one assistant turn go in one user message.
- **Forgetting a result.** Every `tool_use` id needs an answer, even if the answer is an error with `is_error: true`.
- **Parsing partial JSON** mid-stream. Wait for `content_block_stop`.
- **Assuming Claude runs the tools in parallel.** Claude requests; your code executes.
- **Thinking `disable_parallel_tool_use` forbids tools.** It only limits how many are requested per turn.
- **Parallelizing dependent or conflicting writes.**
- **Believing streaming reduces cost or total time.** It improves time to first token only.

## Practice questions

1. Claude returns an assistant message with three `tool_use` blocks. How should you reply?
2. While streaming, you receive `input_json_delta` events for a tool call. When is it safe to parse the input?
3. A task requires "read config, then query the database named in the config". Should you expect parallel calls, and what setting could force one call at a time anyway?

<details>
<summary>Answers</summary>

1. Send one user message with three `tool_result` blocks, each carrying the matching `tool_use_id` (errors use `is_error: true`), placed before any other text.
2. After the `content_block_stop` for that block. Accumulate `partial_json` fragments by block index until then.
3. No. The second call depends on the first call's output, so Claude will call sequentially. To force at most one call per turn regardless, set `disable_parallel_tool_use: true` in `tool_choice`.

</details>

## Key takeaways

- One assistant turn may hold many `tool_use` blocks; answer all in one user message with matching ids.
- Claude requests calls; your code chooses sequential or concurrent execution, and handles per-call errors.
- Parallelize only independent calls; use `disable_parallel_tool_use` to limit.
- Stream tool input as JSON fragments; parse only at block end.
- Parallelism reduces round trips; streaming reduces perceived latency.
