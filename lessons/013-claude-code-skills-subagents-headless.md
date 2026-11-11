# Claude Code Extensibility: Skills, Custom Subagents, Slash Commands, and Headless Mode

## What you will learn

- How skills, custom subagents, slash commands, and CLAUDE.md differ, and when to pick each
- How a skill is structured and why only its short description is loaded up front
- How to define a custom subagent with its own prompt, tools, and isolated context
- How to run Claude Code non-interactively (`claude -p`) in scripts and CI
- How to avoid the common "wrong extension point" exam traps

## Why this matters

Lesson 006 covered CLAUDE.md, permissions, and hooks. Real projects also need
reusable procedures, specialist helpers, and automation. Claude Code gives you
several extension points. The exam often describes a need and asks which one fits.

## The extension points at a glance

| Mechanism | What it is | Loaded when | Best for |
|---|---|---|---|
| CLAUDE.md | Always-on project instructions | Every session start | Conventions, commands, rules that always apply |
| Skill | A folder with a `SKILL.md` and optional files/scripts | Description at start; body only when relevant | Reusable procedures (e.g. "write release notes") |
| Slash command | A saved prompt you trigger by name (`/name`) | When you invoke it | Repeated, user-triggered prompts |
| Custom subagent | A specialist with its own prompt, tools, and context window | When delegated to | Noisy or specialized side tasks |
| Hook | A script run on an event | On the event, deterministically | Rules that must never be skipped |
| MCP server | External tools and data (lesson 005) | At connection | Reaching systems outside the repo |

Rule of thumb: **CLAUDE.md = always true. Skill = know-how loaded on demand.
Subagent = separate worker. Hook = guaranteed enforcement.**

## Skills

A skill is a directory containing a `SKILL.md` file. The file starts with YAML
frontmatter, then instructions:

```markdown
---
name: release-notes
description: Draft release notes from merged PRs. Use when the user asks for a changelog or release notes.
---

# Release notes

1. Run `git log --merges <last-tag>..HEAD`.
2. Group changes under Added / Changed / Fixed.
3. Follow the template in `template.md` (same folder).
```

Key idea: **progressive disclosure.** At startup Claude sees only each skill's
`name` and `description`. It reads the full body (and any extra files or scripts
in the folder) only when the task matches. So you can install many skills
without bloating the context window (lesson 010).

Consequences:

- The `description` is what triggers the skill. Say *what it does* and *when to use it*. A vague description means the skill rarely fires.
- Long reference material belongs in separate files in the skill folder, not in CLAUDE.md.
- Skills can be personal (in your home config) or project-level (committed to the repo so the team shares them).

## Custom subagents

A subagent is defined as a Markdown file with frontmatter, typically in the
project's `.claude/agents/` folder:

```markdown
---
name: code-reviewer
description: Reviews diffs for bugs and style. Use proactively after code changes.
tools: Read, Grep, Glob
---

You are a careful reviewer. Read the changed files, look for bugs and
unclear code, and reply with a short ranked list of issues.
```

What you get (building on lesson 009):

- **Isolated context.** The subagent's file reads and searches stay in its own window. Only its final report returns to the main conversation.
- **Restricted tools.** Listing `tools` gives least privilege. A reviewer that only has read tools cannot edit files.
- **A focused system prompt** tuned to one job.
- **The `description` drives delegation**, just as with skills.

Subagents do not remove cost: they use tokens too, and the main agent only sees
the summary, so a subagent must be told exactly what to return.

## Slash commands

A custom slash command is a Markdown file whose body is a prompt. Typing
`/name` runs it, and arguments can be passed in. Use one when the *user*
decides when it runs and the prompt is a fixed template, such as `/review-pr 123`.
Use a skill when Claude should decide on its own when the know-how applies.

## Headless mode and CI

Claude Code can run without the interactive UI using the print flag:

```bash
claude -p "Summarize the failing tests in this repo" --output-format json
```

- `-p` (print) runs one task and exits, which suits scripts, cron jobs, and CI.
- `--output-format` can be `text`, `json`, or `stream-json`, so other programs can parse the result.
- Because no human is there to click "allow", you must preconfigure permissions, for example with an explicit allowlist of tools, and keep them as narrow as the job allows.
- Pair it with hooks for guaranteed checks, and with the safety ideas in lesson 008: in CI, Claude may read untrusted text such as PR descriptions, so never give it broad write or secret access. See `ci-notes.md` for the CI side.

## Choosing: a worked example

- "Every edit must be followed by the linter." Hook (must be deterministic).
- "Explain how we write database migrations, only when relevant." Skill.
- "Search the whole codebase for deprecated API use without flooding my context." Subagent.
- "I keep retyping the same 'draft a changelog' prompt." Slash command (or a skill).
- "Our code style is 4-space indentation." CLAUDE.md.

## Common exam traps

1. **Putting everything in CLAUDE.md.** It loads every session and costs context every time. Procedures needed only sometimes belong in skills.
2. **Using CLAUDE.md or a skill to enforce a hard rule.** They are instructions Claude may not follow perfectly. Only hooks (and permissions) are deterministic.
3. **Vague skill or subagent descriptions.** Delegation and triggering depend on the description.
4. **Assuming a subagent shares the main context.** It starts fresh and sees only what it is given.
5. **Granting subagents all tools by default.** Restrict tools to what the job needs.
6. **Running `claude -p` in CI with permissive settings** while it reads untrusted input. That is a prompt injection risk.

## Practice questions

<details>
<summary>Show questions and answers</summary>

**Q1.** Your team has a 3,000-word guide for writing API documentation that is needed only occasionally. Where should it go, and why?

**A1.** In a skill, with a clear description such as "Use when writing or editing API docs". Only the short description is loaded at startup; the full guide is read on demand. Putting it in CLAUDE.md would spend context on every session.

**Q2.** You need Claude to search a huge monorepo for every use of a deprecated function and report only the file list, without filling the main conversation with search output. What do you use?

**A2.** A custom subagent (with read-only tools such as Read, Grep, Glob). It works in its own context window and returns only a concise result.

**Q3.** A pipeline job must call Claude Code with no human present and feed the result to another program. What do you configure?

**A3.** Run headless with `claude -p "<task>"` and `--output-format json` so the output is parseable. Preconfigure a narrow tool allowlist because nobody can approve prompts, and treat any untrusted text Claude reads as a possible injection.

</details>

## Key takeaways

- CLAUDE.md is always loaded; skills load on demand; subagents run in separate contexts; hooks enforce rules deterministically.
- A skill is a folder with `SKILL.md`; the `description` decides when it is used.
- Custom subagents give isolated context, restricted tools, and a focused prompt.
- `claude -p` with `--output-format` enables scripting and CI, but needs explicit, minimal permissions.
- Match the need to the mechanism: guaranteed means hook, sometimes means skill, always means CLAUDE.md, separate worker means subagent.
