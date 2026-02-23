# Web & Worker Sub-Agents

Tools for accessing external information and spawning async worker threads.

## Web Search

```
web_search({ query: "react server components tutorial" })
web_search({ query: "node.js 22 release notes", freshness: "1m" })    // past month
web_search({ query: "CVE-2026-1234", freshness: "1w" })               // past week
web_search({ query: "breaking news", freshness: "24h", maxResults: 5 })
```

Freshness filters: `24h` (past day), `1w` (past week), `1m` (past month), `1y` (past year).

## Fetching Pages

```
web_fetch({
  url: "https://react.dev/reference/react/useState",
  question: "What are the rules for using useState?"
})
```

Fetches a page and extracts information focused on your question. More targeted than raw search — use when you know the URL.

## Image Search

```
web_search_images({ query: "microservices architecture diagram" })
web_search_images({ query: "react component lifecycle", maxResults: 3 })
```

Returns markdown with inline thumbnails and links to full-resolution originals.

## Worker Sub-Agents

Workers are task threads that run asynchronously with full MCP tool access and Chorus callbacks. They use the `claude-sonnet-4-6` workhorse model.

```
spawn_worker({ task: "Research Rust async runtime options and summarize tradeoffs" })
spawn_worker({ task: "Review the PR diff for security issues", tools: ["web_fetch"] })
```

Workers inherit the parent agent's MCP servers and can call any tool the parent can. Results are reported back when the worker completes.

```
list_workers()                            // check status of running workers
cancel_worker({ workerId: "w-123" })      // cancel a running worker
```

| Tool | Description |
|------|-------------|
| `spawn_worker` | Spawn an async worker thread for a task (research, review, data processing, etc.) |
| `list_workers` | Check status of running workers |
| `cancel_worker` | Cancel a running worker |

## Task Management

```
list_tasks()                              // see scheduled alarms
list_tasks({ includeCompleted: true })    // include completed/failed
cancel_task({ taskId: "task-123" })       // cancel a scheduled alarm
```

## Patterns

- **Quick lookup** → `web_search` (immediate results)
- **Deep dive** → `spawn_worker` (async, full tool access, thorough)
- **Known URL** → `web_fetch` with a focused question
- **Parallel work** → `spawn_worker` multiple tasks, check with `list_workers`
- **Check later** → `set_alarm` with a delay (see `threads-and-memory.md`)
