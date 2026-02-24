# Execute — JavaScript Tool Chaining

The `execute` tool runs JavaScript that can call any MCP tool as an async function — no inference round-trips between calls. This is dramatically faster for multi-step workflows.

## Why Use Execute

| Approach | 7 tool calls |
|----------|-------------|
| Sequential (normal) | ~30-60s (7 inference rounds) |
| Execute script | ~850ms (1 inference round) |

Use `execute` when you need to:
- **Query multiple sources in parallel** — dashboard-style data gathering
- **Batch operations** — create/mutate multiple documents in one shot
- **Read → transform → write** — process data without round-trips
- **Conditional logic** — branch based on tool results without inference overhead

## Tool Naming Convention

Tools are available as async functions using `serverName__toolName` format with **underscores** (not hyphens):

| MCP Server | Function prefix | Example |
|------------|----------------|---------|
| miriad | `miriad__` | `miriad__plan_status({})` |
| miriad-sandbox | `miriad_sandbox__` | `miriad_sandbox__list({})` |
| miriad-dataset | `miriad_dataset__` | `miriad_dataset__dataset_query({...})` |
| miriad-config | `miriad_config__` | `miriad_config__get_environment({})` |
| miriad-vision | `miriad_vision__` | `miriad_vision__vision({...})` |
| sanity | `sanity__` | `sanity__query_documents({...})` |

**Rule:** Hyphens in server names become underscores. The `__` double-underscore separates server from tool.

## Core API

### Calling tools
```js
// All tools are async — always use await
const result = await miriad__plan_status({});
const files = await miriad__glob({ pattern: "*.md" });
```

### Parallel calls with Promise.all
```js
const [sandboxes, datasets, plan] = await Promise.all([
  miriad_sandbox__list({}),
  miriad_dataset__dataset_list({}),
  miriad__plan_status({})
]);
```

### Progress updates
```js
progress("Loading data...");    // visible in conversation in real-time
progress("Processing results...");
```

### Debugging
```js
console.log("debug info:", someVar);  // appears in execution log on ERROR only
```

Console output is only included when the script fails — keeps successful responses clean. Use it for debugging, not for output.

### Return values
```js
// Return a value to send it back as the result
return { count: results.length, items: results };
```

### Error handling
```js
try {
  const result = await miriad_sandbox__exec({ sandbox: "s", command: "npm test" });
  return result;
} catch (e) {
  console.log("Failed:", e.message);
  return { error: e.message };
}
```

## Sandbox Tools from Execute

All sandbox tools work from execute — you can run an entire sandbox lifecycle in one script:

```js
// Create sandbox, run tests, report, cleanup — zero inference round-trips
const sb = await miriad_sandbox__create({ name: "test-run" });
const sbName = "test-run";

// Clone and test
await miriad_sandbox__exec({ sandbox: sbName, command: "git clone https://github.com/org/repo.git /home/daytona/repo" });
const tests = await miriad_sandbox__exec({ sandbox: sbName, command: "cd /home/daytona/repo && npm install && npm test", extended: true });

// Read a specific file
const pkg = await miriad_sandbox__Read({ sandbox: sbName, path: "/home/daytona/repo/package.json" });

// Cleanup
await miriad_sandbox__delete({ sandbox: sbName });

return { tests, pkg };
```

Sandbox tool naming follows the same convention: `miriad_sandbox__exec`, `miriad_sandbox__Read`, `miriad_sandbox__Write`, etc.

## Discovering Tools

Use `searchTools()` to find available tools by keyword:

```js
const tools = await searchTools("dataset");
// Returns: [{ name, description, serverName, inputSchema }, ...]
```

This is useful when you're not sure of the exact tool name or want to explore what's available.

## Security Model

Execute runs in a **SES (Secure ECMAScript) sandbox**:

- **No filesystem** — `require`, `import` blocked
- **No network** — `fetch`, `XMLHttpRequest`, `WebSocket` unavailable
- **No process** — `process.env`, `child_process` inaccessible
- **Frozen prototypes** — `Object.prototype`, `Array.prototype` not extensible
- **No direct eval** — blocked by SES (indirect via `Function` constructor works but can't escape)
- **Fresh per call** — globalThis is recreated each execution, no state persists between calls
- **Scoped tools** — only tools granted to the agent/worker appear on globalThis

`Math.random()` and `Date` work normally (real randomness, real wall clock).

## Practical Examples

### Gather platform state in one shot
```js
const [sandboxes, datasets, plan, roster] = await Promise.all([
  miriad_sandbox__list({}),
  miriad_dataset__dataset_list({}),
  miriad__plan_status({}),
  miriad__get_roster({})
]);
return { sandboxes, datasets, plan, roster };
```

### Batch dataset queries
```js
const [top, count, recent] = await Promise.all([
  miriad_dataset__dataset_query({
    dataset: "movies",
    query: '*[_type == "movie"] | order(popularity desc) [0..4] { title, popularity }'
  }),
  miriad_dataset__dataset_query({
    dataset: "movies",
    query: 'count(*[_type == "movie"])'
  }),
  miriad_dataset__dataset_query({
    dataset: "movies",
    query: '*[_type == "movie"] | order(releaseDate desc) [0..2] { title, releaseDate }'
  })
]);
return { topMovies: top.result, totalCount: count.result, recentMovies: recent.result };
```

### Read multiple files and combine
```js
const files = await miriad__glob({ pattern: "/specs/*.md" });
const contents = [];
for (const f of files) {
  const content = await miriad__read({ path: f.path });
  contents.push({ path: f.path, lines: content.totalLines, preview: content.content.slice(0, 200) });
}
return contents;
```

### Discover available tools
```js
// searchTools finds tools by keyword
const tools = await searchTools("dataset");
return tools;
```

## Tips

- **Parallel everything** — if calls don't depend on each other, use `Promise.all()`
- **Return structured data** — objects and arrays are serialized cleanly
- **Use progress()** for long scripts — keeps the conversation informed
- **console.log for debugging** — output only appears when the script errors, not on success. Use `progress()` for status updates, `return` for output.
- **Keep scripts focused** — one logical operation per execute call. Don't try to build an entire app in one script.
