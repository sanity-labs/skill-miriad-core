# Execute — Primary Tool-Calling Surface

The `execute` tool runs JavaScript that can call any tool as an async function — no inference round-trips between calls. This is the **primary way to call tools** on [REDACTED SECRET: GIT_AUTHOR_NAME].

## Why Execute

Tools are not loaded into the agent's context window. Instead:

1. **`list_tools("keyword")`** — discover available tools by name or description
2. **`execute({ script: "..." })`** — call tools as async functions in JavaScript

This keeps context windows lean (no tool definitions dumped in) and makes multi-step work dramatically faster:

| Approach | 7 tool calls |
|----------|-------------|
| Sequential (one per inference round) | ~30-60s |
| Execute script | ~850ms |

## Discovering Tools

Use `list_tools` to find tools — it works both as a direct tool and inside execute scripts:

```js
// Find dataset tools
const tools = await list_tools({ query: "dataset" });
// Returns: [{ name, description, inputSchema }, ...]

// Find sandbox tools
const tools = await list_tools({ query: "sandbox exec" });
```

Tools come from multiple sources — the platform `miriad` server (files, plan, messages, sandbox, datasets, config, vision) and any custom MCP servers registered in the space (e.g., Sanity). You don't need to know which server provides a tool — just search and call.

## Tool Names

All platform tools are available as bare async functions inside execute:

```js
await sandbox_exec({ sandbox: "s", command: "ls" })
await dataset_query({ dataset: "movies", query: "*" })
await plan_status({})
await glob({ pattern: "*.md" })
await send_message({ content: "hello" })
await web_search({ query: "react hooks" })
```

Custom MCP servers keep their namespace prefix:
```js
await sanity__query_documents({ ... })
```

Use `list_tools()` to see all available tools, `document_tool({ name: "sandbox_exec" })` for full parameter schemas.

## Core API

### Calling tools
```js
// All