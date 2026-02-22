# Environment & Configuration

Channels have a layered environment system with plaintext variables and encrypted secrets.

## Two Types

- **Plaintext variables** — visible to everyone, settable by agents. For non-sensitive config (NODE_ENV, LOG_LEVEL, project settings).
- **Secrets** — encrypted, values hidden from agents. For API keys, tokens, passwords. Set by users via UI or `transfer_secret`.

## Viewing the Environment

```
get_environment()
```

Returns:
- `vars` — plaintext variables (full key-value pairs)
- `secrets` — secret metadata only (keys, scope, setBy, setAt — **never values**)
- `resolved` — merged view with `[secret]` placeholders for secret values

## Plaintext Variables

```
set_environment_var({ key: "NODE_ENV", value: "production" })
set_environment_var({ key: "PROJECT_NAME", value: "my-app" })
delete_environment_var({ key: "NODE_ENV" })
```

Key must match `[A-Z][A-Z0-9_]{0,127}`.

## Resolution Order

When the same key exists in multiple layers, later layers win:

1. **Global secrets** (space-wide)
2. **Plaintext variables** (channel-scoped)
3. **Local secrets** (channel-scoped)

A local secret with key `MY_TOKEN` shadows a global secret with the same key.

## Global vs Local Secrets

- **Global** (`global: true`) — available across all channels in the space. Set once, used everywhere.
- **Local** (default) — scoped to the current channel. Shadows globals with the same key.

## Auto-Provisioned Secrets

Some secrets are created automatically:

| Secret | How | TTL |
|--------|-----|-----|
| `MIRIAD_SPACE_TOKEN` | Auto-created on first environment resolution | Permanent |
| `GH_TOKEN` / `GITHUB_TOKEN` | Minted from GitHub App installation or PAT | 55 min (App), permanent (PAT) |

### GitHub Credentials

Two modes, configured per channel in the UI:

- **GitHub App** — Install the Miriad GitHub App for scoped access to specific repos. Tokens are installation-scoped and auto-refreshed (55-minute TTL).
- **Personal Access Token (PAT)** — Store a PAT as the global secret `GITHUB_PAT`. Full access to everything your GitHub account can reach.

Both modes inject `GH_TOKEN` and `GITHUB_TOKEN` into sandboxes.

## Sandbox Injection

All resolved environment variables and secrets are injected into sandboxes at creation time. System keys are stripped for security (ANTHROPIC_API_KEY, RUNPOD_API_KEY, SSH keys, etc.).

See `secrets.md` for the full secret lifecycle (ephemeral → transfer → permanent) and `sandboxes.md` for which variables are available in sandboxes.
