# Advanced MCP: Lifecycle, Authorization, Client Features, and Production Concerns

## What you will learn

- How an MCP session starts: initialization, capability negotiation, and change notifications
- How remote MCP servers are secured with OAuth, and why token passthrough is forbidden
- What the client-side features (sampling, roots, elicitation) do and who controls them
- How to design MCP servers for production: tool annotations, pagination, errors, and the confused-deputy risk
- How to choose between MCP, a plain tool, and a Claude Code skill

## Explanation

Lesson 005 covered the basics: tools, resources, prompts, and transports. This lesson covers what happens around them, which is where architect-level exam questions live.

### 1. Session lifecycle and capability negotiation

An MCP connection is **stateful** and uses JSON-RPC 2.0. It starts with a handshake:

```
client -> server: initialize   { protocolVersion, capabilities, clientInfo }
server -> client: result       { protocolVersion, capabilities, serverInfo }
client -> server: notifications/initialized
```

- Each side **declares capabilities**. A server might say `{"tools": {"listChanged": true}, "resources": {"subscribe": true}}`. A client might declare `sampling`, `roots`, or `elicitation`.
- Neither side may use a feature the other did not declare. This is how old and new implementations stay compatible.
- If the protocol versions do not match, the client should disconnect or retry with a version both support.

**Change notifications.** If a server declared `listChanged`, it can send `notifications/tools/list_changed` when its tool set changes. The host then calls `tools/list` again. Tool lists are therefore dynamic, so do not cache them forever.

### 2. Authorization for remote servers

- **stdio servers** run locally and normally take credentials from environment variables. The MCP authorization spec does not apply to them.
- **Streamable HTTP servers** use **OAuth 2.1**. The MCP server acts as an OAuth *resource server*. The client discovers the authorization server through metadata, gets a token (authorization code flow with PKCE), and sends `Authorization: Bearer <token>` on each request.
- Tokens must be **audience-bound** to the MCP server. The server must validate that a token was issued for *it*.

**Token passthrough is an anti-pattern.** If your MCP server accepts a token and forwards it unchanged to a downstream API, you break audit trails and bypass downstream controls. Instead, the MCP server should get its *own* token for the downstream service.

**Confused deputy.** An MCP proxy that uses one shared client ID to talk to a third-party API can be tricked into granting access to the wrong user. The fix is per-client consent and strict validation of redirect URIs and state.

### 3. Client features: sampling, roots, elicitation

Most traffic goes client -> server. These features let the **server ask the client** for something:

| Feature | What it does | Who stays in control |
|---|---|---|
| **Sampling** | Server asks the host to run an LLM completion for it | The client chooses the model, and the user can review and approve the request. The server needs no API key. |
| **Roots** | Client tells the server which filesystem or URI boundaries it may work in | The client sets the boundaries |
| **Elicitation** | Server asks the user for missing input mid-task (a form or a confirmation) | The user can accept, decline, or cancel |

Example: a `deploy` tool may use elicitation to ask "Which environment?" instead of guessing. Elicitation must **not** be used to collect secrets such as passwords or API keys.

Why sampling matters architecturally: the cost and the model choice stay with the host, and the server stays model-agnostic.

### 4. Production design of servers

**Tool annotations** are hints about behavior:

```python
@mcp.tool(annotations={"readOnlyHint": True, "destructiveHint": False,
                       "idempotentHint": True, "openWorldHint": False})
def get_order(order_id: str) -> dict: ...
```

A host can use them to decide whether to ask for approval. But they are **hints from the server, not guarantees**. A client must not trust annotations from an untrusted server for security decisions.

**Two kinds of errors:**

- *Protocol errors* (JSON-RPC errors): unknown tool, malformed request.
- *Tool execution errors*: the call ran but failed. Return a normal result with `isError: true` and a helpful message. Claude can read this and retry or adapt, the same pattern as `is_error` on a `tool_result` (lesson 002).

**Other practices:**

- **Paginate** large resource and tool lists with cursors.
- Return **small, structured results** (lesson 019). Large outputs fill the context window (lesson 010).
- Declare an `outputSchema` when you want typed, structured results.
- Log to **stderr** on stdio servers. Anything written to stdout that is not a protocol message corrupts the stream.
- Keep tool counts low, or let the host defer-load tool definitions, to avoid tool overload.

### 5. MCP vs. other extension points

| Need | Best fit |
|---|---|
| One function in your own app | Plain tool definition |
| Reusable integration shared across hosts and teams | MCP server |
| Instructions or a procedure for Claude Code, loaded on demand | Skill (lesson 013) |
| Deterministic action on an event, with no model choice | Hook (lesson 006) |
| Server-side tools run by Anthropic | Server tools (lesson 025) |

MCP also appears in the API: the **MCP connector** lets the Messages API connect to a remote MCP server directly, so you do not write the client loop yourself. It supports remote HTTP servers only, not local stdio.

## Common exam traps

- **"stdio servers need OAuth."** No. OAuth is for remote HTTP servers. stdio servers use local environment credentials.
- **Passing the user's token through to a downstream API.** This is the named anti-pattern. Use a separate downstream token.
- **Trusting `readOnlyHint` from any server.** Annotations are untrusted hints unless the server is trusted.
- **Caching `tools/list` forever.** Servers can announce changes via `list_changed`.
- **Using elicitation to ask for passwords or API keys.** This is prohibited.
- **Printing debug output to stdout in a stdio server.** It breaks the JSON-RPC stream. Use stderr.
- **Thinking sampling gives the server its own model access.** The client mediates every request and can refuse it.
- **Treating server-returned text as trusted.** It is still prompt-injection surface (lesson 008).

## Practice questions

1. Your remote MCP server receives a user's bearer token and forwards the same token to an internal billing API. What is wrong, and what is the fix?
2. An MCP server wants to summarize a document with an LLM but has no API key and should not choose the model. Which feature fits, and who decides whether the request runs?
3. A stdio MCP server randomly breaks with JSON parse errors in the host after a developer adds `print("debug")` statements. Why?

<details>
<summary>Answers</summary>

1. This is **token passthrough**. It bypasses downstream controls and breaks audit and audience binding. The MCP server should validate that the token's audience is itself, then use its own credential (or a properly exchanged token) for the billing API.
2. **Sampling**. The server sends a sampling request, and the *client/host* picks the model and can let the user review or reject it.
3. On stdio, stdout carries the JSON-RPC messages. Stray prints corrupt the stream. Log to stderr instead.

</details>

## Key takeaways

- MCP sessions start with `initialize` and capability negotiation. Only use features both sides declared.
- Remote servers use OAuth 2.1 with audience-bound tokens. stdio servers use local credentials.
- Never pass client tokens through to downstream APIs.
- Sampling, roots, and elicitation let servers ask the client for model output, boundaries, or user input, with the client and user in control.
- Tool annotations are hints, not guarantees. Return execution failures as `isError` results so Claude can recover.
- Log to stderr, paginate, keep results small, and handle `list_changed`.
