# Stdio MCP Servers in Sandboxes

Agents can run **any stdio MCP server** from a sandbox — no permanent MCP configuration needed. Install the server, call tools on demand, tear down when done.

This is more flexible than mounted MCP servers: no OAuth, no slug registration, no persistent connections. Just `pip install` or `npm install` a server and start calling tools.

## Install `mcpcli`

[mcpcli](https://github.com/sanity-labs/mcpcli) is a TypeScript CLI for interacting with MCP servers. Zero-install via `npx`:

```bash
# No installation needed — npx downloads and caches on first run
npx github:sanity-labs/mcpcli tools <server-command>
```

Or install globally for faster repeated use:

```bash
npm install -g github:sanity-labs/mcpcli
```

### Why mcpcli over mcptools?

| | mcpcli | mcptools |
|---|---|---|
| Install | `npx` (zero-install) | Requires Go toolchain (~30s) |
| Runtime | Node.js (already in sandbox) | Go binary |
| Env vars | Passes full `process.env` to servers | SDK default whitelist only |
| Ecosystem | TypeScript/npm native | Go native |

`mcpcli` passes the full environment to child processes, so API keys like `FAL_KEY` just work. The Go-based `mcptools` uses the MCP SDK's default env whitelist (HOME, PATH, SHELL, TERM, USER, LOGNAME) which drops API keys.

## Using `mcpcli`

```bash
# List all tools a server provides
npx github:sanity-labs/mcpcli tools <server-command>

# Call a specific tool
npx github:sanity-labs/mcpcli call <tool-name> <server-command> -p '<json-params>'

# Output formats
npx github:sanity-labs/mcpcli tools <server-command> -f json     # compact JSON (pipeable)
npx github:sanity-labs/mcpcli tools <server-command> -f pretty   # indented JSON
```

### All Commands

| Command | Description |
|---------|-------------|
| `tools <server...>` | List available tools with parameter types |
| `call <tool> <server...> -p '{...}'` | Call a tool with JSON params |
| `resources <server...>` | List available resources |
| `prompts <server...>` | List available prompts |
| `get-prompt <name> <server...>` | Fetch a rendered prompt |
| `read-resource <uri> <server...>` | Read a resource by URI |

### Table Output

The default table format matches mcptools style — colored parameter display:

```
greet(name:str, [enthusiasm:int])
     Greet someone by name

list_files(path:str, [recursive:bool], [extensions:str[]])
     List files in a directory
```

Required params in green, optional in yellow with brackets. Type shortening: `str`, `int`, `bool`, `num`, `T[]`.

## Example: fal.ai (AI Image/Video/Music Generation)

[fal-mcp-server](https://github.com/raveenb/fal-mcp-server) provides 18 tools for AI generation.

### Prerequisites

Set `FAL_KEY` as a **global environment secret** (via UI or `transfer_secret`) before creating the sandbox. The key is auto-injected into sandboxes at creation time.

### Setup

```bash
# Install the fal MCP server
pip install fal-mcp-server

# List available tools (uses full path since pip installs to ~/.local/bin)
npx github:sanity-labs/mcpcli tools ~/.local/bin/fal-mcp
```

Tools include: `generate_image`, `generate_video`, `generate_music`, `edit_image`, `remove_background`, `upscale_image`, `list_models`, `get_pricing`, and more.

### Generate an image

```bash
npx github:sanity-labs/mcpcli call generate_image ~/.local/bin/fal-mcp -p '{
  "prompt": "A jazz musician playing piano in a smoky club, blue neon lighting",
  "image_size": "landscape_16_9"
}'
# Returns a URL to the generated image
```

### List available models

```bash
npx github:sanity-labs/mcpcli call list_models ~/.local/bin/fal-mcp -p '{"category": "image", "limit": 5}'
```

### Download and use the result

```bash
# Generate and capture JSON output
RESULT=$(npx github:sanity-labs/mcpcli call generate_image ~/.local/bin/fal-mcp \
  -p '{"prompt": "..."}' -f json)
URL=$(echo "$RESULT" | grep -oP 'https://[^\s\\]+\.jpg')

# Download to sandbox
curl -sL "$URL" -o /tmp/generated.jpg

# Transfer to board for sharing
# (use the sandbox transfer tool)
```

## Example: Building a Custom Stdio MCP Server

Use the MCP SDK (TypeScript or Python) to build your own:

### TypeScript (Node.js)

```bash
npm init -y
npm install @modelcontextprotocol/sdk
npm install -D typescript @types/node
```

```typescript
// src/index.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'my-tools', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{
    name: 'my_tool',
    description: 'Does something useful',
    inputSchema: {
      type: 'object',
      properties: { input: { type: 'string' } },
      required: ['input'],
    },
  }],
}));

server.setRequestHandler(CallToolRequestSchema, async (req) => {
  const { name, arguments: args } = req.params;
  return {
    content: [{ type: 'text', text: `Result: ${args?.input}` }],
  };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Important:** MCP SDK v1.26+ requires Zod schema objects (like `ListToolsRequestSchema`) for `setRequestHandler`, not plain strings like `'tools/list'`.

```bash
# Build and test
npx tsc
npx github:sanity-labs/mcpcli tools node dist/index.js
npx github:sanity-labs/mcpcli call my_tool node dist/index.js -p '{"input": "test"}'
```

### Python

```bash
pip install mcp
```

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def my_tool(input: str) -> str:
    """Description of what this tool does"""
    return f"Result: {input}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```bash
npx github:sanity-labs/mcpcli tools python server.py
npx github:sanity-labs/mcpcli call my_tool python server.py -p '{"input": "test"}'
```

## Environment & Secrets

Stdio MCP servers that need API keys (like `FAL_KEY`) get them from the sandbox environment:

1. **Configure the secret** in the channel/space environment (via UI or `transfer_secret`)
2. **Create a sandbox** — secrets are auto-injected as env vars at creation time
3. **The MCP server reads from env** — most servers use standard env var names (e.g., `FAL_KEY`, `OPENAI_API_KEY`)

`mcpcli` passes the full `process.env` to child processes, so all env vars are available to the server automatically.

> **Important:** Secrets added *after* sandbox creation won't appear. Create a new sandbox to pick up new secrets.

## Alternative: mcptools (Go)

If you prefer the Go-based [mcptools](https://github.com/f/mcptools), it can be installed via `go install`:

```bash
sudo apt-get update -qq && sudo apt-get install -y -qq golang-go
go install github.com/f/mcptools/cmd/mcptools@latest
sudo ln -sf ~/go/bin/mcptools /usr/local/bin/mcptools
```

Note: the npm package `mcptools` is a squatter (236 bytes, no binary). The Go install is the only real path. `mcptools` has more features (shell, guard, proxy, mock, web UI) but requires the Go toolchain.

## Tips

- **No permanent setup needed** — install, use, discard. Great for one-off tasks or trying new tools.
- **`npx` caches** — after the first run, `npx github:sanity-labs/mcpcli` is fast on subsequent calls.
- **Parallel tool calls** — run multiple `call` commands with `&` for concurrent operations.
- **Save results before cleanup** — transfer generated files to the board or commit to git before deleting the sandbox.
- **Any ecosystem MCP server works** — if it supports stdio transport, you can run it. Check [MCP server directories](https://github.com/modelcontextprotocol/servers) for options.
- **Pipe-friendly** — mcpcli auto-detects non-TTY and defaults to JSON output, so `mcpcli tools ... | jq .` just works.
