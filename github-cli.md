# GitHub CLI (`gh`) in Sandboxes

The `gh` CLI is the most powerful way to interact with GitHub from a sandbox — repos, issues, PRs, releases, gists, Actions, and more.

## Credentials

GitHub credentials are configured per channel in the **Environment** pane. There are two options:

- **GitHub OAuth App** — Install the Miriad GitHub app for granular, scoped access. Best for teams that want fine-grained control over which repos and permissions are granted.
- **Personal Access Token (PAT)** — Configure a PAT in system settings for full access to everything your GitHub account can reach.

Credentials can be changed at any time. They're injected into sandboxes as `GH_TOKEN` and `GITHUB_TOKEN` environment variables.

> **Note:** If old credentials are lingering after a change, restart the sandbox (`sandbox_stop` + `sandbox_start`) or delete it and create a new one to pick up the fresh credentials.

## Installation

The Daytona sandbox image (Debian 13) has `gh` available in the apt repos. One quirk: the image ships with a broken yarn apt source that needs to be removed first.

```bash
# Remove broken yarn repo (stale GPG key on Daytona image)
sudo rm -f /etc/apt/sources.list.d/yarn.list

# Install gh
sudo apt update && sudo apt install gh -y
```

This installs in ~10 seconds. No further setup needed — `gh` automatically detects `GH_TOKEN` from the environment.

## Verify

```bash
gh auth status
```

Expected output:
```
github.com
  ✓ Logged in to github.com account <username> (GH_TOKEN)
  - Active account: true
  - Git operations protocol: https
  - Token scopes: 'gist', 'read:org', 'repo', 'workflow'
```

## Common Operations

```bash
# Repos
gh repo list <org> --limit 10
gh repo clone <owner>/<repo>
gh repo view <owner>/<repo>

# Issues
gh issue list --repo <owner>/<repo>
gh issue create --repo <owner>/<repo> --title "Bug" --body "Details"
gh issue view 123 --repo <owner>/<repo>

# Pull Requests
gh pr list --repo <owner>/<repo>
gh pr create --title "Fix" --body "Description" --base main
gh pr view 42 --repo <owner>/<repo>
gh pr checkout 42
gh pr merge 42 --squash

# Releases
gh release list --repo <owner>/<repo>
gh release create v1.0.0 --title "v1.0.0" --notes "Release notes"

# Gists
echo "content" | gh gist create --filename "file.txt"

# Actions / Workflows
gh run list --repo <owner>/<repo>
gh workflow run deploy.yml --repo <owner>/<repo>

# Search
gh search repos "topic:mcp language:typescript"
gh search issues "bug label:critical" --repo <owner>/<repo>

# API (raw GitHub REST/GraphQL)
gh api repos/<owner>/<repo>/commits --jq '.[0].sha'
gh api graphql -f query='{ viewer { login } }'
```

## Tips

- **`gh` replaces the GitHub MCP** — it's more powerful and supports the full GitHub API surface
- **Token expiry** — OAuth tokens are short-lived (~55 min). For long-running sandboxes, you may need to recreate the sandbox to get fresh tokens
- **`gh api`** is a Swiss Army knife — call any GitHub REST or GraphQL endpoint directly with automatic auth
- **JSON output** — add `--json <fields>` to most commands for machine-readable output, pipe through `--jq` for filtering
