# Tool Design for Agents: Descriptions, Granularity, Responses, and Tool Overload

## What you will learn

- Why tool descriptions are the most important "prompt" in a tool-using system
- How to choose tool granularity: fewer, well-scoped tools beat many overlapping ones
- How to shape tool responses so they help Claude (and save tokens)
- What goes wrong when an agent has too many tools, and how to fix it
- How to write error messages and parameters that make Claude recover on its own

## Explanation

Lesson 002 showed *how* to call tools. This lesson is about *designing* them. Claude never sees your code. It sees only the tool's **name, description, and input schema**. If those are vague, Claude will pick the wrong tool or pass bad arguments, however good your function is.

### 1. Write descriptions like onboarding notes

A good description says what the tool does, when to use it, when *not* to use it, and what it returns.

```json
{
  "name": "search_orders",
  "description": "Search a customer's past orders by date range or status. Use when the user asks about an existing order. Do NOT use to create or change orders. Returns up to 20 orders, newest first, each with id, status, and total.",
  "input_schema": {
    "type": "object",
    "properties": {
      "customer_id": {"type": "string", "description": "Internal ID, e.g. 'C-1042'. Not the email."},
      "status": {"type": "string", "enum": ["pending", "shipped", "delivered", "cancelled"]},
      "since": {"type": "string", "description": "ISO date, e.g. '2026-01-31'"}
    },
    "required": ["customer_id"]
  }
}
```

Tips:
- Use `enum` for closed sets, and put examples and formats in parameter descriptions.
- Name tools with a clear verb and noun (`search_orders`, not `orders2`). Use consistent prefixes for related tools (`jira_create_issue`, `jira_search`).
- Prefer human-meaningful parameters (a name or a slug) over opaque IDs when you can. If IDs are needed, tell Claude which tool returns them.
- Spell out dependencies: "Call `get_customer` first to obtain `customer_id`."

### 2. Choose the right granularity

| Design | Problem |
|---|---|
| Thin wrapper per API endpoint (`list_users`, `list_events`, `list_rooms`, then Claude joins them) | Many round trips, and Claude must do the joining |
| One giant tool with a `mode` flag and 30 optional parameters | Confusing schema, frequent invalid combinations |
| Task-oriented tools (`schedule_meeting(attendees, duration)`) | Usually best: fewer calls, clear purpose |

Rule of thumb: design tools around **what the agent is trying to accomplish**, not around your internal API shape. Combine steps that are always done together. Keep separate the actions that have different risk, such as `read_file` and `delete_file`, so permissions and approvals can be applied per tool (see lessons 006 and 008).

### 3. Shape the response

The tool result goes straight into the context window (lesson 010), so it costs tokens on every later turn.

- Return only the fields Claude needs. Drop internal IDs, audit metadata, and raw HTML.
- Use readable names (`"status": "shipped"`) rather than codes (`"s": 3`).
- Paginate or truncate large results, and say so: `"showing 20 of 340; refine with filters"`.
- Offer a `detail` parameter (`"concise"` or `"full"`) so Claude can ask for more only when needed.
- Keep output formats stable. Claude learns the shape within a conversation.

### 4. Make errors actionable

Return `is_error: true` with a message that says what to do next (lesson 002):

```text
Bad:  "Error 422"
Good: "No customer matches 'jon smith'. Try search_customers with a partial name or an email."
```

An error that teaches the next step lets the agentic loop (lesson 004) self-correct without a human.

### 5. Beware tool overload

Every tool definition is sent on each request and counts as input tokens. With dozens of tools, such as several MCP servers (lesson 005) connected at once, you get:

- more tokens and higher cost, though prompt caching (lesson 003) softens it. Tool definitions are part of the cacheable prefix, and changing them invalidates the cache.
- worse tool selection, because similar tools compete.

Fixes:
1. Only expose the tools needed for the current task or agent.
2. Split work across **subagents**, each with a small toolset (lesson 009).
3. Use a tool-search or on-demand loading mechanism, where the model finds and loads tool definitions only when needed, if your platform provides one.
4. Merge or delete overlapping tools.

### 6. Do not rely on the description for safety

A description saying "never call this without user approval" is a hint, not a control. Enforce rules in code: validate inputs, check permissions, require confirmation for destructive actions, and log calls (lesson 008).

## Common exam traps

- **"Improve accuracy by fixing the function code."** If Claude picks the wrong tool or arguments, look at the name, description, and schema first.
- **One tool per API endpoint is always best.** Not so. Task-oriented tools usually reduce calls and errors.
- **More tools means a more capable agent.** Past a point, selection quality and cost get worse. Scope tools per agent.
- **Returning the entire raw API response is safest.** It wastes context and buries the useful data.
- **Putting security rules only in the tool description.** Descriptions guide the model but do not enforce anything. Enforce in code.
- **Editing tool definitions mid-conversation is free.** It invalidates the prompt cache for everything after the change.
- **Throwing an exception when a tool fails.** Return a `tool_result` with `is_error: true` instead.

## Practice questions

<details>
<summary>Show questions and answers</summary>

**Q1.** An agent has `get_data` and `fetch_info` tools. It often calls the wrong one. What is the best first fix?

**A1.** Rename them to clear verb-noun names and rewrite the descriptions to say what each returns, when to use it, and when not to. Merge them if they overlap. Changing the model or the backend code does not address the ambiguity.

**Q2.** A `search_logs` tool returns 5,000 lines per call, and the agent's later answers degrade. What should you change?

**A2.** Limit and paginate results, return only relevant fields, state the truncation in the output (for example "showing 50 of 4,812; add a time filter"), and consider a `concise`/`full` option. Large results fill the context window and hurt later turns.

**Q3.** Your agent connects to five MCP servers with 80 tools in total. Cost is up and tool selection is poor. Name two sensible mitigations.

**A3.** Expose only task-relevant tools per agent or step, delegate to subagents with small toolsets, use on-demand tool search if available, and remove overlapping tools. Keep the tool list stable between requests so the prompt cache stays valid.

</details>

## Key takeaways

- Claude sees only the name, description, and schema, so design these carefully.
- Build task-oriented tools of sensible granularity, and separate risky actions from safe ones.
- Return concise, readable, bounded results, and write errors that tell Claude what to try next.
- Too many tools raises cost and lowers accuracy. Scope tools per agent and use subagents.
- Enforce safety in code, not in descriptions.
