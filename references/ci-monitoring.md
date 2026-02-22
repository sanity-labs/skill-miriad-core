# CI Monitoring from Sandboxes

The GitHub MCP server doesn't expose workflow run tools. To check CI status (GitHub Actions), use `curl` against the GitHub REST API from a sandbox.

`$GITHUB_TOKEN` is automatically injected into sandbox environments from channel secrets.

## List Recent Workflow Runs

```bash
# Last 5 runs across all branches
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runs?per_page=5" \
  | jq '.workflow_runs[] | {id, status, conclusion, head_branch, created_at, html_url}'
```

## Filter by Branch

```bash
# Runs for a specific branch
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runs?branch=feat/my-feature&per_page=3" \
  | jq '.workflow_runs[] | {id, status, conclusion, head_branch, created_at, html_url}'
```

## Check Jobs for a Specific Run

```bash
# Get individual job results (typecheck, lint, test, build, etc.)
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runs/{run_id}/jobs" \
  | jq '.jobs[] | {name, status, conclusion}'
```

## Typical Workflow

1. **After pushing a commit** — check that CI picks it up and starts running
2. **Before merging a PR** — verify the branch CI is green
3. **After merging to main** — set a ~3min alarm, then check that the deploy succeeds

```bash
# Quick one-liner: latest run status for a branch
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runs?branch=main&per_page=1" \
  | jq '.workflow_runs[0] | {status, conclusion, created_at, html_url}'
```

## Useful `jq` Filters

```bash
# Only failed runs
| jq '.workflow_runs[] | select(.conclusion == "failure") | {id, head_branch, created_at, html_url}'

# Runs still in progress
| jq '.workflow_runs[] | select(.status == "in_progress") | {id, head_branch, created_at}'

# Compact summary table
| jq -r '.workflow_runs[] | [.head_branch, .status, .conclusion // "—", .created_at] | @tsv'
```

## Notes

- **GitHub token scope** — needs `repo` or `actions:read` scope to access workflow runs
- **Rate limits** — GitHub API allows 5,000 requests/hour with token auth. CI checks are lightweight.
- **Why not the GitHub MCP?** — The GitHub MCP server provides tools for repos, issues, PRs, and code search, but doesn't expose the Actions/workflow runs API. `curl` from a sandbox fills this gap.
