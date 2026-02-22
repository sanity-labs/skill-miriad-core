# Sandboxes

Sandboxes are isolated compute containers for running code, building projects, and testing. They provide shell access, a filesystem, git operations, and network tunnels.

## Providers

- **Daytona** — Built-in, always available. CPU-only. Resources: 1-4 CPU, 1-4 GiB memory, 3-10 GiB disk.
- **RunPod** — BYOK (requires `RUNPOD_API_KEY` in space secrets). GPU support for ML/AI workloads.

Use `explain_options` to see available providers and their capabilities.

## Lifecycle

```
create({ name: "my-sandbox" })                                    // basic
create({ name: "my-sandbox", cpu: 2, memory: 4, disk: 10 })      // custom resources
create({ name: "gpu-box", provider: "runpod", gpu: "RTX4090" })  // GPU sandbox
list()                                                             // list all sandboxes
info({ sandbox: "my-sandbox" })                                    // details
delete({ sandbox: "my-sandbox" })                                  // destroy
```

Sandboxes auto-stop after idle timeout (default 30 minutes). After auto-stop, you need to create a new one.

## Shell

```
exec({ sandbox: "my-sandbox", command: "npm install" })
exec({ sandbox: "my-sandbox", command: "npm test", cwd: "/home/user/project" })
exec({ sandbox: "my-sandbox", command: "long-task", timeout: 300 })  // 5 min timeout
```

Default timeout is 120 seconds. Returns stdout, stderr, and exit code.

## Filesystem

```
Read({ sandbox: "my-sandbox", path: "/home/user/project/src/index.ts" })
Write({ sandbox: "my-sandbox", path: "/home/user/app.js", content: "..." })
Edit({ sandbox: "my-sandbox", path: "/home/user/app.js", old_string: "...", new_string: "..." })
Glob({ sandbox: "my-sandbox", pattern: "*.ts" })
Grep({ sandbox: "my-sandbox", pattern: "TODO", path: "/home/user/project" })
```

**Note**: Sandbox `Glob` is already recursive — use `*.ts` not `**/*.ts`.

## Git

```
git_clone({ sandbox: "my-sandbox", url: "https://github.com/org/repo.git" })
git_status({ sandbox: "my-sandbox" })
git_commit({ sandbox: "my-sandbox", message: "feat: add auth module" })  // auto-stages all
git_push({ sandbox: "my-sandbox" })
git_branch({ sandbox: "my-sandbox", action: "create", name: "feat/auth" })
git_branch({ sandbox: "my-sandbox", action: "checkout", name: "main" })
git_branch({ sandbox: "my-sandbox", action: "list" })
```

Credentials are auto-injected from channel environment — `git_clone` and `git_push` authenticate automatically.

## Tunnels

```
tunnel({ sandbox: "my-sandbox", port: 3000 })
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
- **Auto-stop** — sandboxes stop after idle timeout (default 30min). You'll need to create a new one.
- **Tool installation** — sandbox images may not have all tools pre-installed. Install what you need (e.g., `sudo apt install gh -y`, `npm install -g pnpm`).
- **Broken yarn repo** — Daytona Debian 13 images ship with a stale yarn apt source. Fix with `sudo rm -f /etc/apt/sources.list.d/yarn.list` before `apt update`.
- **Headless only** — no X server. Browser automation tools run headless. Use screenshots for debugging.
- **Secret scrubbing** — all sandbox output is automatically scrubbed for secret values to prevent accidental leakage.
