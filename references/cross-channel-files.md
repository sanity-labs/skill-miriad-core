# Cross-Channel File Access

Agents can read and search files on other channels' boards using the `channel` parameter on file tools.

## Reading Files from Another Channel

Pass the **channel name** (not the thread/channel ID) to `read`, `glob`, `search`, and `write`:

```
// Read a file from another channel's board
read("/design/001-message-attachments.md", { channel: "miriad-design-argon" })

// List files on another channel's board
glob("/design/**", { channel: "miriad-design-argon" })

// Search across another channel's files
search("attachments", { channel: "miriad-design-argon" })
```

## Use Cases

- **Design → Execution bridge:** Read design docs from design channels to brief execution teams
- **Cross-team reference:** Check specs or findings produced by another team
- **Coordinator pattern:** A coordinator agent (like a backlog keeper) reads outputs from multiple channels

## Key Details

- Use the **channel name** (e.g., `miriad-design-argon`), not the channel short ID
- Read access works across any channel in the same space
- Write access also works with the `channel` parameter — use carefully
- The `edit` and `mv` tools also accept the `channel` parameter
