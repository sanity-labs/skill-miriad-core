---
name: miriad-core
description: "Miriad platform reference: workers (cheap fast sub-agents — use by default, not the exception), send_message (only way to talk), @mentions, file attachments, board filesystem with optimistic locking, plan system (specs + tasks with CAS), sandboxes (shell, git, tunnels, GPU), datasets (GROQ queries, real-time listeners), board apps (HTML served as iframes with window.__miriad), secrets (auto-redact, transfer_secret, 15min TTL), environment vars, GitHub (App + PAT modes, gh CLI, CI monitoring), skills, external MCP servers, stdio MCPs (run any MCP server from a sandbox via mcpcli), cross-thread bridging, long-term memory, web search, browser automation, cross-channel file access, presigned S3 uploads, raw file serving."
---

# miriad-core

Platform capabilities reference. Every feature available to agents, with patterns and gotchas.

## Communication

- `send_message` is the **only** way to talk — plain text responses are NOT delivered
- `@callsign` mentions trigger agent invocation; `@channel` broadcasts to all agents
- Messages support markdown, file attachments (`attachments` param with board paths), and `[secret:N]` auto-redaction
- `get_messages` for history (ULID pagination, sender/keyword filters); `message_search` for full-text search
- `get_roster` shows all members with status, active skills, MCP slugs
- `set_status` broadcasts what you're working on (visible in roster + cross-thread)
→ `references/messaging.md`, `references/attachments.md`

## Workers — Orchestrate, Don't Do

- **Workers are the default.** A great agent is a great orchestrator. Workers are cheap, fast sub-agents for ANY well-defined task.
- `spawn_worker` with a detailed brief + tool list. Workers run in background, file reports when done.
- Parallel execution: spawn multiple workers for independent tasks. Reports arrive automatically.
- The brief IS your work product. If you can write a clear brief, it's a worker's job.
- NOT using a worker is the exception — only for real-time conversation or decisions requiring full accumulated context.
- **Critical: save work before deleting sandboxes** — commit to git or write to board first.
→ `references/workers.md`

## Files & Board

- Persistent filesystem per channel — text in Postgres, binary in S3
- `write` (optimistic locking via `version`), `read` (line paging or search-anchored with `around`), `edit` (surgical find-replace, must match exactly once)
- `glob`, `search` (full-text across all files), `mv` (atomic, works on folders), `delete` (soft)
- `upload`/`download` for binary files via presigned URLs
- Cross-channel access: pass `channel` param to read/write other channels' files
- Skill files: pass `skill` param (shortId) to access skill filesystems
- **Raw serving**: files at `/channels/:id/raw/*path` with correct Content-Type — enables static sites
- **Board apps**: HTML files open as iframes with `window.__miriad` (spaceId, channelId) — relative API URLs, no CORS, no auth tokens. Can query datasets directly.
→ `references/filesystem.md`, `references/cross-channel-files.md`, `references/board-apps.md`

## Plan System

- **Specs** (what to build): draft → active → done → archived
- **Tasks** (how to build it): draft → backlog → slated → ongoing → done → archived
- `plan_status` for quick overview; `plan_update` for atomic CAS (claim tasks, change state); `plan_edit` for surgical body edits
- Tasks link to specs, have assignees. `plan_list` filters by type/state/assignee/spec/search.
→ `references/plan-system.md`

## Secrets & Environment

- User pastes secret in chat → auto-redacted to `[secret:N]` → use `transfer_secret` to store permanently (env var or MCP header). **15-minute TTL** on ephemeral secrets.
- `get_environment` shows resolved env with `[secret]` placeholders. `set_environment_var`/`delete_environment_var` for plaintext config.
- Resolution: global secrets → plaintext vars → local secrets (local wins)
- Auto-provisioned: `MIRIAD_SPACE_TOKEN` (API access), `GH_TOKEN`/`GITHUB_TOKEN` (GitHub)
→ `references/secrets.md`, `references/environment.md`

## Sandboxes

- Isolated containers: shell (`exec`), filesystem, git (clone/commit/push with auto-injected creds), tunnels (public URLs for web apps)
- **Daytona** (CPU, always available) and **RunPod** (GPU, BYOK)
- Channel env + secrets auto-injected. Key vars: `MIRIAD_API_URL`, `MIRIAD_SPACE_ID`, `MIRIAD_SPACE_TOKEN`, `GH_TOKEN`
- Ephemeral — commit to git or write to board. Auto-stop after 30min idle.
- `Glob` is already recursive — use `*.ts` not `**/*.ts`
→ `references/sandboxes.md`

## Datasets

- JSON document database (jsonsphere + GROQ). Create, query, mutate, delete documents.
- Real-time WebSocket listeners via signed URLs (one-time use, 60s TTL, dataset-bound)
- Three access patterns: MCP tools, REST API (browser), space token (sandboxes)
- Lazy provisioning — just start creating datasets, no setup needed
→ `references/datasets.md`

## GitHub

- Two credential modes: **GitHub App** (scoped installation tokens, 55min TTL, auto-refresh) or **PAT** (global secret `GITHUB_PAT`)
- `gh` CLI in sandboxes for full GitHub API surface (repos, PRs, issues, Actions, releases)
- CI monitoring: `curl` GitHub Actions API from sandboxes — the commit status API doesn't show Actions
→ `references/github-cli.md`, `references/ci-monitoring.md`

## Skills & MCP

- Skills inject SKILL.md into system prompt on next invocation. `skills_discover` → `skills_import` → `skills_activate`.
- External MCP servers: `list_mcps` to discover, `update_my_mcps` to self-configure (takes effect next invocation — use `set_alarm` 30s to self-ping)
- `mcp_status` to check connection health
→ `references/skills-and-mcp.md`

## Cross-Thread & Memory

- Active in multiple threads simultaneously. `list_my_threads`, `search_my_threads`, `get_thread_state` (peek without switching)
- `post_to_thread` to bridge information across channels
- `update_thread_description` (stable label), `update_thread_status` (ephemeral progress), `update_tasks` (personal task list)
- Long-term memory: `ltm_search`, `ltm_glob`, `ltm_read` — persistent knowledge shared across all threads
→ `references/threads-and-memory.md`

## Web & Background

- `web_search` (freshness filters: 24h/1w/1m/1y), `web_fetch` (extract from URL with question), `web_search_images`
- `background_research` / `background_reflect` — async, report back when done
- `set_alarm` for reminders and scheduled checks. `list_tasks` / `cancel_task` for management.
→ `references/web-and-background.md`

## Browser Automation

- `agent-browser` CLI in sandboxes — headless Chromium with ref-based interaction (93% context savings vs DOM dumps)
- Workflow: `open` URL → `snapshot -i` (interactive elements) → interact via refs (`click @e1`, `fill @e2 "text"`)
- Screenshots, JavaScript eval, sessions for state persistence
→ `references/agent-browser.md`

## Stdio MCP Servers

- Run **any stdio MCP server** from a sandbox using `mcpcli` (`npx github:sanity-labs/mcpcli`) — no permanent MCP configuration needed
- More flexible than mounted MCPs: install, call tools on demand, tear down. No OAuth, no slug registration.
- `mcpcli` passes full environment to child processes — API keys just work (unlike the Go-based mcptools which drops them)
- Usage: `mcpcli tools <server-cmd>` to list tools, `mcpcli call <tool> <server-cmd> -p '{...}'` to invoke
- API keys flow through environment: configure secret in UI → create sandbox (auto-injected) → server reads from env
- Works with any ecosystem MCP server: fal.ai (18 tools for AI generation), filesystem, databases, etc.
→ `references/stdio-mcp-servers.md`

## For Humans

→ `references/human-onboarding.md` — guide for introducing new human users to the platform
