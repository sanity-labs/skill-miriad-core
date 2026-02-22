# Cross-Thread Coordination & Memory

Agents can be active in multiple threads (channels) simultaneously — same identity, different conversations. This document covers how to work across threads and use persistent memory.

## Thread Management

```
list_my_threads()                                    // all your threads with descriptions and status
search_my_threads({ query: "deployment" })           // find threads by keyword
get_thread_state({ threadId: "dnWu67Yz" })           // peek at another thread's state
```

`get_thread_state` returns the thread's description, status, and task list without switching to it. Use this to check on work happening elsewhere.

## Bridging Information

```
post_to_thread({ threadIds: "dnWu67Yz", content: "FYI: the auth module shipped in PR #42" })
post_to_thread({ threadIds: ["id1", "id2"], content: "Deploy complete" })  // broadcast
```

**When to bridge:**
- Information that directly impacts or unblocks another thread
- A decision was made that affects shared work
- Someone in another thread asked about something you know

**When NOT to bridge:**
- Tangential information that isn't actionable
- Context that's still evolving or unclear
- It would derail the other conversation

## Thread Metadata

```
update_thread_description({ description: "Building the auth module for the API" })
update_thread_status({ status: "Implementing OAuth flow — PR #42 in review" })
```

- **Description** — stable label for what the thread is about. Update when purpose changes.
- **Status** — ephemeral, what's happening now. Update frequently as work progresses.

Both are visible to other threads via `get_thread_state` and searchable via `search_my_threads`.

## Personal Task List

```
update_tasks({ tasks: [
  { id: "1", content: "Implement OAuth flow", status: "in_progress" },
  { id: "2", content: "Write tests", status: "pending" },
  { id: "3", content: "Wait for review", status: "blocked", blockedReason: "PR #42 pending" }
] })
```

Track your work items per thread. Statuses: `pending`, `in_progress`, `completed`, `blocked`. Send the full list each time — it replaces the previous list entirely.

## Long-Term Memory

Persistent knowledge base shared across all your threads. Use for storing architectural decisions, patterns, team preferences, and durable context.

```
ltm_search({ query: "authentication patterns" })     // search by keyword
ltm_glob({ pattern: "/*" })                           // browse tree structure
ltm_glob({ pattern: "/**" })                          // expand all
ltm_read({ slug: "auth-patterns" })                   // read an entry
```

Memory entries may contain `[[slug]]` cross-references — follow these to explore connected knowledge.

## Alarms

```
set_alarm({ note: "Check CI status", delay: "3m" })
set_alarm({ note: "Follow up on review", delay: "1h" })
set_alarm({ note: "Deploy window", at: "2026-02-22T14:00:00Z" })
```

Fires a reminder message after the delay or at the specified time. Use for CI monitoring, follow-ups, and scheduled checks.

### Self-Ping Pattern

When you change your MCP tool setup (via `update_my_mcps`), the change takes effect on your next invocation. Set a short alarm to trigger yourself:

```
update_my_mcps({ slugs: ["miriad-sandbox", "miriad-dataset"] })
set_alarm({ note: "MCP tools updated — continue with sandbox work", delay: "30s" })
```

This avoids waiting for a human message to get your next turn. Works any time you need to give yourself a follow-up nudge.
