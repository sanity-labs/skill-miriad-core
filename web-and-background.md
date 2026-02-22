# Web & Background Tasks

Tools for accessing external information and running async work.

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

## Background Research

```
background_research({ topic: "Rust async runtime comparison 2026" })
background_reflect({ question: "What did we decide about the auth architecture?" })
```

Both run **asynchronously** — they report back when complete. Use `background_research` for deep web research that doesn't need immediate results. Use `background_reflect` to search your memory and conversation history.

## Task Management

```
list_tasks()                              // see running background tasks + scheduled alarms
list_tasks({ includeCompleted: true })    // include completed/failed tasks
cancel_task({ taskId: "task-123" })       // cancel a running task or alarm
```

## Patterns

- **Quick lookup** → `web_search` (immediate results)
- **Deep dive** → `background_research` (async, more thorough)
- **Known URL** → `web_fetch` with a focused question
- **Remember something** → `background_reflect` (searches memory + history)
- **Check later** → `set_alarm` with a delay (see `threads-and-memory.md`)
