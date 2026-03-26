# Vision Tool

The `vision` tool analyzes images using different strategies. Pass image sources directly — **you do not need to download images first**.

## Strategies

| Strategy | Description | Requires |
|----------|-------------|----------|
| `describe` | Describe image content using Claude | `ANTHROPIC_API_KEY` in space secrets |
| `detect` | Detect objects with bounding boxes using Gemini | `GEMINI_API_KEY` in environment |
| `colors` | Extract dominant colors (local processing) | Nothing — always available |

## Image Sources

The `image` parameter accepts URLs, board files, cross-channel files, and sandbox files:

```javascript
// URL — pass directly, no need to download first
vision({ strategy: "describe", image: "https://example.com/photo.png" })

// Board file in current channel
vision({ strategy: "describe", image: { path: "/screenshot.png" } })

// Cross-channel board file (by channel name)
vision({ strategy: "colors", image: { channel: "design-channel", path: "/mockup.png" } })

// Cross-channel board file (by filesystem shortId)
vision({ strategy: "detect", image: { fs: "abc123", path: "/diagram.png" } })

// Sandbox file
vision({ strategy: "describe", image: { sandbox: "my-sandbox", path: "/tmp/output.png" } })
```

## Custom Prompts

The `describe` and `detect` strategies accept an optional `prompt` to focus the analysis:

```javascript
vision({
  strategy: "describe",
  image: "https://example.com/ui.png",
  prompt: "Focus on the navigation layout and color scheme"
})
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `strategy` | `"colors" \| "describe" \| "detect"` | Yes | Analysis strategy |
| `image` | string or object | Yes | Image source — URL string or `{ path, channel?, fs?, sandbox? }` |
| `prompt` | string | No | Custom prompt for `describe` / `detect` strategies |

## Discovery

Check the full parameter schema anytime:

```javascript
document_tool({ name: "vision" })
```

This shows all accepted image source formats and available strategies — useful when you're unsure about the exact parameter shape.

## Tips

- **Don't download images to analyze them** — pass URLs directly. The tool handles fetching internally.
- **Use `colors` for quick analysis** — it's local (no API call), always available, and returns structured color data.
- **Board files work directly** — no need to transfer files to a sandbox. Just reference the path.
- **`describe` is the most versatile strategy** — good for general image understanding, UI review, and diagram interpretation.
