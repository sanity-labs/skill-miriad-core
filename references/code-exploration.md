# Code Exploration

How to efficiently explore and understand codebases in sandboxes. Read this once, internalize the habits, and you'll work faster with less wasted context.

## The Core Principle

**Structure first, details second.** Never read entire files to understand what's in them. Use targeted tools to understand the shape of the code, then read only the specific sections you need.

## Tool Output Limits

Sandbox tools have output limits to protect your context window:

| Tool | Default limit | Notes |
|------|--------------|-------|
| **Read** | 500 lines, 2000 chars/line | Use `offset`/`limit` to navigate. Returns raw source code. |
| **exec** | 10 KB output | Set `extended: true` for up to 50 KB. Overflow saved to temp file. |
| **Grep** | 100 matches | Use `offset` for pagination. Line numbers match Read. |
| **Glob** | 100 files | Use `offset` for pagination. |

When output is truncated, the response tells you where the full output is stored (as a temp file in the sandbox). You can then grep/head/tail that file to find what you need.

## Workflow: Exploring a New Codebase

### 1. Get the lay of the land

```
sandbox_glob({ sandbox: "s", pattern: "*.md" })        # find READMEs, docs
sandbox_glob({ sandbox: "s", pattern: "*.json" })       # find package.json, tsconfig, etc.
sandbox_read({ sandbox: "s", path: "README.md" })       # read the README
sandbox_read({ sandbox: "s", path: "package.json" })    # understand dependencies
```

### 2. Understand the directory structure

```
exec({ sandbox: "s", command: "find . -type f -name '*.ts' | head -50" })
exec({ sandbox: "s", command: "find . -type d -maxdepth 3 | sort" })
```

### 3. Get symbol outlines before reading files

Instead of reading a 500-line file top to bottom, get the symbol outline first:

```
# Quick symbol overview using grep
exec({ sandbox: "s", command: "grep -n 'export\\|^function\\|^class\\|^interface\\|^type\\|^const' src/auth.ts" })

# If universal-ctags is available (check with `which ctags`):
exec({ sandbox: "s", command: "ctags -x --sort=no src/auth.ts" })

# If not installed and you need it:
exec({ sandbox: "s", command: "sudo apt-get install -y universal-ctags 2>/dev/null && ctags -x --sort=no src/auth.ts" })
```

This gives you function names, class names, and line numbers — a table of contents for the file. Then read only the specific function you need:

```
# grep told you `authenticate` is at line 47
sandbox_read({ sandbox: "s", path: "src/auth.ts", offset: 45, limit: 30 })
```

### 4. Use grep to find specific code

Grep is your best friend. It returns line numbers that map directly to Read offsets:

```
# Find where a function is defined
sandbox_grep({ sandbox: "s", pattern: "function handleAuth", include: "*.ts" })

# Find all usages of a type
sandbox_grep({ sandbox: "s", pattern: "AuthConfig", include: "*.ts" })

# Find TODO/FIXME comments
sandbox_grep({ sandbox: "s", pattern: "TODO|FIXME" })
```

When grep tells you something is at line 142, you can `Read(path, offset=140, limit=20)` to see it in context.

### 5. Use exec for targeted extraction

```
# Count lines in a file before deciding how to read it
exec({ sandbox: "s", command: "wc -l src/big-file.ts" })

# See just the imports
exec({ sandbox: "s", command: "head -30 src/auth.ts" })

# See just the exports
exec({ sandbox: "s", command: "grep -n '^export' src/auth.ts" })

# Get a function with surrounding context
exec({ sandbox: "s", command: "sed -n '45,75p' src/auth.ts" })

# Find all files that import a module
exec({ sandbox: "s", command: "grep -rl 'from.*./auth' src/" })
```

## Anti-Patterns to Avoid

### ❌ Reading files sequentially
```
# DON'T do this — wastes context reading code you don't need
sandbox_read({ path: "big-file.ts" })                    # lines 1-500
sandbox_read({ path: "big-file.ts", offset: 500 })       # lines 500-1000
sandbox_read({ path: "big-file.ts", offset: 1000 })      # lines 1000-1500
# ... you've now burned 150 lines of context and maybe found what you need
```

### ✅ Jump to what you need
```
# DO this — find first, read targeted
sandbox_grep({ pattern: "handleAuth", include: "*.ts" }) # → big-file.ts:87
sandbox_read({ path: "big-file.ts", offset: 85, limit: 30 })  # just the function
```

### ❌ Using exec to dump entire files
```
# DON'T — cat bypasses Read limits
exec({ command: "cat src/big-file.ts" })
```

### ✅ Using exec for targeted extraction
```
# DO — extract just what you need
exec({ command: "grep -n 'export function' src/big-file.ts" })
exec({ command: "head -20 src/big-file.ts" })
```

### ❌ Broad glob/grep patterns
```
# DON'T — returns hundreds of results
sandbox_grep({ pattern: "import", include: "*.ts" })
sandbox_glob({ pattern: "*" })
```

### ✅ Narrow your search
```
# DO — specific patterns, specific directories
sandbox_grep({ pattern: "import.*auth", include: "*.ts", path: "src/routes" })
sandbox_glob({ pattern: "*.test.ts", path: "src/routes" })
```

## When to Use Extended exec

Set `extended: true` when you genuinely need more output:

- **Test results**: `exec({ command: "npm test", extended: true })` — test output is often 5-20 KB
- **Build logs**: `exec({ command: "npm run build", extended: true })` — build errors need full context
- **Diffs**: `exec({ command: "git diff", extended: true })` — diffs can be large

Even with `extended: true`, output over 50 KB is saved to a temp file. The response tells you the file path — use grep/head/tail on it.

## Quick Reference

| I want to... | Use this |
|--------------|----------|
| Understand file structure | `Glob` + `exec("find ...")` |
| See what's in a file | `exec("grep -n 'export\|function\|class' file")` |
| Read a specific section | `Read` with `offset` and `limit` |
| Find where something is used | `Grep` with specific pattern |
| Count lines before reading | `exec("wc -l file")` |
| See imports/exports | `exec("grep -n '^import\|^export' file")` |
| Get a function definition | `Grep` to find line → `Read` with offset |
| Compare files | `exec("diff file1 file2")` |
| Check file size | `exec("ls -la file")` or `exec("wc -c file")` |
