# Multi-Agent Architectures: Orchestrators, Subagents, and Context Isolation

## What you will learn

- When one agent is enough and when splitting work across several agents pays off
- The orchestrator-worker pattern and how subagents keep the main context clean
- How to brief a subagent, since it only knows what you tell it
- The cost, latency, and reliability trade-offs of multi-agent systems
- How subagents work in Claude Code and the Agent SDK

## Explanation

### Start with one agent

Lesson 004 showed the agentic loop: call Claude, run tools, repeat. A single agent with good tools solves most tasks. Add more agents only when you hit a real limit:

- **Context pressure**: a long task fills the context window with file dumps and search results.
- **Parallelism**: independent sub-tasks could run at the same time.
- **Specialization**: different steps need different tools, permissions, system prompts, or models.

Multi-agent systems usually use more tokens in total. Use them only when the benefit is clear.

### Orchestrator-worker pattern

One **orchestrator** (lead) agent plans and delegates. Several **workers** (subagents) each run their own agentic loop in a **fresh context** and return a short result.

```
User --> Orchestrator --> Subagent A (search auth code)    --> summary
                     |--> Subagent B (search billing code) --> summary
                     |--> Subagent C (read the docs)       --> summary
Orchestrator combines the summaries --> final answer
```

The key benefit is **context isolation**. A subagent may read 50 files, but the orchestrator only sees a 200-word summary. The main context stays small and focused.

A simple version: expose a "delegate" tool to the orchestrator.

```python
tools = [{
    "name": "run_subagent",
    "description": "Run a focused research task in a separate agent. "
                   "Returns a short summary. Use for tasks that need many reads.",
    "input_schema": {
        "type": "object",
        "properties": {
            "task": {"type": "string", "description": "Self-contained instructions"},
            "output_format": {"type": "string"},
        },
        "required": ["task"],
    },
}]

def run_subagent(task, output_format="Max 200 words"):
    # A brand-new messages list: the subagent sees none of the parent's history.
    return agent_loop(
        system="You are a research worker. Be concise.",
        messages=[{"role": "user", "content": f"{task}\n\nFormat: {output_format}"}],
        tools=READ_ONLY_TOOLS,   # least privilege
    )
```

### Briefing a subagent

A subagent starts with an empty history. Your task description is its entire world. A good brief includes:

1. The goal and why it matters
2. Relevant file paths, names, or facts already known
3. What has already been ruled out
4. The scope (what not to touch)
5. The exact output format and length you want

Vague briefs ("look into the bug") produce vague or duplicated work.

### Other orchestration patterns

| Pattern | Idea | Good for |
|---|---|---|
| Prompt chaining | Fixed sequence of calls, output feeds the next | Predictable pipelines |
| Routing | A cheap classifier picks a specialized handler | Mixed request types |
| Parallelization | Run independent calls at once and merge | Speed, or voting for confidence |
| Orchestrator-worker | Lead decides sub-tasks dynamically | Open-ended tasks |
| Evaluator-optimizer | One call drafts, another critiques, loop | Quality-sensitive output |

Prefer the simplest pattern that works. A fixed workflow is more predictable and cheaper than a free-roaming agent.

### Trade-offs

- **Cost**: each agent has its own loop and system prompt. Total tokens go up. Prompt caching (lesson 003) on shared prefixes helps.
- **Latency**: parallel subagents cut wall-clock time; sequential delegation adds it.
- **Model mix**: use a stronger model for orchestration and a smaller, faster one for simple workers.
- **Reliability**: errors compound across agents. Validate subagent outputs, and set limits on turns, tokens, and depth of delegation.
- **Security**: a subagent that reads untrusted content can be prompt-injected (lesson 008). Give it only the tools it needs, and treat its summary as data, not instructions.
- **Debugging**: log each subagent's task, tool calls, and result. Evaluate (lesson 007) both the whole system and individual workers.

### Subagents in Claude Code and the Agent SDK

In Claude Code you can define subagents as Markdown files with frontmatter (for example under `.claude/agents/`). Each has a name, a `description` that tells the main agent when to delegate, an optional restricted tool list, optionally a model, and a system prompt body.

```markdown
---
name: code-reviewer
description: Reviews diffs for bugs. Use after code changes.
tools: Read, Grep, Glob
---
You review code for correctness bugs. Report file, line, and a one-sentence reason.
```

Points to remember:

- The `description` drives automatic delegation, so write it clearly.
- A subagent runs in its own context window and returns its result to the parent.
- Restricting `tools` gives least privilege (a reviewer should not need Write).
- The Agent SDK lets you define subagents programmatically for your own applications.
- Hooks (lesson 006) can run around subagent activity for logging or policy checks.

## Common exam traps

- **"More agents is always better."** Wrong. Start simple; add agents for context, parallelism, or specialization.
- **Assuming a subagent inherits the parent's conversation.** It does not. Pass everything it needs in the brief.
- **Returning raw transcripts to the orchestrator.** This defeats context isolation. Return concise summaries.
- **Giving every subagent all tools.** Use least privilege.
- **Parallelizing dependent steps.** Only independent tasks can run concurrently.
- **Trusting subagent output blindly.** Validate it, especially when the subagent read untrusted content.
- **No stop conditions.** Cap turns, tokens, and delegation depth to avoid runaway loops and cost.

## Practice questions

1. An agent researching a large codebase keeps running out of context because of many file reads. What is the best architectural change?

<details>
<summary>Answer</summary>

Delegate the file-heavy exploration to subagents with fresh contexts, each returning a short summary to the orchestrator. This is context isolation. Simply switching to a bigger model does not fix the pattern.
</details>

2. A subagent keeps redoing work the orchestrator already did and returns off-target results. What is the most likely cause and fix?

<details>
<summary>Answer</summary>

The subagent starts with no history, so the brief was incomplete. Write a self-contained brief: goal, known facts, what is ruled out, scope, and the desired output format.
</details>

3. You are building a code-review subagent in Claude Code that must never modify files. How do you enforce this?

<details>
<summary>Answer</summary>

Restrict its tool list to read-only tools (such as Read, Grep, Glob) in the subagent definition, rather than only asking in the prompt. Tool restrictions are enforced; prompt instructions are not guaranteed.
</details>

## Key takeaways

- Use one agent by default; split when you need context isolation, parallelism, or specialization.
- Orchestrator-worker: the lead delegates, workers run in fresh contexts and return concise results.
- A subagent knows only its brief, so make briefs self-contained.
- Multi-agent systems cost more tokens; mix models, cache shared prefixes, and cap limits.
- Enforce least privilege through tool restrictions, and treat subagent output as untrusted data to validate.
- In Claude Code, subagents are Markdown files whose `description` drives delegation.
