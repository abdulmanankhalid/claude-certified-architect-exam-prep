# Model Context Protocol (MCP): A Standard Way to Connect Tools and Data

## What you will learn

- What MCP is and which problem it solves
- The three roles: host, client, and server
- The three server primitives: tools, resources, and prompts
- How MCP relates to the tool use you saw in lesson 002
- How transports, scoping, and security work in practice

## Explanation

### The problem

In lesson 002 you defined tools yourself and wrote the code that runs them. If you have N applications and M data sources, you end up writing N x M custom integrations. The **Model Context Protocol (MCP)** is an open standard that turns this into N + M: a data source exposes one MCP **server**, and any MCP-capable app can use it.

### Roles

- **Host**: the app the user interacts with (Claude Code, Claude Desktop, your own agent).
- **Client**: a connector inside the host. Each client keeps a 1:1 connection with one server.
- **Server**: a program that exposes capabilities (a GitHub server, a database server, a filesystem server).

### What a server can expose

| Primitive | Controlled by | Purpose | Example |
|---|---|---|---|
| **Tools** | The model | Actions the model may call | `create_issue`, `run_query` |
| **Resources** | The application | Read-only data identified by a URI | `file:///logs/app.log` |
| **Prompts** | The user | Reusable prompt templates | `/summarize-pr` |

Tools are the most common. Under the hood, an MCP tool is still a tool definition (name, description, JSON input schema) given to Claude. The host discovers them with `tools/list`, and when Claude emits a `tool_use` block, the host forwards it to the server with `tools/call` and returns the result as a `tool_result`. The agentic loop from lesson 004 is unchanged; only the source of the tools differs.

### Transports

- **stdio**: the host launches the server as a local subprocess. Good for local tools; no network exposure.
- **Streamable HTTP**: the server is remote. Use it for shared or hosted servers, with authentication (OAuth is the standard approach).

Messages are JSON-RPC 2.0 in both cases.

### Tiny server example (Python SDK)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("orders")

@mcp.tool()
def get_order(order_id: str) -> dict:
    """Look up an order by ID and return its status."""
    return {"id": order_id, "status": "shipped"}

if __name__ == "__main__":
    mcp.run()  # stdio by default
```

The docstring becomes the tool description, and type hints become the input schema. Good descriptions matter just as much as in lesson 002.

### Using MCP in Claude Code

```bash
claude mcp add orders -- python orders_server.py
```

Project-shared servers can be committed in `.mcp.json` at the repo root so the whole team gets them. Scopes are roughly: local (you, this project), project (shared via `.mcp.json`), and user (you, all projects). Never put secrets directly in `.mcp.json`; use environment variable expansion.

### Design and safety tips

- Too many tools bloat the context and confuse tool selection. Expose a focused set.
- Tool results from a server are **untrusted input**. A malicious or compromised server (or data it returns) can contain prompt injection.
- Only install servers you trust, and apply least privilege (read-only credentials where possible).
- Keep human approval on destructive tools.

## Common exam traps

- **"MCP replaces tool use."** No. MCP is a way of supplying and executing tools; the model still uses ordinary tool use.
- **Confusing the primitives.** Tools = model-controlled actions; resources = application-controlled data; prompts = user-controlled templates.
- **Thinking one client talks to many servers.** Each client holds a 1:1 connection; a host runs multiple clients.
- **Choosing stdio for a shared remote service**, or HTTP for a simple local script. Match the transport to the deployment.
- **Trusting server output.** Tool results can carry injected instructions.
- **Committing credentials in `.mcp.json`.**

## Practice questions

1. A team has 5 internal apps and 8 data sources. Why does MCP reduce integration work?
2. Which MCP primitive fits "expose a read-only company policy document by URI for the app to attach as context"?
3. Your MCP server runs on a shared hosted service used by many developers. Which transport and what auth approach?

<details>
<summary>Answers</summary>

1. Without a standard you need up to 5 x 8 = 40 custom integrations. With MCP you build 8 servers and 5 clients (13 pieces), since each side implements the protocol once.
2. A **resource**, because it is read-only data addressed by a URI and chosen by the application. A tool would be for model-initiated actions.
3. **Streamable HTTP** with authentication, typically OAuth. stdio only suits local subprocesses.

</details>

## Key takeaways

- MCP is an open standard that turns N x M integrations into N + M.
- Host contains clients; each client connects 1:1 to a server.
- Servers expose tools (model), resources (application), and prompts (user).
- MCP tools become ordinary tool definitions; the agentic loop is the same.
- stdio for local, Streamable HTTP for remote; share config via `.mcp.json` without secrets.
- Treat server output as untrusted and keep tool sets small and least-privileged.
