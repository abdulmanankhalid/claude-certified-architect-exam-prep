# Tool Use: Letting Claude Call Your Functions

## What you will learn

- How to define a tool with a name, description, and JSON `input_schema`
- The tool-use loop: `stop_reason: "tool_use"`, `tool_use` blocks, and `tool_result` blocks
- Who actually runs the tool (your code, not Claude)
- How to control tool choice with `tool_choice`
- How to report tool errors back to Claude

## Explanation

Claude cannot run your code or reach your systems by itself. With **tool use** (function calling), you describe tools; Claude decides when to call one and returns the arguments; **your application executes it** and sends back the result. This builds on the stateless request/response cycle from [Lesson 001](001-messages-api-basics.md).

### Defining a tool

```python
tools = [{
    "name": "get_order_status",
    "description": "Look up the shipping status of an order by its ID. "
                   "Use when the user asks where their order is.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
}]
```

The **description** is the most important field: Claude uses it to decide whether and how to call the tool. Say what it does, when to use it, and what the parameters mean.

### The loop

```python
messages = [{"role": "user", "content": "Where is order A-1001?"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-5-5", max_tokens=1024,
        tools=tools, messages=messages,
    )
    if response.stop_reason != "tool_use":
        break  # "end_turn": final answer is ready

    messages.append({"role": "assistant", "content": response.content})

    results = []
    for block in response.content:
        if block.type == "tool_use":
            output = run_tool(block.name, block.input)  # YOUR code
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,   # must match the call
                "content": output,
            })
    messages.append({"role": "user", "content": results})
```

Key points:

- When Claude wants a tool, `stop_reason` is `"tool_use"` and `content` holds one or more `tool_use` blocks (with `id`, `name`, `input`). There may also be text before them.
- You append Claude's whole assistant turn unchanged, then reply with a **user** message containing `tool_result` blocks.
- Each `tool_result` references its call through `tool_use_id`. Claude may request several tools in one turn (parallel calls); return all results in a single user message.
- Pass `tools` on every request, because the API is stateless.

### Controlling tool choice

| `tool_choice` | Behavior |
|---|---|
| `{"type": "auto"}` (default) | Claude decides whether to call a tool |
| `{"type": "any"}` | Must call some tool |
| `{"type": "tool", "name": "x"}` | Must call that specific tool |
| `{"type": "none"}` | No tool calls |

Forcing a tool is a common way to get **structured output**: define a tool whose schema is the shape you want and read `block.input`.

### Handling errors

If your tool fails, still return a `tool_result`, with `"is_error": true` and a helpful message. Claude can then retry with different arguments or explain the problem to the user.

```python
{"type": "tool_result", "tool_use_id": block.id,
 "content": "Order not found. IDs look like A-1234.", "is_error": True}
```

### Server tools vs client tools

Client tools (above) run in your code. Server tools (such as web search) are run by Anthropic's platform; results come back inside the same response with no loop on your side.

## Common exam traps

- **"Claude executes the function."** No. Claude only emits a request; your code runs it.
- Forgetting to send the `tool_use` assistant turn back before the `tool_result`, or mismatching `tool_use_id`.
- Putting `tool_result` blocks in an `assistant` message. They belong in a `user` message.
- Treating `stop_reason == "tool_use"` as the end. It means you must run tools and call again.
- Vague tool descriptions. Poor descriptions are the usual cause of wrong or missed tool calls.
- Leaving out the loop's exit condition or a max-iteration limit, risking infinite loops.
- Trusting `input` blindly: validate arguments before executing, especially for side effects.

## Practice questions

1. After a response with `stop_reason: "tool_use"`, what must your next request contain?
2. You need JSON matching a schema from a text-extraction call. How can tool use help?
3. A tool throws an exception. What should you send back?

<details>
<summary>Answers</summary>

1. The prior messages, Claude's assistant turn (including the `tool_use` blocks) appended unchanged, and a new `user` message with `tool_result` blocks whose `tool_use_id` values match each call. Also resend `tools`.
2. Define a tool whose `input_schema` is the desired structure and set `tool_choice` to that tool; read the structured data from the `tool_use` block's `input`.
3. A `tool_result` with `is_error: true` and a clear message, so Claude can recover or explain, rather than dropping the call or crashing the loop.

</details>

## Key takeaways

- Claude requests tools; your code executes them and returns results.
- The loop continues while `stop_reason == "tool_use"` and ends on `end_turn`.
- Clear descriptions and schemas drive tool selection quality.
- `tool_choice` controls whether and which tool is called; forcing a tool yields structured output.
- Return errors with `is_error`, validate inputs, and cap loop iterations.
