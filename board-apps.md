# Board Apps (Micro Apps)

HTML files on the board filesystem are served with proper mime types and open as **micro apps** in an iframe. They can use relative paths for JS, CSS, SVG, and other assets, and can call the Miriad API directly using the user's session — no special authentication needed.

## Platform Context: `window.__miriad`

The platform injects a script tag before the HTML content that sets space context on the window object:

```html
<script>window.__miriad = {"spaceId":"mKo70yF8","channelId":"oRfgW2vN"};</script>
```

Apps read this to make API calls:

```js
const { spaceId, channelId } = window.__miriad || {};
```

This works both inside the iframe and if the page is loaded standalone.

## Calling the Dataset API

Since board apps are served from the same origin as the Miriad API, use **relative URLs** — no CORS issues, no auth tokens needed:

```js
// Query a dataset
const res = await fetch(`/spaces/${spaceId}/datasets/movies/query`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ query: '*[_type == "movie"] | order(popularity desc) [0..9]' }),
});
const { result, ms } = await res.json();
```

### Available REST endpoints (relative)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/spaces/:spaceId/datasets` | Create dataset |
| `GET` | `/spaces/:spaceId/datasets` | List datasets |
| `DELETE` | `/spaces/:spaceId/datasets/:name` | Delete dataset |
| `POST` | `/spaces/:spaceId/datasets/:name/mutate` | Mutate documents |
| `POST` | `/spaces/:spaceId/datasets/:name/query` | GROQ query |
| `GET` | `/spaces/:spaceId/datasets/:name/documents/:docId` | Get document |

## File Structure

Board apps support multi-file structures with relative paths:

```
/my-app/
  index.html        ← entry point (text/html)
  css/style.css      ← external stylesheet (text/css)
  js/api.js          ← JavaScript module (application/javascript)
  js/app.js          ← app logic
  assets/logo.svg    ← images (image/svg+xml)
```

Reference assets with standard relative paths:
```html
<link rel="stylesheet" href="css/style.css">
<script src="js/api.js"></script>
<img src="assets/logo.svg">
```

## Best Practices

- **Use `window.__miriad`** for space/channel context — never hardcode IDs
- **Use relative URLs** for API calls — no need for absolute URLs or CORS
- **16px minimum font** on inputs to prevent Safari auto-zoom
- **`touch-action: manipulation`** on html to eliminate double-tap zoom
- **`viewport-fit=cover`** and safe area padding for mobile
- **Warn gracefully** if `window.__miriad` is missing (e.g., local development)
- **Parallel queries** with `Promise.all()` when loading multiple datasets

## Example: Minimal Dataset App

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My App</title>
</head>
<body>
  <div id="app">Loading...</div>
  <script>
    const { spaceId } = window.__miriad || {};

    async function query(groq) {
      const res = await fetch(`/spaces/${spaceId}/datasets/my-data/query`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query: groq }),
      });
      return res.json();
    }

    async function init() {
      const { result } = await query('*[_type == "item"] | order(_createdAt desc) [0..9]');
      document.getElementById('app').innerHTML = result
        .map(item => `<p>${item.title}</p>`)
        .join('');
    }

    init();
  </script>
</body>
</html>
```
