# Datasets

Miriad provides a built-in JSON document database powered by **jsonsphere** (Postgres JSONB) with **spandex** (GROQ query engine). Every space gets isolated document storage with full CRUD and query capabilities.

## Core Concepts

- **Dataset** — A named collection of JSON documents within a space (e.g., `blog`, `user-prefs`, `project-data`)
- **Document** — A JSON object with system fields: `_id`, `_type`, `_rev`, `_createdAt`, `_updatedAt`
- **GROQ** — The query language for filtering, projecting, and ordering documents
- **Manifest** — Every dataset has a singleton document (`_id: "manifest"`, `_type: "system.manifest"`) with `title` and `description`

## Three Access Patterns

### 1. Execute Scripts (Agent Tool Calls)

Dataset tools are available inside `execute` scripts. Use `search_tools("dataset")` to discover them:

| Tool | Description |
|------|-------------|
| `miriad__dataset_create` | Create a new dataset (name, title, description) |
| `miriad__dataset_list` | List all datasets in the space |
| `miriad__dataset_delete` | Delete a dataset and all its documents |
| `miriad__dataset_mutate` | Create, update, patch, or delete documents (transactional) |
| `miriad__dataset_query` | Execute a GROQ query |
| `miriad__dataset_get` | Get a single document by `_id` |

**Mutations** are passed as a JSON string. Example:

```json
{
  "dataset": "blog",
  "mutations": "[{\"op\":\"create\",\"document\":{\"_id\":\"post-1\",\"_type\":\"post\",\"title\":\"Hello\"}}]"
}
```

Supported mutation operations:
- `create` — Create a new document (fails if `_id` exists)
- `createOrReplace` — Create or fully replace a document
- `createIfNotExists` — Create only if `_id` doesn't exist (idempotent)
- `patch` — Apply JSON Patch operations to an existing document
- `delete` — Delete a document by `_id`

**Assertions** enable optimistic concurrency (CAS):
```json
{
  "assertions": "[{\"_id\":\"post-1\",\"op\":\"rev\",\"value\":\"01HQ...\"}]",
  "mutations": "[{\"op\":\"patch\",\"_id\":\"post-1\",\"patch\":[{\"op\":\"replace\",\"path\":\"/title\",\"value\":\"Updated\"}]}]"
}
```

**GROQ queries** — the `query` parameter is a GROQ string, optional `params` as JSON string:
```json
{
  "dataset": "blog",
  "query": "*[_type == \"post\"] | order(_createdAt desc) [0..9]"
}
```

### 2. REST API (Browser / Session Auth)

The dataset REST routes are mounted at `/spaces/:spaceId/datasets` and use standard Miriad session authentication (cookies from WorkOS).

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/spaces/:id/datasets` | Create dataset |
| `GET` | `/spaces/:id/datasets` | List datasets |
| `DELETE` | `/spaces/:id/datasets/:name` | Delete dataset |
| `POST` | `/spaces/:id/datasets/:name/mutate` | Mutate documents |
| `POST` | `/spaces/:id/datasets/:name/query` | GROQ query |
| `GET` | `/spaces/:id/datasets/:name/documents/:docId` | Get document |
| `GET` | `/spaces/:id/datasets/:name/listen-token` | Mint signed URL for direct WS listener access |

These routes require a valid user session with membership in the space. Unlike MCP tools, the REST API accepts mutations as a proper JSON array (not a JSON string).

### 3. Space Token (Sandbox Access)

Sandboxes access datasets through the Miriad REST API using a **space token** — a Bearer token automatically provisioned and injected as `MIRIAD_SPACE_TOKEN` env var.

**From a sandbox:**
```bash
# Environment variables are auto-injected:
#   MIRIAD_API_URL     — the Miriad API base URL
#   MIRIAD_SPACE_ID    — the space's short ID
#   MIRIAD_CHANNEL_ID  — the channel's short ID
#   MIRIAD_SPACE_TOKEN — Bearer token for auth

# Query documents
curl -s -X POST "${MIRIAD_API_URL}/spaces/${MIRIAD_SPACE_ID}/datasets/blog/query" \
  -H "Authorization: Bearer ${MIRIAD_SPACE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"query": "*[_type == \"post\"]"}'

# Mutate documents
curl -s -X POST "${MIRIAD_API_URL}/spaces/${MIRIAD_SPACE_ID}/datasets/blog/mutate" \
  -H "Authorization: Bearer ${MIRIAD_SPACE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"mutations": [{"op": "create", "document": {"_id": "doc-1", "_type": "note", "text": "hello"}}]}'

# Get a document
curl -s "${MIRIAD_API_URL}/spaces/${MIRIAD_SPACE_ID}/datasets/blog/documents/doc-1" \
  -H "Authorization: Bearer ${MIRIAD_SPACE_TOKEN}"
```

**Space token scope:** The token grants full CRUD on content/work objects — all dataset operations (create, list, delete, mutate, query, get), plus plan, files, assets, skills, MCP servers, and messages. It is **denied** access to configuration routes (agents, GitHub config, environment variables, secrets).

Sandboxes also receive `MIRIAD_CHANNEL_ID` for accessing channel-level resources (plan, files, etc.) at `/channels/${MIRIAD_CHANNEL_ID}/...`.

## Dataset Name Rules

Dataset names must match: `/^[A-Za-z0-9](?:[A-Za-z0-9]|-(?=[A-Za-z0-9])){0,127}$/` — starts with alphanumeric, 1-128 chars, hyphens allowed but no consecutive (`--`) or trailing hyphens. Case-sensitive.

## Architecture

```
Agent / Browser / Sandbox
  │
  ▼
Miriad API (miriad-redux)
  │ admin token (server-side only)
  ▼
jsonsphere API (Rust, axum)
  │
  ├── Postgres (JSONB) ← source of truth, transactional mutations
  │     └── transactions outbox
  │
  └── Change Consumer (background)
        │ polls outbox, pushes to spandex
        ▼
      Spandex (GROQ query engine) ← derived read index
```

- **Mutations** are immediately durable in Postgres
- **Queries** reflect changes within 1-2 seconds (async sync to spandex)
- The admin token never leaves the server — all access is mediated through Miriad's auth layer

## Lazy Provisioning

Each space maps to a jsonsphere **account** (using the space's `shortId` as the account slug). Accounts are created lazily:

- `dataset_create` auto-provisions the jsonsphere account if it doesn't exist
- `dataset_list` returns `[]` if no account exists (no error)
- `dataset_query`, `dataset_mutate`, `dataset_get` return 404 if the dataset doesn't exist
- `dataset_delete` is a no-op if the dataset doesn't exist

You don't need to set up anything — just start creating datasets.

## Real-time Listeners

Datasets support real-time WebSocket change streams via jsonsphere listeners. To avoid proxying long-lived WS connections through Miriad, a **two-step signed URL flow** is used:

1. Call `GET /spaces/:id/datasets/:name/listen-token` (authenticated via session or space token)
2. Open WebSocket directly to jsonsphere using the returned URL: `new WebSocket(url)`

The signed URL is **one-time-use** (consumed on connect), expires in **60 seconds**, and is **bound to the specific dataset**. On disconnect, repeat from step 1.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/spaces/:id/datasets/:name/listen-token` | Mint a signed URL for direct WebSocket access |

**Response:**
```json
{
  "url": "wss://jsonsphere.fly.dev/accounts/.../datasets/.../listen?token=jsT_...",
  "expires_in": 60
}
```

### GROQ filter and projection

Listeners support server-side filtering and projection using GROQ expressions. Only matching events with the fields you need are sent over the WebSocket.

**Query parameters** (appended to the signed URL from `listen-token`):

| Parameter | Description | Default |
|-----------|-------------|---------|
| `filter` | GROQ filter expression (e.g., `_type == 'post'`) | All events pass through |
| `projection` | GROQ projection expression (e.g., `{title, author, _id, _rev}`) | Document stub (`_id`, `_rev`, `_type`) |

Example URL:
```
wss://jsonsphere.fly.dev/.../listen?token=jsT_...&filter=_type == 'post'&projection={title, author, _id, _rev}
```

**Limits:** Both expressions capped at **4KB**. Projection must produce an object — scalar projections (e.g., `count(*)`) are rejected at parse time.

### Mid-connection reconfiguration

Send a `configure` message to update filter/projection on an active connection:

```json
{
  "type": "configure",
  "filter": "_type == 'post' && status == 'published'",
  "projection": "{title, _id, _rev}"
}
```

**Three-state field semantics:**
- **Omit a field** → keep current value
- **Set to `null`** → clear to default (all events / document stub)
- **Set to a string** → use new GROQ expression

**Server acknowledgment:** Successful configure returns a `configured` event echoing the effective values (including preserved ones from partial updates):

```json
{
  "type": "configured",
  "filter": "_type == 'post' && status == 'published'",
  "projection": "{title, _id, _rev}"
}
```

**Rate limiting:** 1 configure message per second per connection. Excess messages get an error event; current config preserved.

**Error handling:** Invalid GROQ returns an error event and preserves the previous working config — the connection is never dropped.

### Event shapes

Events are JSON objects with a `type` field. Create and update events include a `document` stub; delete events use top-level `id` and `rev` (since the document no longer exists).

**Create / Update:**
```json
{
  "type": "create",
  "dataset": "my-dataset",
  "eventId": "1771696667549-0",
  "document": { "_id": "doc-1", "_rev": "01KJ0NK...", "_type": "post" }
}
```

**Delete:**
```json
{
  "type": "delete",
  "dataset": "my-dataset",
  "eventId": "1771696671745-0",
  "id": "doc-1",
  "rev": "01KJ0NK..."
}
```

| Field | Create/Update | Delete | Description |
|-------|:---:|:---:|-------------|
| `type` | ✓ | ✓ | `"create"`, `"update"`, or `"delete"` |
| `dataset` | ✓ | ✓ | Dataset name |
| `eventId` | ✓ | ✓ | Ordered event ID — use for resuming streams |
| `document` | ✓ | — | Stub with `_id`, `_rev`, `_type` (not the full document) |
| `id` | — | ✓ | Deleted document's `_id` (no underscore prefix) |
| `rev` | — | ✓ | Deleted document's final `_rev` (no underscore prefix) |

**Note:** Create/update events include a document stub (`_id`, `_rev`, `_type`) — not the full document with all fields. To get the full document, fetch it by `_id` after receiving the event.

### Resuming streams (`since`)

Each event includes an `eventId` — a Redis Stream ID (e.g., `"1771696667549-0"`). To resume a stream after reconnection without missing events, pass the last seen `eventId` as the `since` query parameter on the **WebSocket URL** (not on `listen-token`):

```
wss://jsonsphere.fly.dev/accounts/.../listen?token=jsT_...&since=1771696667549-0
```

The `since` parameter is appended to the signed URL returned by `listen-token`. The WebSocket connection replays all events **after** that ID (exclusive — the event you already have is not re-sent), then transitions to live tail.

**When `since` is omitted** (or set to `$`), the connection starts from "now" — no catch-up, straight to live tail. This is the default for new subscriptions that don't need replay.

### Gap detection and retention

Events are retained in a Redis Stream with a **~5000 entry** cap per dataset (`MAXLEN ~ 5000`). The `~` means Redis trims approximately for efficiency.

If a client reconnects with a `since` cursor that has fallen off the retention window (i.e., the events have been trimmed), the server sends a **reset event** and closes the connection:

```json
{"type": "reset", "reason": "gap_too_large"}
```

On receiving a `reset`, the client should:
1. Re-fetch the full current state via GROQ query
2. Reconnect **without** `since` (or with `since=$`) to start fresh from live tail

### Browser usage

```javascript
let lastEventId = null;

function connect() {
  fetch(`/spaces/${spaceId}/datasets/${name}/listen-token`)
    .then(r => r.json())
    .then(({ url }) => {
      // Append since param for resume if we have a cursor
      let wsUrl = url;
      if (lastEventId) wsUrl += `&since=${lastEventId}`;

      const ws = new WebSocket(wsUrl);

      ws.onmessage = (e) => {
        const event = JSON.parse(e.data);

        if (event.type === 'reset') {
          // Gap too large — re-fetch full state and reconnect fresh
          console.warn('Stream gap detected, re-fetching state');
          lastEventId = null;
          ws.close();
          return;
        }

        lastEventId = event.eventId;  // track for resume

        if (event.type === 'delete') {
          console.log('deleted:', event.id);
        } else {
          console.log(event.type, event.document._id);
          // Fetch full document if needed:
          // fetch(`/spaces/${spaceId}/datasets/${name}/documents/${event.document._id}`)
        }
      };

      ws.onclose = () => {
        // Reconnect with jittered backoff
        setTimeout(connect, 1000 + Math.random() * 2000);
      };
    });
}

connect();
```

### Sandbox usage

```bash
# Environment variables are auto-injected:
#   MIRIAD_API_URL, MIRIAD_SPACE_ID, MIRIAD_SPACE_TOKEN

# Step 1: Get a signed URL
TOKEN_RESP=$(curl -s "${MIRIAD_API_URL}/spaces/${MIRIAD_SPACE_ID}/datasets/blog/listen-token" \
  -H "Authorization: Bearer ${MIRIAD_SPACE_TOKEN}")
WS_URL=$(echo "$TOKEN_RESP" | jq -r '.url')

# Step 2: Connect directly to jsonsphere
wscat -c "$WS_URL"

# Or in a script — use websocat, wscat, or any WebSocket client
# The connection receives JSON events for every mutation on the dataset
```

### Key properties

- **One-time use** — the token is consumed on first WebSocket connect. A second attempt returns 401.
- **60s TTL** — unused tokens auto-expire. Always mint a fresh token before connecting.
- **Dataset-bound** — the token only works for the specific dataset it was minted for.
- **No Miriad proxy** — the WebSocket connection goes directly to jsonsphere. Miriad only brokers the auth.
- **Document stubs** — events contain `_id`, `_rev`, `_type` only. Fetch the full document separately if you need all fields.
- **Resumable** — pass `since` on the WebSocket URL (appended to the signed URL) to replay missed events after reconnection. See "Resuming streams" above.
- **Gap detection** — if the `since` cursor has fallen off the ~5000 entry retention window, the server sends `{"type": "reset", "reason": "gap_too_large"}` and closes. Client should re-fetch state and reconnect fresh.
- **Reconnection** — on disconnect, call `listen-token` again with `lastEventId` for a new signed URL. Use jittered exponential backoff to avoid thundering herd.
- **Filterable** — pass `filter` on the WebSocket URL or via `configure` message to receive only matching events. GROQ expressions, 4KB limit.
- **Projectable** — pass `projection` to control which fields are included in events. Must produce an object.
- **Reconfigurable** — send `configure` messages to update filter/projection mid-connection. Server echoes back effective config via `configured` ack.

