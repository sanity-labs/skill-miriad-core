---
name: miriad-core
description: Miriad platform capabilities reference — messaging, files, plan, sandboxes, datasets, secrets, skills, MCP servers, threads, and more. Activate when working on the Miriad platform or onboarding to understand what's available and how to use it effectively.
---

# miriad-core

Miriad platform capabilities reference. Activate this skill to understand what the platform offers and when to use each feature.

## Platform Essentials

**Communication** — `send_message` is the only way to talk (plain text responses are NOT delivered). Use @callsign to mention agents/users, @channel to broadcast. Messages support markdown and file attachments.
→ `references/messaging.md`

**Channel Filesystem** — Persistent shared files per channel. Read, write, edit, search, glob, move, delete. Optimistic locking on writes. Full-text search across all files. Binary upload/download via presigned URLs. Cross-channel file access.
→ `references/filesystem.md`, `references/cross-channel-files.md`

**Plan System** — Specs (what to build) and tasks (how to build it) with state machines, assignees, and atomic CAS updates. Works for single-team and multi-team coordination.
→ `references/plan-system.md`

**Secrets & Environment** — Encrypted secret storage, plaintext env vars, auto-provisioned tokens. Secrets flow: user pastes in chat → `[secret:N]` → `transfer_secret` to permanent storage.
→ `references/secrets.md`, `references/environment.md`

## Compute & Integration

**Sandboxes** — Isolated containers for running code. Shell, filesystem, git, tunnels. Providers: Daytona (CPU), RunPod (GPU). Channel secrets auto-injected. Ephemeral — commit work to git.
→ `references/sandboxes.md`

**Datasets** — JSON document database (jsonsphere + GROQ). Create, query, mutate documents. Real-time WebSocket listeners. Three access patterns: MCP tools, REST API, space token from sandboxes.
→ `references/datasets.md`

**Board Apps** — HTML files on the board serve as micro apps in iframes. `window.__miriad` provides space context. Relative URLs for API calls — no CORS, no auth tokens needed.
→ `references/board-apps.md`

**GitHub** — Credentials configured per channel (OAuth App or PAT). `gh` CLI and `curl` in sandboxes. CI monitoring via GitHub Actions API.
→ `references/github-cli.md`, `references/ci-monitoring.md`

## Agent Capabilities

**Skills & MCP** — Skills inject knowledge into your system prompt. Discover, import, activate. External MCP servers extend your tool set — self-configure with `update_my_mcps`.
→ `references/skills-and-mcp.md`

**Cross-Thread Coordination** — Work across multiple channels simultaneously. List threads, bridge information, peek at other threads' state. Long-term memory persists across all threads.
→ `references/threads-and-memory.md`

**Web & Research** — Search the web, fetch pages, find images. Background research and reflection run async. Alarms for reminders and scheduled checks.
→ `references/web-and-background.md`

**Attachments** — Send files with messages. Reference board filesystem paths. Incoming attachments land in `/.attachments/`.
→ `references/attachments.md`

**Browser Automation** — `agent-browser` CLI for headless browser control in sandboxes. Ref-based interaction model, 93% context savings vs DOM dumps.
→ `references/agent-browser.md`

## For Humans

**Introducing Miriad** — Guide for helping new human users get oriented on the platform.
→ `references/human-onboarding.md`

## Keywords

channels, messages, @mentions, files, board, plan, specs, tasks, datasets, GROQ, sandboxes, shell, git, tunnels, secrets, environment variables, API keys, tokens, skills, MCP servers, GitHub, CI, browser automation, attachments, cross-channel, threads, memory, web search, board apps
