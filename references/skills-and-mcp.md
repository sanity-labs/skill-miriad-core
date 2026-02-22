# Skills & MCP Servers

## Skills

Skills are modular knowledge packs that inject into your system prompt when activated. They provide domain-specific instructions, patterns, and reference material.

### Discovery & Installation

```
skills_discover({ query: "react" })                              // search ecosystem
skills_import({ source: "owner/repo" })                          // install from GitHub
skills_import({ source: "owner/repo", skill: "skill-name" })     // multi-skill repo
skills_list()                                                      // see installed skills
```

### Activation

```
skills_activate({ shortId: "KTtFa8vL" })     // activate for yourself
skills_deactivate({ shortId: "KTtFa8vL" })   // deactivate
```

Activation injects the skill's `SKILL.md` content into your system prompt on your **next invocation** (not immediately). Each active skill adds to your context, so activate only what's relevant.

### Reading Skill Files

```
read({ path: "/SKILL.md", skill: "KTtFa8vL" })
glob({ pattern: "**", skill: "KTtFa8vL" })
search({ query: "patterns", skill: "KTtFa8vL" })
```

Always read `SKILL.md` first — it's the index. Detailed content lives in subdirectories.

### Creating Skills

```
skills_create({ name: "my-patterns" })    // creates skill with template SKILL.md
skills_remove({ shortId: "abc123" })      // remove from space
```

Best practice: keep SKILL.md under 200 lines. Put detailed reference material in separate files.

## MCP Servers

MCP (Model Context Protocol) servers provide tools. Miriad has built-in servers and supports custom external ones.

### Built-in Servers

| Server | Tools | Capabilities |
|--------|-------|-------------|
| **miriad** | ~27 | Messaging, files, plan, skills, roster |
| **miriad-config** | ~11 | Environment, secrets, MCP server management, skill management |
| **miriad-sandbox** | ~20 | Sandbox lifecycle, filesystem, shell, git, tunnels |
| **miriad-dataset** | ~6 | Dataset CRUD, GROQ queries, document operations |
| **miriad-vision** | ~1 | Image analysis (describe, extract colors) |

Built-in servers are always available. Use `update_my_mcps` to add optional ones (like `miriad-vision` or `miriad-config`) to your tool set.

### Attaching MCP Servers

```
update_my_mcps({ slugs: ["miriad-sandbox", "miriad-dataset", "miriad-config"] })
```

**Important:** `update_my_mcps` **replaces** your entire MCP list — always include all servers you want, not just the new one. Changes take effect on your next invocation. Use `set_alarm({ delay: "10s" })` to self-ping and pick up the new tools.

```
mcp_status()    // check which servers are connected and how many tools each provides
```

### Custom MCP Servers

Spaces can register external MCP servers for third-party integrations (e.g., Sanity, Stripe, Notion). The `miriad-config` MCP provides tools to manage these:

#### Registering a Server

```
mcp_put({
  slug: "sanity",
  url: "https://mcp.sanity.io",
  name: "Sanity",
  description: "Sanity.io content management"
})
```

This creates the server registration. If the server uses **OAuth**, the user completes the authorization flow through the Miriad UI. If it uses **API key auth**, use `transfer_secret` to store the key as an MCP header:

```
transfer_secret({
  messageId: "<ULID>",
  secretIndex: 0,
  destination: "mcp_header",
  key: "Authorization",
  mcpSlug: "my-server",
  template: "Bearer {secret}"
})
```

#### Managing Servers

```
mcp_get({ slug: "sanity" })       // get details (URL, name, headers, OAuth status)
delete_mcp({ slug: "sanity" })    // remove a custom server
```

`mcp_get` returns the server's OAuth status (`oauth.connected`, `oauth.authorizedAt`) and header keys (values are redacted).

#### Activating for Yourself

After registering a custom server, add it to your MCP list:

```
update_my_mcps({ slugs: ["miriad-sandbox", "miriad-dataset", "miriad-config", "sanity"] })
```

Then set an alarm to pick up the new tools on your next turn.

### GitHub

GitHub integration uses the `gh` CLI in sandboxes rather than an MCP server. Credentials (`GH_TOKEN`, `GITHUB_TOKEN`) are injected into sandboxes from channel environment. See `github-cli.md` for details.

