# Agent Memory and State Persistence: Memory Tool, Scratchpads, and Checkpoints

## What you will learn

- Why Claude has no memory between API calls, and what that means for long-running agents
- The difference between short-term state (the context window) and long-term memory (external storage)
- How the client-side memory tool pattern works: Claude asks, your code reads and writes files
- How scratchpads and checkpoints let an agent resume after a crash or a context reset
- How to keep memory safe: validate paths, limit what is stored, and treat memory as untrusted input

## Explanation

### The model is stateless

Every Messages API call starts fresh. Claude only "knows" what is in the request: the system prompt, the messages, and tool results. Anything you want available later must be stored by your application and sent again (or fetched on demand).

Two kinds of state matter for agents:

| Kind | Where it lives | Lifetime | Example |
|---|---|---|---|
| Short-term (working) | Context window | One session, limited by token budget | Recent messages, tool results |
| Long-term (persistent) | Files, database, memory store | Across sessions and context resets | User preferences, project decisions, task progress |

Lesson 010 covered trimming and compacting the context. Memory is the complement: instead of only shrinking context, you move important facts *out* of it and bring them back when needed.

### The memory tool pattern

Anthropic provides a memory tool that is **client-side**: Claude emits tool calls such as view, create, str_replace, insert, delete, and rename against a `/memories` directory, and *your code* executes them against real storage (files, a database, encrypted blobs). Claude never touches your storage directly.

Typical flow:

1. At the start of a task, Claude checks `/memories` for earlier progress.
2. While working, it saves notes: decisions made, files done, open questions.
3. If the context is cleared or compacted, or a new session begins, Claude re-reads the notes and continues.

Your handler is ordinary tool-use code (lesson 002):

```python
import pathlib

ROOT = pathlib.Path("/var/agent-memory/user-42").resolve()

def safe_path(virtual: str) -> pathlib.Path:
    p = (ROOT / virtual.lstrip("/").removeprefix("memories/")).resolve()
    if ROOT not in p.parents and p != ROOT:
        raise ValueError("path escapes memory root")   # blocks ../../etc/passwd
    return p

def handle_memory(cmd: dict) -> str:
    if cmd["command"] == "view":
        return safe_path(cmd["path"]).read_text()
    if cmd["command"] == "create":
        p = safe_path(cmd["path"]); p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(cmd["file_text"]); return "created"
    ...
```

Because the storage is yours, you decide scoping (per user, per project), retention, encryption, and size limits.

### Scratchpads and progress files

For a long task, have the agent keep a structured progress file instead of relying on its recollection:

```
# progress.md
- [x] Migrate auth module
- [x] Migrate billing module
- [ ] Migrate reports module (blocked: needs schema decision)
Decision: use UUIDs, not integers.
```

A fresh context (new session, or a subagent) can read this file and be productive immediately. Claude Code uses the same idea: CLAUDE.md is persistent project memory loaded each session (lesson 006).

### Checkpoints for resumability

Agent loops (lesson 004) can fail mid-run. Persist after each meaningful step:

- the message history, or a summary plus the progress file
- completed tool calls and their results, keyed by ID
- the step number

On restart, load the last checkpoint and continue. Make tool calls **idempotent** (or check "was this already done?") so a retry does not charge a customer twice.

### Memory versus the alternatives

- **Put it in the system prompt**: fine for small, stable facts; expensive if large (use prompt caching, lesson 003).
- **RAG** (lesson 015): best for big, read-mostly knowledge bases.
- **Memory tool / notes**: best for facts the *agent itself* learns and updates.
- **Context editing / compaction**: removes stale tool results from context to make room; it is not a substitute for saving what must be remembered.

### Safety of memory

Memory is a prompt-injection persistence channel (lesson 008). If an agent writes text from a malicious web page into memory, that text is read back in every later session.

- Treat memory contents as untrusted data, not as instructions with system authority.
- Do not store secrets or sensitive personal data unless required; apply retention and deletion rules.
- Validate paths (see above) and cap file sizes and counts.
- Scope memory per user or tenant so one user's notes never leak to another.
- Log memory reads and writes (lesson 021) so you can audit what the agent "learned".

## Common exam traps

- **"Claude remembers earlier conversations automatically."** No. The API is stateless; you must resend or store state.
- **"The memory tool stores data on Anthropic's servers."** The memory tool is client-side; your application implements storage.
- **Skipping path validation.** Directory traversal in a memory handler is a classic vulnerability.
- **Treating a bigger context window as memory.** It is still lost when the session ends and costs tokens every turn.
- **Non-idempotent tools with checkpoint replay.** Resuming can repeat side effects.
- **Trusting stored notes blindly.** Poisoned memory persists across sessions.

## Practice questions

1. An agent works on a migration over several sessions and keeps forgetting which modules are finished. What is the simplest robust fix?
2. Where does the data written through the memory tool actually live, and who is responsible for securing it?
3. Your agent crashes after charging a customer but before saving its checkpoint. What design prevents a double charge on resume?

<details>
<summary>Answers</summary>

1. Have the agent maintain a progress file (via the memory tool or a notes file) updated after each module, and read it at the start of every session.
2. In storage your application controls, since the tool is client-side. You are responsible for path validation, access scoping, encryption, retention, and size limits.
3. Make the charge idempotent (for example, an idempotency key derived from the task and step) and/or record intent before executing, so a replay detects the earlier charge instead of repeating it.

</details>

## Key takeaways

- The API is stateless; context is short-term, external storage is long-term.
- The memory tool is client-side: Claude requests operations, your code performs them.
- Progress files and checkpoints make long agent tasks resumable; pair them with idempotent tools.
- Memory is untrusted input and a persistence vector for injection: validate, scope, limit, and log.
- Choose the right store: system prompt for small stable facts, RAG for large knowledge, memory for what the agent learns.
