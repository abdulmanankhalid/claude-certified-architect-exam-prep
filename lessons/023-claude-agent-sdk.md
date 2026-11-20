# The Claude Agent SDK: Building Production Agents with Built-in Tools, Permissions, and Sessions

## What you will learn

- What the Agent SDK gives you compared with writing your own loop on the Messages API
- How to configure an agent: tools, system prompt, permission mode, turn limits, and MCP servers
- How hooks, custom tools, and programmatic subagents extend an SDK agent
- How sessions let you resume or fork a conversation
- How to choose between the raw Messages API, the SDK tool runner, and the Agent SDK

## Explanation

### What the Agent SDK is

The Agent SDK (Python and TypeScript) exposes the same agent harness that powers Claude Code as a library. In lesson 004 you wrote the agentic loop yourself: call the model, run tools, append results, repeat. The Agent SDK runs that loop for you and adds the pieces production agents need:

| Capability | You build it yourself on the Messages API | Agent SDK |
|---|---|---|
| Agentic loop | Yes | Built in |
| File, shell, search, and web tools | Yes | Built-in tools (Read, Edit, Bash, Grep, Glob, WebFetch, ...) |
| Context compaction | Yes (lesson 010) | Handled automatically |
| Permissions and approval | Yes (lesson 020) | Permission modes, allow/deny rules, callbacks |
| Hooks, skills, CLAUDE.md, subagents | No | Supported (lessons 006, 009, 013) |
| MCP servers | Client code | Configured by options |
| Sessions | Persist messages yourself | Session IDs, resume, fork |

Key idea: the Agent SDK is for agents that *act* (edit files, run commands, browse). If you only need one completion or a fixed workflow (lesson 018), the Messages API is simpler and cheaper.

### A minimal agent

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(
        system_prompt="You are a careful code reviewer.",
        allowed_tools=["Read", "Grep", "Glob"],   # read-only
        permission_mode="default",
        max_turns=10,                             # cap the loop
    )
    async for message in query(prompt="Review src/auth.py for bugs", options=options):
        print(message)       # assistant text, tool calls, and a final result message

asyncio.run(main())
```

`query()` streams messages as the agent works. The last message is a result containing the final text, cost, turn count, and a session ID. Exact class and field names can change between versions, so check the current docs, but the concepts below are stable.

### Configuring tools and permissions

Two separate questions control tool access:

1. **Which tools exist and may run without asking?** `allowed_tools` pre-approves tools. `disallowed_tools` blocks them outright.
2. **What happens to everything else?** The **permission mode** decides.

| Permission mode | Behavior |
|---|---|
| `default` | Unlisted tools need approval (via a callback or prompt) |
| `acceptEdits` | File edits are auto-approved; other risky actions still ask |
| `plan` | Claude may only plan and read; no changes are executed |
| `bypassPermissions` | Everything runs without asking. Only for isolated sandboxes |

Least privilege applies: a review agent gets Read, Grep, and Glob, not Bash. Deny rules take priority over allow rules.

For custom decisions, supply a permission callback (often called `can_use_tool`) that inspects the tool name and input and returns allow or deny. This is the human-in-the-loop gate from lesson 020, implemented in code.

### Custom tools and MCP

You can add your own tools two ways:

- **In-process tools**: define a Python or TypeScript function, wrap it as a tool, and bundle it into an SDK MCP server passed through `mcp_servers`.
- **External MCP servers**: point `mcp_servers` at a stdio or HTTP server (lesson 005).

MCP tools are named `mcp__<server>__<tool>`, and you allow them with that full name, for example `allowed_tools=["mcp__crm__lookup_customer"]`. Forgetting this prefix is a common reason a tool is "available but never used" or keeps prompting for approval.

### Hooks

Hooks are your code that runs at fixed points in the loop, such as `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, and `Stop`. Unlike instructions in a prompt, hooks are **deterministic**: Claude cannot forget or talk its way past them.

```python
async def block_secrets(input_data, tool_use_id, context):
    path = input_data["tool_input"].get("file_path", "")
    if path.endswith(".env"):
        return {"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "Do not read .env files"}}
    return {}
```

Use hooks for audit logging, blocking dangerous inputs, injecting context, and running a linter after every edit. Use prompts for guidance and judgment; use hooks for rules that must always hold.

### Subagents

Pass `agents={...}` to define subagents programmatically: each has a description (used by the main agent to decide when to delegate), a prompt, and its own restricted tool list. Subagents run in a fresh context and return only a summary, which keeps the main context small (lesson 009). The main agent needs the `Task` tool allowed to delegate.

### Sessions

Every run has a session ID. Pass it back with `resume=<session_id>` to continue with full prior context, for example a user returning to a conversation the next day. Forking creates a new session branching from an old one, so you can try an alternative without corrupting the original. Sessions complement, but do not replace, the memory patterns of lesson 022: a session preserves one conversation's transcript, while memory carries curated facts across sessions.

### Settings sources

By default, SDK agents may not load filesystem settings such as CLAUDE.md, `.claude/skills`, or `.claude/settings.json`. You opt in with a setting-sources option (for example project and user). If your agent ignores your CLAUDE.md in a server environment, this is the first thing to check.

### Choosing the right layer

| Need | Use |
|---|---|
| One-shot call, classification, extraction | Messages API |
| Your own tools with a simple loop | SDK tool runner on the Messages API |
| Agent that reads/edits files, runs commands, needs permissions and compaction | Agent SDK |
| Interactive developer use or CI one-offs | Claude Code CLI (headless mode, lesson 013) |

### Production checklist

- Run the agent in a container or sandbox with only the files and network it needs.
- Set `max_turns` and a budget so a confused agent cannot loop forever.
- Log every message and tool call (lesson 021).
- Keep API keys out of the agent's reachable filesystem.
- Evaluate with a fixed task set before and after changes (lesson 007).

## Common exam traps

- **Assuming `allowed_tools` restricts the agent to those tools.** It pre-approves them. Other tools can still be used if the permission mode and rules allow, so use `disallowed_tools` or a callback to truly block.
- **Using `bypassPermissions` on a developer machine or production host.** It is only acceptable inside an isolated sandbox.
- **Relying on a prompt to enforce a hard rule.** Rules like "never touch .env" belong in a hook or deny rule.
- **Reaching for the Agent SDK for a fixed pipeline.** A prompt chain on the Messages API is cheaper and more predictable (lesson 018).
- **Forgetting the `mcp__server__tool` naming** when allowing MCP tools.
- **Expecting CLAUDE.md or skills to load automatically** in SDK code without enabling setting sources.
- **Confusing sessions with memory.** Resuming replays one transcript; it is not curated cross-session knowledge.

## Practice questions

1. You are building a nightly agent that reads a repository and writes a summary report. It must never modify files or run shell commands. What configuration is most appropriate?

<details>
<summary>Answer</summary>

Allow only read tools (Read, Grep, Glob), explicitly disallow Edit, Write, and Bash, and set a restrictive permission mode (default or plan) with a `max_turns` cap. Run in a sandbox. Relying on `allowed_tools` alone is not enough because it only pre-approves tools; the deny list or a permission callback is what guarantees nothing else runs.
</details>

2. A team wants every file write by their agent to be checked against a company policy and logged, and they do not want to depend on Claude following instructions. What SDK feature fits, and why?

<details>
<summary>Answer</summary>

A `PreToolUse` hook (optionally with `PostToolUse` for logging). Hooks execute deterministically in your code on every matching tool call and can deny it, whereas prompt instructions are probabilistic and can be ignored or overridden by injected content.
</details>

3. A user comes back the next day and wants to continue yesterday's code-review conversation exactly where it ended. Separately, the agent should remember the user's preferred coding style across all future conversations. Which mechanism handles each?

<details>
<summary>Answer</summary>

Resuming with the saved session ID restores yesterday's conversation transcript. The long-term style preference should go in memory (the memory tool pattern from lesson 022, or a CLAUDE.md file) because it must persist across sessions rather than live in one transcript.
</details>

## Key takeaways

- The Agent SDK packages Claude Code's agent harness (loop, built-in tools, compaction, permissions, hooks, subagents, MCP, sessions) as a library.
- Use it for agents that take actions; use the Messages API for simple calls and fixed workflows.
- Control access with allow lists, deny lists, permission modes, and callbacks, applying least privilege; `bypassPermissions` is for sandboxes only.
- Use hooks for rules that must always hold, and prompts for guidance.
- Sessions resume or fork one conversation; memory carries knowledge across conversations.
- Cap turns and cost, sandbox the environment, log everything, and evaluate changes.
