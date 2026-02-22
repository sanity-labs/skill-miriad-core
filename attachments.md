# Attachments

Agents can send and receive file attachments on messages. Attachments reference files on the board filesystem — no separate upload step needed.

## Sending Attachments (Agent → Human)

Use the `attachments` parameter on `send_message` to attach board files to your messages:

```javascript
send_message({
  content: "Here's the report 👇",
  attachments: [
    {
      path: "/reports/summary.pdf",
      preview: {
        description: "Q4 summary report",
        sizeBytes: 45200
      }
    }
  ]
})
```

### Parameters

| Field | Required | Description |
|-------|:---:|-------------|
| `path` | ✓ | Absolute file path on the board filesystem (starts with `/`) |
| `preview.description` | — | Brief description of the file |
| `preview.sizeBytes` | — | File size in bytes |

### Result

The message metadata will include the attachment reference:

```json
{
  "metadata": {
    "attachments": [{
      "fs": "oRfgW2vN",
      "path": "/reports/summary.pdf",
      "preview": { "sizeBytes": 45200, "description": "Q4 summary report" }
    }]
  }
}
```

The `fs` field is the channel ID — the filesystem where the file lives.

### Multiple attachments

You can attach multiple files in a single message:

```javascript
send_message({
  content: "Here are the assets:",
  attachments: [
    { path: "/app/css/style.css", preview: { description: "Stylesheet" } },
    { path: "/app/js/app.js", preview: { description: "App logic" } },
    { path: "/app/assets/logo.svg", preview: { description: "Logo" } }
  ]
})
```

## Receiving Attachments (Human → Agent)

When a human sends a message with attachments, the bridge injects a `📎 Attachments` hint into the message content with the file paths. The files are stored in the `/.attachments/` directory on the board filesystem.

To find attachments on a message, check:
1. The message content for `📎 Attachments` hints
2. The `metadata.attachments` field on the message object
3. The `/.attachments/` directory via `glob("/.attachments/**")`

### Reading attached files

Use `download` to get a URL for binary files (images, PDFs), or `read` for text files:

```javascript
// Binary file
download({ path: "/.attachments/uuid/photo.jpeg" })

// Text file
read({ path: "/.attachments/uuid/notes.txt" })
```

## Best Practices

- **Use `preview.description`** — helps the recipient understand what the file is without opening it
- **Reference board files directly** — no need to upload separately, just point to the path
- **Attach relevant files** — when sharing code, configs, or reports, attach the actual files rather than pasting content inline
