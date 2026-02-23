# Stdio MCP Servers in Sandboxes

Agents can run **any stdio MCP server** from a sandbox using `mcptools` — no permanent MCP configuration needed. Install the server, call tools on demand, tear down when done.

This is more flexible than mounted MCP servers: no OAuth, no slug registration, no persistent connections. Just `pip install` or `npm install` a server and start calling tools.

## Install `mcptools`

[mcptools](https://github.com/f/mcptools) is a Go CLI for interacting with stdio MCP servers. The only install method is `go install` (no pre-built binaries, npm `mcptools` is a squatter package).

```bash
# Fix broken yarn repo (Daytona image)
sudo rm -f /etc/apt/sources.list.d/yarn.list

# Install Go + mcptools + symlink into PATH (~30s total)
sudo apt-get update -qq && sudo apt-get install -y -qq golang-go
go install github.com/f/mcptools/cmd/mcptools@latest
sudo ln -sf ~/go/bin/mcptools /usr/local/bin/mcptools
```

The `ln -sf` puts it in `/usr/local/bin` so it's always in PATH. Go auto-downloads a newer toolchain if needed (transparent).

## Using `mcptools`

```bash
# List all tools a server provides
mcptools tools <server-command>

# Call a specific tool
mcptools call <tool-name> <server-command> -p '<json-params>'

# Output formats
mcptools tools <server-command> -f json     # JSON output
mcptools call <tool> <server> -p \'{}\' -f pretty  # pretty-printed
```

## Example: fal.ai (AI Image/Video/Music Generation)

[fal-mcp-server](https://github.com/raveenb/fal-mcp-server) provides 18 tools for AI generation.

### Prerequisites

Set `FAL_KEY` as a **global environment secret** (via UI or `transfer_secret`) before creating the sandbox. The key is auto-injected into sandboxes at creation time.

### Setup

```bash
# Install the fal MCP server
pip install fal-mcp-server

# List available tools
mcptools tools fal-mcp
```

Tools include: `generate_image`, `generate_video`, `generate_music`, `edit_image`, `remove_background`, `upscale_image`, `list_models`, `get_pricing`, and more.

### Generate an image

```bash
mcptools call generate_image fal-mcp -p \'{"prompt": "A jazz musician playing piano in a smoky club", "image_size": "landscape_16_9"}\'
# Returns a URL to the generated image
```

### Download and use the result

```bash
# Generate
RESULT=$(mcptools call generate_image fal-mcp -p \'{"prompt": "..."}\'  -f json)
URL=$(echo "$RESULT" | grep -oP \'https://[^\\s\\\\]+\\.jpg\')

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
npm install @modelcontextprotocol/sdk sharp
npm install -D typescript @types/node
```

```typescript
// src/index.ts
#!/usr/bin/env node
import { McpServer } from \'@modelcontextprotocol/sdk/server/mcp.js\';
import { StdioServerTransport } from \'@modelcontextprotocol/sdk/server/stdio.js\';
import { z } from \'zod\';

const server = new McpServer({ name: \'my-tools\', version: \'1.0.0\' });

server.tool(
  \'my_tool\',
  \'Description of what this tool does\',
  { input: z.string().describe(\'Input parameter\') },
  async ({ input }) => {
    try {
      const result = { /* ... */ };
      return { content: [{ type: \'text\', text: JSON.stringify(result) }] };
    } catch (error: unknown) {
      const msg = error instanceof Error ? error.message : String(error);
      return { content: [{ type: \'text\', text: msg }], isError: true };
    }
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

```bash
# Build and test
npx tsc
mcptools tools node dist/index.js
mcptools call my_tool node dist/index.js -p \'{"input": "test"}\'
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
mcptools tools python server.py
mcptools call my_tool python server.py -p \'{"input": "test"}\'
```

## Environment & Secrets

Stdio MCP servers that need API keys (like `FAL_KEY`) get them from the sandbox environment:

1. **Configure the secret** in the channel/space environment (via UI or `transfer_secret`)
2. **Create a sandbox** — secrets are auto-injected as env vars at creation time
3. **The MCP server reads from env** — most servers use standard env var names (e.g., `FAL_KEY`, `OPENAI_API_KEY`)

> **Important:** Secrets added *after* sandbox creation won\'t appear. Create a new sandbox to pick up new secrets.

## Tips

- **No permanent setup needed** — install, use, discard. Great for one-off tasks or trying new tools.
- **Parallel tool calls** — run multiple `mcptools call` commands with `&` for concurrent operations.
- **Save results before cleanup** — transfer generated files to the board or commit to git before deleting the sandbox.
- **Any ecosystem MCP server works** — if it supports stdio transport, you can run it. Check [MCP server directories](https://github.com/modelcontextprotocol/servers) for options.
- **`mcptools` binary name** — on Linux it\'s `mcptools` (not `mcp`). The `mcp` command may be the official MCP CLI (different tool).
