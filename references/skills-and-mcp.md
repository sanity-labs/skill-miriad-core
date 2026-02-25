# Skills & Custom MCP Servers

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
skills_activate({ shortId: "miriad-core" })     // activate for yourself
skills_deactivate({ shortId: "miriad-core" })   // deactivate
```

Activation injects the skill's `SKILL.md` content into your system prompt on your **next invocation** (not immediately). Each active skill adds to your context, so activate only what's relevant.

### Reading Skill Files

```
read({ path: "/SKILL.md", skill: "miriad-core" })
glob({ pattern: "**", skill: "miriad-core" })
search({ query: "patterns", skill: "miriad-core" })
```

Always read `SKILL.md` first — it's the index. Detailed content lives in subdirectories.

### Creating Skills

```
skills_create({ name: "my-patterns" })    // creates skill with template SKILL.md
skills_remove({ shortId: "abc123" })      // remove from space
```

Best practice: keep SKILL.md under 200 lines. Put detailed reference material in separate files.

## Custom MCP Servers

Spaces can register external MCP servers for third-party integrations (e.g., Sanity, Stripe, Notion). These provide additional tools accessible through `execute` scripts.

### Registering a Server

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

### Managing Servers

```
mcp_get({ slug: "sanity" })       // get details (URL, name, headers, OAuth status)
delete_mcp({ slug: "sanity" })    // remove a custom server
```

`mcp_get` returns the server's OAuth status (`oauth.connected`, `oauth.authorizedAt`) and header keys (values are redacted).

### Using Custom Server Tools

Once registered and authorized, the server's tools appear in `execute` scripts under the server's slug as a namespace:

```js
// Inside execute — custom MCP server tools are available
const projects = await sanity__list_projects({});
const docs = await sanity__query_documents({
  resource: { projectId: "abc123", dataset: "production" },
  query: '*[_type == "article"] | order(_createdAt desc) [0..9]'
});
```

Use `search_tools("sanity")` inside execute to discover available tools from a custom server.

### GitHub

GitHub integration uses the `gh` CLI in sandboxes rather than an MCP server. Credentials (`GH_TOKEN`, `GITHUB_TOKEN`) are injected into sandboxes from channel environment. See `github-cli.md` for details.
