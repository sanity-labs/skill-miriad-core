# Introducing Miriad to New Human Users

A guide for agents helping new human users get oriented on the Miriad platform.

## First Interaction

When a new user joins a channel, they may not know what's possible. Start by understanding what they want to accomplish, then show them the relevant capabilities. Don't dump a feature list — introduce things as they become relevant.

**Good first response:**
> "Welcome! I can help you with [their stated goal]. To get started, [specific next step]. As we go, I'll show you the tools we have available."

## Key Concepts to Introduce Gradually

### The Channel
- This is your workspace — a persistent environment with shared files, a plan board, and message history
- Everything here persists between sessions — files, plans, datasets, messages
- Multiple agents and humans can collaborate in the same channel

### Files & The Board
- The channel has a shared filesystem — I can create, read, and edit files
- Files are visible in the Board panel on the right side of the screen
- HTML files open as interactive micro apps in the browser
- You can also upload files by dragging them into the chat

### The Plan
- We can track work with specs (what to build) and tasks (how to build it)
- The Plan panel shows active work items — useful for coordinating on larger projects
- I can create, update, and manage plan items as we work

### Sandboxes
- I can spin up compute environments to run code, build projects, and test things
- Sandboxes have shell access, git, and can expose ports for previewing web apps
- Code in sandboxes is ephemeral — I'll commit important work to git or save results to the board

### Datasets
- For structured data, we have a built-in JSON document database
- I can create datasets, store documents, and query them with GROQ
- Board apps can read from datasets to build interactive dashboards and tools

### Secrets & Configuration
- If you need to give me API keys or tokens, just paste them in chat — Miriad automatically redacts and encrypts them
- I'll store them securely as environment variables that get injected into sandboxes
- You'll never see me echo back a secret value

## Common First Tasks

### "I want to build something"
1. Discuss the idea, create a spec on the plan board
2. Spin up a sandbox, clone a repo or start fresh
3. Build iteratively — I can run code, show previews via tunnels, commit to git

### "I want to analyze some data"
1. Create a dataset, load the data
2. Query with GROQ or build a board app for visualization
3. Results persist in the dataset for future reference

### "I want to set up a project"
1. Configure GitHub credentials (paste a PAT or install the GitHub App)
2. Set up environment variables for the project
3. Clone the repo in a sandbox, start working

### "I want to automate something"
1. Understand the workflow
2. Build it in a sandbox with the right tools (gh CLI, curl, scripts)
3. Test and iterate

## Tips for Working with Humans

- **Don't overwhelm** — introduce features as they become relevant, not all at once
- **Show, don't tell** — create a file, run some code, show a preview. Concrete demos beat abstract explanations.
- **Explain what you're doing** — "I'm creating a sandbox to run this code" is better than silently calling tools
- **Offer the plan board** for anything beyond a quick task — it helps humans track progress
- **Ask about preferences** — some users want to see every step, others just want results
- **Point out the UI** — "You can see the files in the Board panel" helps users discover the interface
