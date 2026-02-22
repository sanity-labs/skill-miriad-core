# The Plan System

Miriad channels have a built-in plan for tracking work — specs (what to build) and tasks (how to build it). The plan is channel-scoped: each channel has its own plan, and tools operate on the current channel's plan.

## Core Concepts

### Specs
Specs describe **what** to build. They capture requirements, design decisions, and context. A spec might cover a feature, a system change, or a design direction.

### Tasks
Tasks describe **how** to build it. They're concrete, assignable work items. Tasks can optionally link to a parent spec via the `spec` field.

### States

**Spec states:**
| State | Meaning |
|-------|---------|
| `draft` | Idea captured, needs design/validation before anyone builds |
| `active` | Verified and ready to build — design reviewed, scope clear |
| `done` | Shipped and verified |
| `archived` | No longer relevant, or already done when captured |

**Task states:**
| State | Meaning |
|-------|---------|
| `draft` | Captured but not ready for work |
| `backlog` | Ready but not prioritized |
| `slated` | Prioritized and ready to pick up |
| `ongoing` | Someone is actively working on it |
| `done` | Completed |
| `archived` | No longer relevant |

## Tools

### Quick Overview
```
plan_status()                    — Glance at active specs and non-backlog tasks
plan_list({ type, state, ... })  — Browse with filters
plan_read("shortid")             — Full details on one item
```

### Creating Items
```
plan_create({
  type: "spec",
  title: "Message attachments",
  body: "## Summary\nAllow users to attach files to messages...",
  state: "draft"
})

plan_create({
  type: "task",
  title: "Add attachment field to send_message",
  body: "Modify the send_message MCP tool to accept an attachments array...",
  spec: "Ab12CdEf",     // link to parent spec
  assignee: "harmon",    // who's doing it
  state: "slated"
})
```

### Updating Items
Use `plan_update` for atomic field changes — it uses compare-and-swap to prevent conflicts:

```
// Claim a task
plan_update("taskId", {
  before: { state: "slated", assignee: null },
  after:  { state: "ongoing", assignee: "harmon" }
})

// Mark done
plan_update("taskId", {
  before: { state: "ongoing" },
  after:  { state: "done" }
})
```

### Editing Content
Use `plan_edit` for surgical find-replace on the body (same as file `edit`):

```
plan_edit({
  shortid: "Ab12CdEf",
  old_string: "## Status\nIn progress",
  new_string: "## Status\nShipped in PR #42"
})
```

### Full Overwrite
Use `plan_write` when you need to replace the entire body. Requires the current `version` (from `plan_read`) for optimistic locking:

```
plan_write({
  shortid: "Ab12CdEf",
  title: "Updated title",
  state: "active",
  body: "Completely new body content...",
  version: 3
})
```

### Reordering
```
plan_move({ shortid: "taskA", above: "taskB" })
plan_move({ shortid: "taskA", below: "taskB" })
```

## Single-Team Usage

In a single channel with one or two builders, the plan is straightforward:

### Pattern: Flat Task List
For small teams doing quick work, you may not need specs at all. Just create tasks:

```
plan_create({ type: "task", title: "Fix mobile layout bug", state: "slated", assignee: "harmon" })
plan_create({ type: "task", title: "Add dark mode toggle", state: "slated", assignee: "bart" })
```

Use `plan_status()` to see what's active. Builders claim tasks by updating `assignee` and moving to `ongoing`. Mark `done` when shipped.

### Pattern: Spec + Tasks for Features
For anything non-trivial, create a spec first, then link tasks to it:

```
// 1. Create the spec
plan_create({
  type: "spec",
  title: "User onboarding flow",
  body: "## Summary\nNew users need a guided setup...\n\n## Design\n...",
  state: "active"
})
// Returns: { shortid: "xY9zAbCd" }

// 2. Create tasks linked to the spec
plan_create({
  type: "task",
  title: "Build welcome screen component",
  spec: "xY9zAbCd",
  assignee: "bart",
  state: "slated"
})
plan_create({
  type: "task",
  title: "Add onboarding API endpoint",
  spec: "xY9zAbCd",
  assignee: "harmon",
  state: "slated"
})
```

### Task Flow
1. **Create** tasks as `slated` (ready to pick up)
2. Builder **claims** → `ongoing` with their name as assignee
3. Builder **ships** → `done`
4. When all tasks are done, mark the spec `done`

## Multi-Team Usage

When work spans multiple channels (a coordinator channel + execution channels), the plan system works differently in each:

### The Coordinator Channel (HQ)
The coordinator maintains the **canonical backlog** — all specs and high-level tasks live here. Specs go through a lifecycle:

```
draft → (design review) → active → (execution) → done → archived
```

**Key rules:**
- Brain dump items start as `draft` — they need validation before anyone builds
- Only move to `active` after design review or explicit approval
- Never send `draft` specs to execution channels

### Execution Channels
Each execution channel has its **own local plan** with implementation-level tasks. The coordinator bridges work by sending task descriptions (via messages), and the local aspect creates plan items:

```
// In the execution channel:
plan_create({
  type: "task",
  title: "Implement attachment upload — HQ ref: 9ar3Az84",
  body: "From HQ spec...\n\nKey files:\n- apps/api/src/routes/messages.ts\n...",
  assignee: "harmon",
  state: "slated"
})
```

### Bridging Pattern
The coordinator can't create plan items in another channel's plan directly (plans are channel-scoped). Instead:

1. **Coordinator sends a message** to the execution channel describing the work — task IDs, key files, implementation notes
2. **The coordinator's aspect** in that channel creates local plan items and assigns builders
3. **Builders execute** using their local plan
4. **Execution channel reports back** when done — the coordinator archives the HQ task

### Splitting Work Across Channels
- Assign each task to **one channel only** — never broadcast the same task to multiple channels (causes duplicate work)
- Split by theme, complexity, or to balance load
- Track which channel has which tasks
- When a channel's queue clears, proactively feed the next batch

### Cross-Channel Visibility
- Use `plan_status()` in each channel to see that channel's state
- The coordinator tracks the big picture across all channels
- Execution channels don't need to see HQ's full backlog — they get focused briefs

## Writing Good Specs

A well-structured spec helps builders move fast without guessing:

```markdown
## Summary
What this is and why we're building it. One paragraph.

## Design
The technical approach. Subsections for each major component.
Include API shapes, data models, and key decisions.

## Key Files
- `apps/api/src/routes/messages.ts` — message posting endpoint
- `apps/console/src/components/MessageInput.tsx` — input component

## Technical Notes
Implementation considerations, edge cases, dependencies.

## Related
- Spec Ab12CdEf (parent feature)
- Task xY9zAbCd (prerequisite)
```

## Writing Good Tasks

Tasks should be actionable without needing to ask questions:

```markdown
**What:** Add `attachments` field to the `send_message` MCP tool

**Key files:**
- `apps/api/src/mcp/tools/messages.ts` — tool definition
- `apps/api/src/lib/operations/messages.ts` — postMessage operation

**Acceptance:**
- send_message accepts optional `attachments` array
- Each attachment has `{ path, preview? }` shape
- Attachments are stored in message metadata
- Existing messages without attachments still work

**HQ ref:** 9ar3Az84
```

## Tips

- **`plan_status()` is your friend** — call it to orient yourself when starting work or checking progress
- **Use `plan_list({ search: "keyword" })` to find items** — full-text search across titles and bodies
- **Link tasks to specs** — makes it easy to see all work for a feature via `plan_list({ spec: "specId" })`
- **Archive liberally** — done and irrelevant items should be archived to keep the active view clean
- **Specs are living documents** — use `plan_edit` to update them as designs evolve, don't create new specs for revisions
- **The plan is not a project management tool** — it's a lightweight coordination mechanism. Keep items focused and actionable.

