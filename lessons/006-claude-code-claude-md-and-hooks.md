# Claude Code Configuration: CLAUDE.md, Permissions, and Hooks

## What you will learn

- What `CLAUDE.md` is for and where Claude Code looks for it
- The difference between instructions (advisory) and hooks (deterministic)
- How hook events, matchers, and exit codes work
- How permissions and settings files layer together
- When to choose CLAUDE.md, a hook, or a permission rule

## Explanation

### CLAUDE.md: persistent instructions

Claude has no memory between sessions. `CLAUDE.md` is a Markdown file that Claude Code loads into context at the start of every session, so you write project knowledge once instead of repeating it.

Common locations (all are combined, more specific ones add detail):

| Location | Scope |
|---|---|
| `~/.claude/CLAUDE.md` | Your personal preferences, all projects |
| `./CLAUDE.md` | Project, committed and shared with the team |
| `./CLAUDE.local.md` | Personal project notes, not committed |
| Subdirectory `CLAUDE.md` | Loaded when Claude works in that folder |

Good content: build/test commands, code style, architecture notes, "never do X" rules. Keep it short and specific. It consumes context on every turn, and long vague files get followed less reliably. You can import other files with `@path/to/file`.

```markdown
# Project notes
- Run tests with `npm test`; lint with `npm run lint`.
- API handlers live in `src/api/`; never edit `src/generated/`.
```

Key point: CLAUDE.md is **guidance**. Claude usually follows it but is not forced to.

### Hooks: guaranteed behavior

A hook is a shell command (or similar handler) that the Claude Code **harness** runs at a lifecycle event. Because the harness runs it, not the model, it happens every time. Typical events:

- `PreToolUse`: before a tool runs; can block it
- `PostToolUse`: after a tool succeeds (e.g. run a formatter)
- `UserPromptSubmit`: when the user sends a prompt
- `Stop` / `SubagentStop`: when Claude finishes responding
- `SessionStart`: at session start (e.g. inject context)

Configuration lives in `settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npx prettier --write \"$CLAUDE_PROJECT_DIR\"" }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/hooks/block-rm.sh" }]
      }
    ]
  }
}
```

The hook receives event details as JSON on stdin. Exit codes matter:

- `0`: success, continue
- `2`: **blocking** error; for `PreToolUse` the tool call is denied and stderr is fed back to Claude so it can adjust
- other non-zero: non-blocking error, shown to the user, execution continues

Hooks can also print structured JSON to make finer decisions (allow, deny, ask, add context).

### Permissions and settings layering

`settings.json` also holds `permissions` with `allow`, `ask`, and `deny` rules such as `Bash(npm test:*)` or `Read(./.env)`. Deny rules win over allow. Settings layer by precedence: managed/enterprise policy (cannot be overridden) > command-line flags > local project (`.claude/settings.local.json`) > shared project (`.claude/settings.json`) > user (`~/.claude/settings.json`).

### Choosing the right tool

- "Prefer our naming style" → CLAUDE.md
- "Always run the formatter after edits" → PostToolUse hook
- "Never read `.env` / never run `rm -rf`" → deny permission rule and/or PreToolUse hook
- "Org-wide mandatory policy" → managed settings

## Common exam traps

- **Relying on CLAUDE.md for hard guarantees.** It is advisory. Anything that must always or never happen belongs in a hook or permission rule.
- **Wrong exit code.** Exit 1 does not block a tool call; exit 2 does.
- **Thinking the model runs hooks.** The harness does, so they fire even if Claude "forgets".
- **Putting secrets or personal prefs in the shared CLAUDE.md.** Use `CLAUDE.local.md` or the user file.
- **Bloated CLAUDE.md.** More text is not better; it costs context and dilutes important rules.
- **Matcher confusion.** Matchers apply to tool names (`Bash`, `Edit|Write`), not file paths; path checks go in the hook script.

## Practice questions

<details>
<summary>Show questions and answers</summary>

**Q1.** Your team requires that every file Claude edits is auto-formatted. Instructions in CLAUDE.md work most of the time but occasionally are skipped. What is the best fix?

**A1.** Add a `PostToolUse` hook with matcher `Edit|Write` that runs the formatter. Hooks are executed by the harness deterministically, unlike advisory CLAUDE.md text.

**Q2.** A `PreToolUse` hook script detects a dangerous command and exits with code 1. The command still runs. Why, and what should change?

**A2.** Exit code 1 is a non-blocking error. The script should exit with code 2 (writing the reason to stderr) or return JSON with a deny decision, so the call is blocked and Claude sees the reason.

**Q3.** Where should a developer put their personal preference for verbose explanations so it applies to all projects without affecting teammates?

**A3.** In the user-level `~/.claude/CLAUDE.md`, not the committed project `CLAUDE.md`.

</details>

## Key takeaways

- CLAUDE.md = persistent, advisory context; keep it concise and specific.
- Hooks = deterministic harness-run commands on lifecycle events; exit code 2 blocks.
- Permissions use allow/ask/deny rules; deny wins, and managed policy cannot be overridden.
- Pick the mechanism by strength of guarantee needed: instruction < permission rule < hook < managed policy.
