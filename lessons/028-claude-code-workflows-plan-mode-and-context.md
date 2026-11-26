# Claude Code Workflows: Plan Mode, Context Hygiene, and Parallel Sessions

## What you will learn

- How the explore, plan, implement, verify workflow reduces wrong-direction work
- What permission modes (default, accept edits, plan) do and when to use each
- How to keep a Claude Code session's context clean with `/clear`, `/compact`, and subagents
- How to run parallel sessions safely with git worktrees
- How to give Claude a way to verify its own work

## Explanation

Lessons 006 and 013 covered configuration (CLAUDE.md, hooks, skills, subagents). This lesson is about **how you work** with Claude Code. The same tools give very different results depending on workflow.

### 1. Explore, plan, implement, verify

For anything bigger than a one-line fix, do not jump straight to editing:

1. **Explore**: ask Claude to read the relevant files and explain how they work. No changes yet.
2. **Plan**: ask for a step-by-step plan. Review and correct it.
3. **Implement**: let Claude make the changes, following the approved plan.
4. **Verify**: run tests, linters, or type checks, and fix failures.

Skip planning when the diff is small and obvious (a typo, a rename). Planning pays off when the change touches several files or the approach is uncertain.

### 2. Permission modes and plan mode

| Mode | Behavior |
|---|---|
| Default | Asks before edits and risky commands (subject to your allow/deny rules) |
| Accept edits | File edits are auto-approved; other risky actions still prompt |
| **Plan mode** | Read-only: Claude can read and search but cannot edit files or run mutating commands. It produces a plan for you to approve |

Plan mode is a safety and quality tool: you can let Claude investigate an unfamiliar codebase with no risk of changes. You switch modes in the session (for example with Shift+Tab) or start in one with a CLI flag. Permission rules from lesson 006 still apply; deny rules always win.

### 3. Context hygiene

Claude Code's context window fills with file contents, command output, and conversation. As it fills, quality drops and cost rises (lesson 010). Tools to manage it:

| Tool | Use it when |
|---|---|
| `/clear` | Switching to an unrelated task. Starts fresh; CLAUDE.md is reloaded |
| `/compact [focus]` | Same task, but history is long. Summarizes the conversation; you can say what to preserve |
| **Subagents** | A side investigation would read many files. The subagent's work stays in its own context; only a summary returns (lesson 009) |
| Specific references | Point to exact files instead of "look around the repo" |

A rule of thumb: one task per session. If you have corrected Claude on the same mistake twice, the context is probably polluted; `/clear` and restate the task with a better prompt instead of piling on corrections.

Durable rules (build commands, conventions) belong in **CLAUDE.md**, not in the chat, because chat is lost on `/clear` and summarized on `/compact`.

### 4. Give Claude a way to verify

The highest-leverage habit is giving Claude a **check it can run itself**: a test command, a linter, a script, an expected output. Compare:

```text
Weak:   "Fix the date parsing bug."
Strong: "Parsing '2026-02-30' should raise ValueError. Add a failing test
         in tests/test_dates.py, then fix the code until `pytest tests/test_dates.py` passes."
```

With a verification target, Claude can loop (edit, run, read errors, edit) without you in the middle. This is the agentic loop from lesson 004 applied to coding. A **hook** (lesson 006) can enforce it, for example running the formatter or tests after edits.

### 5. Parallel sessions with git worktrees

Two sessions editing the same working directory will collide. A **git worktree** gives each session its own checkout and branch from the same repository:

```bash
git worktree add ../app-feature-a -b feature-a
git worktree add ../app-bugfix-b  -b bugfix-b
# run a separate `claude` session in each directory
```

Each session has its own files and context, and you merge via normal pull requests. This is the same isolation principle as multi-agent context isolation (lesson 009), at the filesystem level.

### 6. Headless and CI use

The same workflow runs non-interactively with `claude -p "<prompt>"` (lesson 013), for example in CI (see `ci-notes.md`). There is no human to approve prompts, so restrict tools with an explicit allowlist and give a verifiable goal.

## Common exam traps

- **"Plan mode lets Claude edit once I approve in chat."** Plan mode itself is read-only; you leave plan mode (or approve the plan) before edits happen.
- **Using `/compact` when switching tasks.** For an unrelated task, `/clear` is cleaner; compaction keeps a summary of stale work.
- **Putting permanent rules only in the conversation.** They vanish on `/clear`. Put them in CLAUDE.md.
- **Running two sessions in one directory.** Use separate worktrees.
- **Asking for a fix with no verification step.** The answer that adds a runnable test or check is usually better.
- **Thinking plan mode replaces permissions.** It is an additional safeguard, not a substitute for deny rules.

## Practice questions

1. You must understand an unfamiliar module before changing it, and you want a guarantee that nothing is modified during that investigation. What do you use?
2. You finished a feature in a long session and now want to start an unrelated bug fix. Should you use `/compact` or `/clear`, and why?
3. A teammate wants two Claude Code sessions working on different features of the same repo at the same time. What setup avoids conflicts?

<details>
<summary>Answers</summary>

1. **Plan mode**. It is read-only, so Claude can explore and propose a plan without editing files or running mutating commands.
2. **`/clear`**. The task is unrelated, so the old history is just noise that costs tokens and can distract the model. CLAUDE.md is reloaded after clearing. `/compact` is for continuing the same task with a shorter history.
3. Create a **git worktree** (and branch) per feature and run one session in each directory. Each has isolated files and context; merge through pull requests.

</details>

## Key takeaways

- Explore, plan, implement, verify: plan first when the change is multi-file or uncertain.
- Plan mode is read-only; permission rules and deny lists still apply.
- Keep context clean: one task per session, `/clear` between tasks, `/compact` within a task, subagents for noisy side investigations.
- Persistent instructions go in CLAUDE.md, not chat.
- Give Claude a runnable way to verify its work; this enables the agentic loop.
- Use git worktrees for parallel sessions.
