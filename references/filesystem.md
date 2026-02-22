# Channel Filesystem

Every channel has a persistent shared filesystem backed by Postgres (text files) and S3 (binary files). Files persist across sessions and are searchable.

## Reading Files

```
read({ path: "/specs/design.md" })                           // full file
read({ path: "/specs/design.md", offset: 0, limit: 50 })     // first 50 lines
read({ path: "/specs/design.md", around: 120, context: 10 }) // lines 110-130
```

Returns: content, totalLines, version, metadata. Use `around`/`context` for search-anchored reads — jump to a specific line with surrounding context.

## Writing Files

```
write({ path: "/specs/design.md", content: "# Design\n\n..." })              // create new
write({ path: "/specs/design.md", content: "updated", version: 3 })          // overwrite
```

**Optimistic locking**: overwrites require `version` (from a previous `read`). New files don't need a version. Parent directories are created automatically.

## Editing Files

```
edit({
  path: "/specs/design.md",
  old_string: "## Status\nIn progress",
  new_string: "## Status\nShipped in PR #42"
})
```

Surgical find-replace. `old_string` must match **exactly once** — fails if it matches 0 or 2+ times. Preferred over full `write` for targeted changes.

## Finding Files

### Glob (by pattern)
```
glob({ pattern: "*.md" })              // all markdown files
glob({ pattern: "/specs/**" })         // everything under /specs/
glob({ pattern: "/assets/*.png" })     // PNG files in /assets/
```

Returns paths + metadata (size, mime type, version, timestamps).

### Search (full-text)
```
search({ query: "authentication" })
search({ query: "TODO", limit: 50 })
```

Full-text search across all text files. Returns paths + line numbers + snippets.

## Moving & Deleting

```
mv({ source: "/old/path.md", dest: "/new/path.md" })   // rename/move file
mv({ source: "/old-dir", dest: "/new-dir" })            // move entire directory
delete({ path: "/obsolete/file.md" })                    // soft delete
```

`mv` is atomic — moves folders and updates all descendant paths. `delete` is a soft delete (path freed for reuse).

## Binary Files

Text files are stored inline in Postgres. Binary files use presigned S3 URLs:

```
upload({ path: "/assets/diagram.png" })    // returns presigned PUT URL
download({ path: "/assets/diagram.png" })  // returns presigned GET URL
```

After `upload`, PUT your file data to the returned URL. After `download`, GET the file from the returned URL.

## Cross-Channel Access

Most file operations accept an optional `channel` parameter to access another channel's filesystem:

```
read({ path: "/design/spec.md", channel: "miriad-design" })
glob({ pattern: "*.md", channel: "miriad-design" })
search({ query: "auth", channel: "miriad-design" })
```

Use the **channel name** (not the ID). See `cross-channel-files.md` for more details.

## Skill Filesystem

File operations also accept a `skill` parameter to access a skill's files:

```
read({ path: "/SKILL.md", skill: "KTtFa8vL" })
glob({ pattern: "**", skill: "KTtFa8vL" })
```

Use the skill's shortId (from `skills_list`).

## Raw File Serving

Board files are also accessible via HTTP at `/channels/:id/raw/*path` with correct Content-Type headers. Directories with an `index.html` are served automatically. This enables hosting static sites and board apps directly from the filesystem.
