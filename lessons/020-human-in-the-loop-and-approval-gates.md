# Human-in-the-Loop: Approval Gates, Risk Tiers, and Safe Autonomy for Agents

## What you will learn

- Why autonomy should be matched to the risk of each action, not set once for the whole agent
- How to classify tools into risk tiers and place approval gates in the agentic loop
- How to pause, resume, and record approvals without losing agent state
- How to avoid approval fatigue and how to design good escalation paths
- Which controls must be enforced in code rather than in the prompt

## Explanation

### Autonomy is a dial, not a switch

An agent that asks permission for every step is useless. An agent that never asks is dangerous. The architect's job is to decide, **per action**, how much autonomy is acceptable. A simple model uses risk tiers:

| Tier | Examples | Policy |
|------|----------|--------|
| Read-only | search, read file, get order | Run automatically |
| Reversible write | create draft, edit file in a branch | Run automatically, log it |
| Costly or hard to undo | send email, refund, deploy, delete | Require human approval |
| Forbidden | drop production database | Block entirely |

Two questions place an action in a tier: **How bad is it if the model is wrong?** and **Can it be undone?**

### Where the gate lives: in the loop, in code

Lesson 004 showed the agentic loop. An approval gate is a check between "Claude requested a tool" and "your code executes it":

```python
NEEDS_APPROVAL = {"issue_refund", "send_email", "delete_record"}

for block in response.content:
    if block.type != "tool_use":
        continue
    if block.name in NEEDS_APPROVAL:
        decision = ask_human(block.name, block.input)   # blocks or queues
        if not decision.approved:
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": f"Denied by reviewer: {decision.reason}",
                "is_error": True,
            })
            continue
    results.append(run_tool(block))
```

Key points:

- Every `tool_use` still needs a matching `tool_result`, including denials. Skipping one breaks the conversation.
- Return the denial reason to Claude. It can then change plan, ask the user, or pick a safer route.
- The gate is **code you control**. A system prompt line like "always ask before refunding" is guidance, not enforcement. Prompt injection (lesson 008) can defeat it.

### Show the reviewer what matters

An approval prompt is only useful if the human can judge it. Show the exact tool name and arguments, the agent's stated reason, and the expected effect (for example "Refund $240 to order 1182, cannot be reversed"). Do not show only "Agent wants to continue?". Approve the *concrete action*, not a vague intent.

Validate what was approved: if the reviewer approved a $240 refund, execute exactly that input, not one Claude regenerates later.

### Pause and resume

Approval may take minutes or days. Do not hold a process open. Persist the conversation `messages`, the pending `tool_use` block, and the task ID, then resume when the decision arrives. This is a checkpoint. Resuming means appending the `tool_result` and calling the API again. Remember that prompt cache entries expire (lesson 003), so a long pause costs a cache miss on resume.

### Avoid approval fatigue

If humans click "approve" 200 times a day, they stop reading. Reduce prompts without reducing safety:

- Auto-approve low-risk tiers and batch related approvals into one plan review.
- Use allowlists and scoped permissions (for example, allow `git status` but ask for `git push`). Claude Code's permission rules and hooks work this way (lesson 006).
- Add thresholds: refunds under $20 run automatically, above that need approval.
- Review the plan once up front for long tasks, then gate only the risky steps.

### Escalation and audit

Humans are not only gatekeepers; the agent should also **escalate** when it is stuck, uncertain, or sees something unexpected. Give it an explicit tool such as `request_human_help(reason, context)` and tell it when to use it. Always log who approved what, when, with which inputs. Audit logs support debugging, evaluation (lesson 007), and compliance.

### Prefer guardrails that make mistakes cheap

Approval is one layer. Combine it with least-privilege credentials, dry-run modes, idempotency keys (lesson 016), sandboxes, and rate or spend limits. A human cannot catch everything, so limit the blast radius too.

## Common exam traps

- **Enforcing approval only in the system prompt.** The correct answer puts the gate in application code or tool permissions.
- **Dropping the `tool_result` when a request is denied.** Every `tool_use` needs a result; return an error result with the reason.
- **Approval on everything.** This causes fatigue; the better design uses risk tiers.
- **Approving intent instead of the exact action.** Reviewers must see concrete arguments, and the approved input must be the one executed.
- **Holding a request open while waiting for a human.** Checkpoint state and resume instead.
- **Treating human review as the only safety layer.** Combine it with least privilege and limits.

## Practice questions

1. An agent can read tickets, draft replies, and issue refunds. Which actions should require human approval, and why?
2. A reviewer denies a `delete_record` call. What should your loop send back to Claude?
3. Your approval step sometimes takes two days. How should you architect the wait?

<details>
<summary>Answers</summary>

1. Issuing refunds (costly and hard to undo) should require approval, perhaps only above a threshold. Reading tickets is read-only and drafting replies is reversible, so both can run automatically (with logging).
2. A `tool_result` with the matching `tool_use_id`, with `is_error: true` and a clear denial reason. Claude can then adapt, for example by asking the user or choosing another approach. Omitting the result would make the next request invalid.
3. Persist the messages, the pending tool call, and a task ID, then end the process. When the decision arrives, load the state, append the `tool_result`, and continue the loop. Do not keep a connection or process open, and expect a prompt cache miss on resume.

</details>

## Key takeaways

- Match autonomy to risk: automatic for read-only, gated for irreversible or costly, blocked for forbidden.
- Enforce gates in code between `tool_use` and execution; prompts alone are not enforcement.
- Always return a `tool_result`, including denials, with the reason.
- Show reviewers the concrete action, and execute exactly what was approved.
- Checkpoint state for long waits, reduce approval fatigue with thresholds and allowlists, and log every decision.
