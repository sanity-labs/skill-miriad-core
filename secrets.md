# Secrets Management

Miriad has a built-in system for handling sensitive values (API keys, tokens, passwords) without ever exposing them in plaintext to agents or chat history.

## Core Concepts

- **Ephemeral secrets** — When a user pastes a secret in chat, it appears as `[secret:N]` (e.g., `[secret:0]`). The plaintext is held server-side for **15 minutes**, then expires.
- **Channel secrets** — Persistent encrypted storage scoped to a channel. Injected as environment variables into sandboxes.
- **Space secrets** — Same as channel secrets but available across all channels in the space.
- **MCP headers** — Secrets can also be stored as HTTP headers on MCP server configurations.

## The Secret Lifecycle

### 1. User shares a secret in chat

The user pastes a token or key in a message. Miriad redacts it immediately — you'll see:
```
Here is my API key: [secret:0]
```

You **never** see the plaintext value. The `[secret:N]` placeholder is all you get.

### 2. Transfer the secret to permanent storage

Use `transfer_secret` to move the ephemeral value to a permanent destination:

```
transfer_secret({
  messageId: "01KHVH3ZYG...",   // ULID of the message containing [secret:N]
  secretIndex: 0,                // The N from [secret:N]
  destination: "env_secret",     // "env_secret" or "mcp_header"
  key: "MY_API_TOKEN"            // Environment variable name
})
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `messageId` | Yes | ULID of the message with the secret |
| `secretIndex` | Yes | Index N from `[secret:N]` |
| `destination` | Yes | `"env_secret"` (env var) or `"mcp_header"` (MCP server header) |
| `key` | Yes | Variable name (e.g., `MY_TOKEN`) or header key |
| `global` | No | `true` for space-wide, `false` (default) for channel-scoped |
| `template` | No | Wrap the value, e.g., `"Bearer {secret}"` |
| `mcpSlug` | No | Required when `destination` is `"mcp_header"` |

### 3. Secret is available in sandboxes

Channel and space secrets are **automatically injected** as environment variables when a sandbox is created. Important: secrets added *after* a sandbox is created won't appear in that sandbox — you need to create a new sandbox to pick up new secrets.

## Practical Patterns

### Pattern: Store a secret as a channel env var

Most common case — user gives you an API key to use in sandboxes:

```
User: "Here's my Fly.io token: [secret:0]"

→ transfer_secret({
    messageId: "<message ULID>",
    secretIndex: 0,
    destination: "env_secret",
    key: "FLY_API_TOKEN"
  })
```

### Pattern: Forward a secret to an external service

When you need to set a secret on an external service (e.g., Fly.io app secrets), the flow is:

1. **Transfer** the secret to a channel env var first
2. **Create a new sandbox** (so the secret is injected)
3. **Use the env var** in the sandbox to pass it along

```bash
# In sandbox — the secret is available as an env var
flyctl secrets set "ADMIN_TOKEN=$ADMIN_TOKEN" -a my-app
```

### Pattern: Bearer token for MCP server

```
transfer_secret({
  messageId: "<ULID>",
  secretIndex: 0,
  destination: "mcp_header",
  key: "Authorization",
  mcpSlug: "my-mcp-server",
  template: "Bearer {secret}"
})
```

## Plaintext Environment Variables

Not everything is a secret. For non-sensitive config, use `set_environment_var`:

```
set_environment_var({ key: "NODE_ENV", value: "production" })
```

These are visible to everyone and stored in plain text. Use `get_environment` to see all current vars and secrets (secret values are shown as `[secret]` placeholders — you can see the keys but never the values).

## Key Rules

1. **Never ask users to paste secrets in plain text** — they should just paste them and Miriad handles the redaction automatically.
2. **Secrets expire in 15 minutes** — if you don't `transfer_secret` in time, ask the user to share again.
3. **New sandbox required after adding secrets** — existing sandboxes don't pick up newly added secrets. Always create a fresh sandbox.
4. **Key naming convention** — use `SCREAMING_SNAKE_CASE` following the pattern `<PROJECT>_<SERVICE>_<CREDENTIAL_TYPE>` (e.g., `CAST_ORG_FLY_TOKEN`, `JSONSPHERE_ADMIN_TOKEN`).
5. **Channel vs space scope** — default is channel-scoped. Use `global: true` only when the secret needs to be available across all channels.
6. **You can't read secret values** — `get_environment` shows secret key names but never values. This is by design.

## Checking Current Environment

```
get_environment()
```

Returns:
- `vars` — plaintext environment variables (full key-value pairs)
- `secrets` — secret key names only (values are never exposed)
- `resolved` — merged view with `[secret]` placeholders for secret values
