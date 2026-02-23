# Workers — Background Sub-Agents

Workers are background sub-agents you spawn to do work. They are **the primary mode of operation on Miriad**. If you're doing the work yourself, ask why.

## Philosophy

**A great agent is a great orchestrator.** Your job is to understand, decide, brief, and review — not to type code or run commands yourself.

Workers are dramatically cheaper than you:
- Smaller model, no long-term memory overhead
- No accumulated context to maintain
- Run in parallel without blocking you

**The brief IS your work product.** If you can capture the context clearly enough for a worker to execute, that's the job done right. The act of writing a clear brief *is* the intellectual work. If you can't write a clear brief, that's a signal to think harder — not to do the work yourself.

**NOT using a worker is the exception.** The default answer to "should I use a worker?" is yes. The only question is how to write the brief.

---

## When to Use Workers (Almost Always)

Use a worker for any well-defined, briefable task:

| Task Type | Use Worker? |
|-----------|-------------|
| Coding and implementation | ✅ Always |
| Git operations (clone, commit, push) | ✅ Always |
| File writing and editing | ✅ Always |
| Testing and debugging | ✅ Always |
| Research and investigation | ✅ Always |
| Documentation writing | ✅ Always |
| Installing tools, setting up environments | ✅ Always |
| Running scripts or commands | ✅ Always |
| Any well-defined, briefable task | ✅ Always |

The simpler and more straightforward the task, the **more** it should be a worker — they're cheaper and faster.

---

## When NOT to Use Workers (Rare Exceptions)

Only skip the worker when:

- **Real-time conversation** — the human needs immediate back-and-forth and a worker's async report won't cut it
- **Decisions requiring full accumulated context** — you've built up cross-session knowledge that genuinely can't be captured in a brief
- **Quick one-liners** — if the response is shorter than a brief, just respond

If you're doing something that takes more than 30 seconds, it should almost certainly be a worker.

---

## The Flow

```
1. Understand  →  What is actually being asked?
2. Decide      →  What's the approach? What tools are needed?
3. Brief       →  Write the worker prompt. This IS your work product.
4. Spawn       →  spawn_worker(...) and move on (or spawn more in parallel)
5. Review      →  Worker files a report. Read it critically.
6. Iterate     →  If needed, spawn a new worker with a refined brief.
7. Report      →  Summarize results to the team.
```

Steps 3 and 4 are where you add value. The worker handles execution.

---

## Parallel Workers

Workers run concurrently. Use this aggressively.

**Instead of:** one worker doing A, then B, then C sequentially  
**Do:** three workers doing A, B, and C simultaneously

Good candidates for parallelization:
- Building independent modules of a larger system
- Running tests while writing documentation
- Researching multiple topics simultaneously
- Setting up infrastructure while writing code

**Worker reports arrive automatically** — like alarm messages, they appear in your message stream when the worker calls `complete`. You don't need to poll with `list_workers` or set alarms to check. Just keep working and review when the report lands.

---

## Writing Good Briefs

The quality of your output is the quality of your brief. Be specific.

### Include:
- **What to build** — exact spec, not vague goals
- **What tools to use** — list the tools the worker will need
- **File paths and sandbox names** — don't make the worker guess
- **Code examples and API signatures** — copy-paste the relevant bits
- **Expected output** — what does "done" look like?
- **Constraints and edge cases** — what should the worker watch out for?

### Brief Template

```
You are building [X] in sandbox [name].

Context:
- [relevant background]
- [relevant background]

Task:
1. [specific step]
2. [specific step]
3. [specific step]

The output should be [description of done state].

Use these tools: [tool1, tool2, tool3]

Constraints:
- [constraint]
- [constraint]

When done, call complete() with a summary of what you built and any issues encountered.
```

### Example: Good Brief

> You are implementing a WebSocket listener in sandbox `ws-test-abc123`. The sandbox already has Node.js installed and the repo cloned at `/home/daytona/myapp`.
>
> Task: Add a WebSocket client in `/home/daytona/myapp/src/listener.ts` that:
> 1. Connects to `wss://api.example.com/events` using the `WS_TOKEN` env var for auth
> 2. Logs each message to stdout with a timestamp
> 3. Reconnects automatically on disconnect (exponential backoff, max 30s)
>
> Use the `ws` npm package (already installed). Export a `startListener(url: string): void` function.
>
> When done, run `npx tsc --noEmit` to verify types compile, then call complete() with the file path and any type errors.

### Example: Bad Brief

> Add a WebSocket listener to the project.

The bad brief will produce mediocre output. The good brief will produce something you can ship.

---

## spawn_worker API

```javascript
spawn_worker({
  description: "Short description (shown in worker list)",
  prompt: "Full instructions/brief for the worker",
  tools: ["tool1", "tool2"],  // subset of your available tools
  model: "workhorse"          // or "reasoning" for complex tasks
})
```

**Parameters:**
- `description` — shown in `list_workers` output; make it scannable
- `prompt` — the full brief; this is what the worker sees
- `tools` — give the worker exactly what it needs (sandbox tools, file tools, etc.)
- `model` — `workhorse` for most tasks; `reasoning` for complex coding, architecture decisions, or multi-step analysis

The worker automatically gets a `complete` tool to file its report when done. The report arrives in your message stream.

---

## Critical Rules

### 🔴 Save work before cleanup

**ALWAYS commit to git or write to board BEFORE deleting a sandbox.** Code dies with the sandbox. This is the most common way to lose work.

Brief your workers to commit their work as part of the task:
> "When done, commit all changes with `git add -A && git commit -m 'feat: ...' && git push`"

Or write key outputs to the board:
> "Save the final output to `/results/analysis.md` on the board using the write tool."

### 🟡 Don't poll for workers

Worker reports arrive automatically. You don't need to:
- Call `list_workers` repeatedly
- Set alarms to check on workers
- Wait idle for workers to finish

Spawn the worker, keep working on other things, and review the report when it arrives.

### 🟢 Give workers the right tools

Workers only have access to tools you explicitly give them. A worker that needs to run shell commands needs sandbox tools. A worker that needs to write files needs file tools. Think through what the worker will need before spawning.

Common tool sets:
- **Coding task in sandbox**: `exec`, `read`, `write`, `edit`, `glob`
- **Research task**: `web_search`, `web_fetch`, `write`
- **Git operations**: `exec` (with sandbox that has git configured)
- **Board file work**: `read`, `write`, `edit`, `glob`, `search`

---

## Model Selection

| Task | Model |
|------|-------|
| Coding, implementation | `workhorse` (most tasks) or `reasoning` (complex architecture) |
| Research, summarization | `workhorse` |
| Complex debugging | `reasoning` |
| Multi-step analysis | `reasoning` |
| File writing, editing | `workhorse` |
| Git operations | `workhorse` |

When in doubt, use `workhorse`. It's faster and cheaper. Upgrade to `reasoning` when the task requires genuine multi-step reasoning or complex code architecture.

---

## Anti-Patterns to Avoid

**❌ Doing the work yourself**
If you're writing code directly, running shell commands yourself, or editing files manually — ask why this isn't a worker.

**❌ Vague briefs**
"Fix the bug" is not a brief. "Fix the null pointer exception in `src/auth.ts` line 47 — the `user` object can be undefined when the session expires" is a brief.

**❌ Polling for results**
Don't call `list_workers` in a loop. Reports arrive automatically.

**❌ Forgetting to save**
Don't delete a sandbox before the worker has committed or transferred its work.

**❌ Under-tooling workers**
A worker that needs to run commands but doesn't have sandbox tools will fail. Give workers what they need.

**❌ Sequential when parallel is possible**
If tasks A and B don't depend on each other, spawn both at once.

---

## Summary

Workers are cheap, fast, and the default. You are the orchestrator — you understand, decide, brief, and review. Workers execute.

The brief is your work product. Write it well.
