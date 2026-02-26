# Sandboxes

Sandboxes are isolated compute containers for running code, building projects, and testing. They provide shell access, a filesystem, git operations, and network tunnels.

## Providers

- **Daytona** — Built-in, always available. CPU-only. Resources: 1-4 CPU, 1-4 GiB memory, 3-10 GiB disk.
- **RunPod** — BYOK (requires `RUNPOD_API_KEY` in space secrets). GPU support for ML/AI workloads.

Use `sandbox_explain_options` to see available providers and their capabilities.

## Lifecycle

```
sandbox_create({ name: "my-sandbox" })                                    // basic
sandbox_create({ name: "my-sandbox", cpu: 2, memory: 4, disk: 10 })      // custom resources
sandbox_create({ name: "gpu-box", provider: "runpod", gpu: "RTX4090" })  // GPU sandbox
sandbox_list()                                                             // list all sandboxes
sandbox_info({ sandbox: "my-sandbox" })                                    // details
sandbox_start({ sandbox: "my-sandbox" })                                   // wake a stopped sandbox
sandbox_stop({ sandbox: "my-sandbox" })                                    // hibernate (scale to zero)
sandbox_delete({ sandbox: "my-sandbox" })                                  // destroy permanently
```

Sandboxes **scale to zero** after idle timeout (default 30 minutes) — they're not destroyed. Use `sandbox_start` to wake them back up. This is useful for complex setups that take time to install or compile. For most tasks, create-use-delete is the simplest pattern.

## Shell

```
sandbox_exec({ sandbox: "my-sandbox", command: "npm install" })
sandbox_exec({ sandbox: "my-sandbox", command: "npm test", cwd: "/home/daytona/project" })
sandbox_exec({ sandbox: "my-sandbox", command: "long-task", timeout: 300 })  // 5 min timeout
```

Default timeout is 120 seconds. Returns stdout, stderr, and exit code.

### Output limits (context window control)

Output is **capped at ~2KB by default** to protect context windows. When output exceeds the limit:
- `truncated: true` is set in the response
- Full output is saved to a temp file (e.g., `/tmp/.miriad-cmd-{timestamp}.log`)
- A `hint` tells you where to find the full output

```
// Default: ~2KB output limit
sandbox_exec({ sandbox: "s", command: "npm test" })
// → { truncated: true, full_output: "/tmp/.miriad-cmd-1771849544899.log",
//    hint: "Output truncated... Use exec to grep/head/tail that file." }

// Extended: ~50KB for test results, build logs, diffs
sandbox_exec({ sandbox: "s", command: "npm test", extended: true })
```

**Use `extended: true`** for commands where you need full output (test suites, build logs, large diffs). Default mode is best for most commands — keeps context lean.

**Tip:** When output is truncated, use targeted follow-up commands instead of re-running with extended:
```
sandbox_exec({ sandbox: "s", command: "tail -50 /tmp/.miriad-cmd-1771849544899.log" })
sandbox_exec({ sandbox: "s", command: "grep -n 'FAIL' /tmp/.miriad-cmd-1771849544899.log" })
```

## Filesystem

```
sandbox_read({ sandbox: "my-sandbox", path: "/home/daytona/project/src/index.ts" })
sandbox_write({ sandbox: "my-sandbox", path: "/home/daytona/app.js", content: "..." })
sandbox_edit({ sandbox: "my-sandbox", path: "/home/daytona/app.js", old_string: "...", new_string: "..." })
sandbox_glob({ sandbox: "my-sandbox", pattern: "*.ts" })
sandbox_grep({ sandbox: "my-sandbox", pattern: "TODO", path: "/home/daytona/project" })
```

### Read: line numbers and paging

sandbox_read returns **line-numbered output** with padded alignment:
```
 1	func main() {
 2	    fmt.Println("hello")
 3	}
```

Default limit is **50 lines**. Response includes `total_lines`, `total_bytes`, `offset`, and `lines_returned`. When there's more content, a hint nudges you toward targeted reads:

```
// Default: first 50 lines
sandbox_read({ sandbox: "s", path: "/home/daytona/big-file.ts" })
// → { total_lines: 500, lines_returned: 50, hint: "💡 Use Grep to find specific code..." }

// Jump to a specific region
sandbox_read({ sandbox: "s", path: "/home/daytona/big-file.ts", offset: 100, limit: 30 })

// Max: 2000 lines per read
sandbox_read({ sandbox: "s", path: "/home/daytona/big-file.ts", limit: 2000 })
```

**Best practice:** Use `sandbox_grep` to find what you're looking for first, then `sandbox_read` with `offset` to see the surrounding context. Avoid reading entire large files.

**Note**: `sandbox_glob` is already recursive — use `*.ts` not `**/*.ts`.

## Git

```
sandbox_git_clone({ sandbox: "my-sandbox", url: "https://github.com/org/repo.git" })
sandbox_git_status({ sandbox: "my-sandbox" })
sandbox_git_commit({ sandbox: "my-sandbox", message: "feat: add auth module" })  // auto-stages all
sandbox_git_push({ sandbox: "my-sandbox" })
sandbox_git_branch({ sandbox: "my-sandbox", action: "create", name: "feat/auth" })
sandbox_git_branch({ sandbox: "my-sandbox", action: "checkout", name: "main" })
sandbox_git_branch({ sandbox: "my-sandbox", action: "list" })
```

Credentials are auto-injected from channel environment — `git_clone` and `git_push` authenticate automatically.

## Tunnels

```
sandbox_tunnel({ sandbox: "my-sandbox", port: 3000 })
```

Exposes a port with a public URL for previewing web apps. Default expiry: 1 hour.

## Environment Injection

Channel environment variables and secrets are **automatically injected** into sandboxes at creation time. Key variables available:

| Variable | Description |
|----------|-------------|
| `MIRIAD_API_URL` | Miriad API base URL |
| `MIRIAD_SPACE_ID` | Current space ID |
| `MIRIAD_CHANNEL_ID` | Current channel ID |
| `MIRIAD_SPACE_TOKEN` | Bearer token for API access |
| `GH_TOKEN` / `GITHUB_TOKEN` | GitHub credentials (if configured) |
| `GIT_AUTHOR_NAME` / `GIT_AUTHOR_EMAIL` | Git identity |

System keys (ANTHROPIC_API_KEY, RUNPOD_API_KEY, SSH keys) are **stripped** — never injected into sandboxes.

**Important**: Secrets added *after* sandbox creation won't appear. Create a new sandbox to pick up new secrets.

## Key Gotchas

- **Sandboxes are ephemeral** — code and state are lost when deleted. Always commit+push to git, or write results to channel filesystem/datasets.
- **Auto-stop** — sandboxes scale to zero after idle timeout (default 30min). Use `sandbox_start` to wake them. State and files are preserved.
- **Tool installation** — sandbox images may not have all tools pre-installed. Install what you need (e.g., `sudo apt install gh -y`, `npm install -g pnpm`).
- **Broken yarn repo** — Daytona Debian 13 images ship with a stale yarn apt source. Fix with `sudo rm -f /etc/apt/sources.list.d/yarn.list` before `apt update`.
- **Headless only** — no X server. Browser automation tools run headless. Use screenshots for debugging.
- **Secret scrubbing** — all sandbox output is automatically scrubbed for secret values to prevent accidental leakage.
