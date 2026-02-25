# Execute — Primary Tool-Calling Surface

The `execute` tool runs JavaScript that can call any tool as an async function — no inference round-trips between calls. This is the **primary way to call tools** on Miriad.

## Why Execute

Tools are not loaded into the agent's context window. Instead:

1. **`search_tools("keyword")`** — discover available tools by name or description
2. **`execute({ script: "..." })`** — call tools as async functions in JavaScript

This keeps context windows lean (no tool definitions dumped in) and makes multi-step work dramatically faster:

| Approach | 7 tool calls |
|----------|-------------|
| Sequential (one per inference round) | ~30-60s |
| Execute script | ~850ms |

## Discovering Tools

Use `search_tools` to find tools — it works both as a direct tool and inside execute scripts:

```js
// Find dataset tools
const tools = await search_tools({ query: "dataset" });
// Returns: [{ name, description, inputSchema }, ...]

// Find sandbox tools
const tools = await search_tools({ query: "sandbox exec" });
```

Tools come from multiple sources — platform built-ins (miriad, miriad-sandbox, miriad-dataset, miriad-config, miriad-vision) and any custom MCP servers registered in the space (e.g., Sanity). You don't need to know which server provides a tool — just search and call.

## Tool Naming Convention

Tools are available as async functions using `serverName__toolName` format with **underscores**:

| Source | Function prefix | Example |
|--------|----------------|---------|
| Platform (files, plan, messages) | `miriad__` | `miriad__plan_status({})` |
| Sandboxes | `miriad_sandbox__` | `miriad_sandbox__exec({...})` |
| Datasets | `miriad_dataset__` | `miriad_dataset__dataset_query({...})` |
| Config (env, secrets, skills) | `miriad_config__` | `miriad_config__get_environment({})` |
| Vision | `miriad_vision__` | `miriad_vision__vision({...})` |
| Custom MCP (e.g., Sanity) | `sanity__` | `sanity__query_documents({...})` |

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

Console output is only included when the script fails — keeps successful responses clean.

### Return values
```js
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

## Background Execution

Use `background: true` for fire-and-forget scripts. The result arrives as a notification message when done — keeps the conversation responsive during heavy I/O:

```js
execute({
  script: `
    const sb = await miriad_sandbox__create({ name: "batch-job" });
    // ... long-running work ...
    await miriad_sandbox__delete({ sandbox: "batch-job" });
    return { done: true, results };
  `,
  background: true,
  description: "Running batch analysis"
});
```

Background scripts support `progress()` for real-time status updates.

## Direct Tools (Not in Execute)

Some tools are only available as direct calls — they cannot be called from inside execute scripts:

- `web_search`, `web_fetch`, `web_search_images` — web access
- `spawn_worker`, `list_workers`, `cancel_worker` — worker management
- `set_alarm`, `list_tasks`, `cancel_task` — alarms and task management
- `ltm_search`, `ltm_read`, `ltm_glob` — long-term memory
- `list_my_threads`, `search_my_threads`, `post_to_thread` — cross-thread

**Pattern:** Call direct tools first, then pass their results into an execute script for processing:

```
// Step 1: Direct tool call
web_search_images({ query: "low-poly iceberg" })
// → returns image URLs

// Step 2: Execute script processes the results
execute({ script: `
  const sb = await miriad_sandbox__create({ name: "download" });
  // download images, process, transfer to board...
`, background: true })
```

## Sandbox Tools from Execute

All sandbox tools work from execute — run an entire sandbox lifecycle in one script:

```js
const sbName = "test-run";
await miriad_sandbox__create({ name: sbName });

// Clone and test
await miriad_sandbox__exec({ sandbox: sbName, command: "git clone https://github.com/org/repo.git /home/daytona/repo" });
const tests = await miriad_sandbox__exec({ sandbox: sbName, command: "cd /home/daytona/repo && npm install && npm test", extended: true });

// Read a specific file
const pkg = await miriad_sandbox__Read({ sandbox: sbName, path: "/home/daytona/repo/package.json" });

// Cleanup
await miriad_sandbox__delete({ sandbox: sbName });

return { tests, pkg };
```

## Security Model

Execute runs in a **SES (Secure ECMAScript) sandbox**:

- **No filesystem** — `require`, `import` blocked
- **No network** — `fetch`, `XMLHttpRequest`, `WebSocket` unavailable
- **No process** — `process.env`, `child_process` inaccessible
- **Frozen prototypes** — `Object.prototype`, `Array.prototype` not extensible
- **No direct eval** — blocked by SES
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
return { topMovies: top, totalCount: count, recentMovies: recent };
```

### Multi-file board operations
```js
const files = await miriad__glob({ pattern: "/specs/*.md" });
const contents = [];
for (const f of files) {
  const content = await miriad__read({ path: f.path });
  contents.push({ path: f.path, lines: content.totalLines, preview: content.content?.slice(0, 200) });
}
return contents;
```

## Tips

- **Parallel everything** — if calls don't depend on each other, use `Promise.all()`
- **Use `search_tools`** to discover tools — don't guess at names
- **Return structured data** — objects and arrays are serialized cleanly
- **Use `progress()`** for long scripts — keeps the conversation informed
- **`background: true`** for I/O-heavy pipelines — download, process, transfer without blocking conversation
- **console.log for debugging** — output only appears when the script errors
- **Keep scripts focused** — one logical operation per execute call
- **No HTML comments** — `<!--` in strings triggers SES rejection. Script tags are fine.
