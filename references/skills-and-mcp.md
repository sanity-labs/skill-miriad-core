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

MCP (Model Context Protocol) servers provide tools. Miriad has built-in servers and supports external ones.

### Built-in Servers

| Server | Tools | Capabilities |
|--------|-------|-------------|
| **miriad** | ~40 | Messaging, files, plan, skills, environment, secrets, roster |
| **miriad-sandbox** | ~17 | Sandbox lifecycle, filesystem, shell, git, tunnels |
| **miriad-dataset** | ~6 | Dataset CRUD, GROQ queries, document operations |
| **miriad-vision** | ~1 | Image analysis (describe, detect objects, extract colors) |

### External Servers

Spaces can configure additional MCP servers for external integrations.

```
list_mcps()                                          // discover available servers
update_my_mcps({ slugs: ["miriad-sandbox", "my-mcp"] })  // self-configure
mcp_status()                                         // check connection status
```

`update_my_mcps` takes effect on your next invocation. Use `mcp_status` to diagnose connection issues.

### GitHub MCP

When GitHub credentials are configured (`GH_TOKEN` available), a GitHub MCP server is auto-included. For full GitHub API access, the `gh` CLI in sandboxes is often more capable — see `github-cli.md`.
