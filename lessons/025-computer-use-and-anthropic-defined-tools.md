# Computer Use and Anthropic-Defined Tools: Client Tools vs. Server Tools

## What you will learn

- The difference between user-defined tools and Anthropic-defined tools
- Who executes a tool: your code (client tools) or Anthropic's infrastructure (server tools)
- How computer use works: screenshots in, actions out, in a loop you run
- How to sandbox and constrain computer-use agents
- When to prefer an API or MCP tool over driving a GUI

## Explanation

### Three kinds of tools

| Kind | Who defines the schema | Who runs it | Examples |
|---|---|---|---|
| User-defined client tool | You | Your code | `get_order_status` |
| Anthropic-defined client tool | Anthropic (schema is built into the model's training) | Your code | computer use, bash, text editor |
| Server tool | Anthropic | Anthropic's servers | web search, code execution, web fetch |

The key question on the exam is always: **who executes the tool?**

- **Client tools** (including computer use): Claude returns a `tool_use` block with `stop_reason: "tool_use"`. Your code runs the action and sends back a `tool_result`. This is the agentic loop from lesson 004.
- **Server tools**: Claude calls the tool, the platform runs it, and the results come back inside the same response (as `server_tool_use` and result blocks). You do not send a `tool_result`, and there is no loop for you to write for that step.

Tool versions are dated strings (for example a `type` like `computer_20250124`) and often need a beta header. Always check the current docs for the exact identifier. Do not hard-code from memory.

### Server tools in short

```python
response = client.messages.create(
    model=MODEL,
    max_tokens=1024,
    tools=[{"type": "web_search_20250305", "name": "web_search", "max_uses": 3}],
    messages=[{"role": "user", "content": "What changed in the latest Python release?"}],
)
```

- `max_uses` caps cost per request.
- Results carry citations you should surface to users.
- Web content is untrusted input. Lesson 008 (prompt injection) applies fully.
- Domain allow/block lists can limit where the tool searches.
- Some server tools are not available on every cloud platform (see lesson 024). Check availability before designing around one.

### Computer use

Computer use lets Claude operate a desktop the way a person does: look at a screen, move the mouse, click, type.

The loop:

1. You send a task and the computer tool definition (display width and height).
2. Claude asks for an action, usually `screenshot` first.
3. **Your code** performs it on a real or virtual machine and returns the result, typically a screenshot as an image block (see lesson 017).
4. Repeat until Claude stops requesting tools.

```python
tools = [{
    "type": "computer_20250124",   # check docs for the current version
    "name": "computer",
    "display_width_px": 1280,
    "display_height_px": 800,
}]

while True:
    resp = client.messages.create(model=MODEL, max_tokens=2048,
                                  tools=tools, messages=messages)
    messages.append({"role": "assistant", "content": resp.content})
    if resp.stop_reason != "tool_use":
        break
    results = []
    for block in resp.content:
        if block.type == "tool_use":
            png_b64 = run_action(block.input)   # click, type, screenshot...
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": [{"type": "image", "source": {
                    "type": "base64", "media_type": "image/png", "data": png_b64}}],
            })
    messages.append({"role": "user", "content": results})
```

Practical points:

- **Coordinates matter.** Claude's clicks are in the coordinate space of the screenshots you send. Keep the display at a modest, fixed resolution. If you downscale screenshots, scale the coordinates back up yourself.
- **It is slow and token-heavy.** Every step costs a screenshot's worth of input tokens plus a model round trip. Use prompt caching (lesson 003) and trim old screenshots from history (lesson 010).
- **It is less reliable than structured tools.** Pop-ups, animations, and layout changes cause failures. Add step limits and verification steps ("take a screenshot and confirm the form was submitted").
- **Give explicit guidance** for brittle steps, such as waiting for page loads.

### Safety for computer use

A GUI agent can do anything a user can, and it reads whatever is on screen. That makes prompt injection (a web page or document with hidden instructions) the main risk.

- Run it in a **dedicated VM or container** with minimal privileges, not on a personal machine.
- Give it **no credentials** beyond what the task needs; avoid logged-in sessions to sensitive accounts.
- Use **network allowlists** so it can reach only needed domains.
- Require **human approval** for consequential actions such as payments, sending messages, or deleting data (lesson 020).
- Log every action and screenshot for audit (lesson 021).

### Bash and text editor tools

These are also Anthropic-defined **client** tools. Claude is trained on their schemas, so it uses them more reliably than a similar tool you invent. But you still implement execution, and you still own the sandbox and permissions. Claude Code and the Agent SDK (lesson 023) ship ready-made implementations.

### Choosing the right interface

Prefer, in order:

1. A direct API or **MCP tool** (fast, cheap, structured).
2. A **code or shell tool** in a sandbox.
3. **Computer use**, only when there is no API: legacy desktop apps, or sites you cannot integrate with.

## Common exam traps

- **"Computer use runs on Anthropic's servers."** No. It is a client tool. Your code executes every action in your environment.
- **"Server tools need a `tool_result` from you."** No. Anthropic runs them and returns results in the response.
- **Giving the agent your real desktop** or logged-in browser for convenience. The correct answer is an isolated VM or container.
- **Treating screen content as trusted.** Text on a web page is untrusted data, not instructions.
- **Using computer use when an API exists.** It is the last resort, not the default.
- **Forgetting coordinate scaling** when screenshots are resized.
- **Assuming every tool version or server tool is available on every platform.** Verify first.

## Practice questions

1. Claude returns `stop_reason: "tool_use"` with a `computer` tool call asking to click at (400, 300). What must your application do next?
2. A team wants an agent to look up current pricing on public web pages. They consider building a scraper tool versus enabling the web search server tool. What differences affect the architecture?
3. You must automate data entry into a legacy desktop app with no API, and the app also displays emails from customers. Describe the design and the main risk.

<details>
<summary>Answers</summary>

1. Perform the click in your sandboxed machine, capture a new screenshot, and send it back as a `tool_result` tied to the matching `tool_use_id`. Then call the API again and continue the loop until `stop_reason` is not `tool_use`.
2. With the server tool, Anthropic runs the search and returns results with citations in the same response, so you write no execution code and no extra loop step. You control cost with `max_uses` and scope with domain lists. A custom scraper gives you full control over sources and parsing but you build, host, and secure it. In both cases, retrieved content is untrusted and must not be followed as instructions.
3. Use computer use, since there is no API, inside an isolated VM with minimal credentials and network restrictions. Add step limits, verification screenshots, logging, and human approval for irreversible actions. The main risk is prompt injection: a customer email on screen could contain instructions, so the agent must treat on-screen text as data and have no access beyond what the task needs.

</details>

## Key takeaways

- Ask first: who executes the tool? Client tools run in your code; server tools run on Anthropic's side.
- Computer use is a client-tool loop: screenshot in, action out, your code executes.
- It is slower, costlier, and less reliable than structured tools, so use it only when no API exists.
- Isolate it: sandbox VM, least privilege, network allowlist, human approval for risky actions.
- Screen and web content is untrusted input; prompt injection is the central risk.
- Tool type strings are versioned and may need beta headers, so confirm them in the current docs.
