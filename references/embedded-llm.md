# Embedded LLM Functions — reason() and comprehend()

Execute scripts have access to embedded LLM functions that process data **inline** — no inference round-trips back to the agent. This keeps your context window clean while enabling sophisticated data processing.

## Why This Matters

Without embedded LLM functions, processing 200 items means either:
- Dumping all 200 into your context window (expensive, noisy)
- Spawning a worker to process them (async, can't compose inline)

With `reason()` and `comprehend()`, you process data inside an execute script and only surface the clean results. Your context window never sees the raw data.

## reason() — Single-Item Deep Processing

Inline LLM call for processing one piece of data. Like `jq()` for unstructured data.

```js
const result = await reason({
  prompt: "Extract all API endpoints. Return JSON: [{method, path, description}]",
  data: someDocument,
  model: "light"  // or "workhorse" or "reasoning"
});
// result.result contains the parsed response
```

**Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `prompt` | string | required | Processing instructions |
| `data` | any | optional | Data to process (serialized as JSON) |
| `model` | string | `"light"` | Model tier: `light` (fast/cheap), `workhorse` (balanced), `reasoning` (deep) |
| `tokenLimit` | number | 20000 | Input token guardrail |
| `maxOutputTokens` | number | 16000 | Max output tokens |

**Returns:** `{ result: any }` — the model's response, parsed as JSON when possible.

### When to Use reason()

- **Extract structure from unstructured text** — meeting notes into action items, HTML into metadata
- **Classify or label** a single document
- **Synthesize** results from a previous comprehend() pass
- **Transform** data between schemas
- **Answer questions** about data you just fetched

### Model Selection for reason()

| Use Case | Model | Why |
|----------|-------|-----|
| Classification, extraction, labeling | `light` | Fast, cheap, accurate enough |
| Synthesis, analysis, essay writing | `workhorse` | Better reasoning, richer output |
| Complex multi-step analysis | `reasoning` | Deep thinking, highest accuracy |

## comprehend() — Batch Fan-Out Processing

Maps a prompt over an array of items with automatic batching and parallelism. **Light model only** — designed for high-throughput, low-cost processing across many items.

```js
const results = await comprehend({
  prompt: "Return JSON: { title: string, genre: string, mood: string }",
  items: arrayOfMovies
});
// results is a flat array of parsed JSON objects
```

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `prompt` | string | Processing instructions applied to each batch |
| `items` | array | Array of items to process |

**Returns:** Array of results. When the model returns valid JSON, results are automatically parsed into objects. When items fit in one batch, results may arrive as individual objects or as a single string containing a JSON array — handle both cases (see "Parsing comprehend Output" below).

### How Batching Works

`comprehend()` automatically groups items into batches of ~150k tokens and processes batches in parallel. You don't manage chunking — just pass the full array.

- 30 items with short text -> typically 1 batch
- 200+ items or items with long text -> multiple parallel batches
- Progress updates show batch completion: `"comprehend: 2/4 batches complete"`

### When to Use comprehend()

- **Classify or tag** hundreds of items (pages, documents, records)
- **Extract** the same fields from many documents
- **Filter by meaning** — "which of these pages contain pricing information?"
- **Score or rank** items by fuzzy criteria
- **Down-select** a large dataset before deeper analysis with reason()

### comprehend() Output Can Be Noisy

The light model is fast but not always precise. This is a feature, not a bug — comprehend is a **recall-optimized** stage. You want it to flag 30 "maybe relevant" items so a smarter model can sort them out, rather than miss the one edge case where the match is subtle.

### Parsing comprehend() Output

comprehend() batches items and returns one result per batch. A batch result may be:
- A **parsed JSON array** (ideal case) — each element corresponds to an input item
- A **string** containing JSON (possibly with markdown code fences) — parse it yourself

Always handle both cases:

```js
const raw = await comprehend({ prompt: "...", items: myItems });

// Flatten batch results into individual items
const results = [];
for (const item of raw) {
  if (Array.isArray(item)) {
    results.push(...item);
  } else if (typeof item === 'object' && item !== null) {
    results.push(item);
  } else if (typeof item === 'string') {
    const cleaned = item.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
    const parsed = JSON.parse(cleaned);
    if (Array.isArray(parsed)) results.push(...parsed);
    else results.push(parsed);
  }
}
```

## The Pipeline Pattern — Fan Out, Then Synthesize

The most powerful pattern combines comprehend and reason in a funnel:

```
Hundreds of items
    |
    v
comprehend()     <-- Fan out: light model, parallel batches
    |                Classify, tag, extract, score
    v
  Filter         <-- JS array methods or jq() to narrow down
    |
    v
reason()         <-- Go deep: workhorse/reasoning model
    |                Synthesize, analyze, generate insights
    v
Clean output     <-- Only this enters your context window
```

### Example: Finding Monster Movies in a Dataset

```js
// Step 1: Fetch a lot of data
const movies = await dataset_query({
  dataset: "movies",
  query: '*[_type == "movie"][0...200] { _id, title, overview, releaseDate }'
});

// Step 2: Fan out with comprehend (light model, auto-batched)
const classified = await comprehend({
  prompt: 'Does this movie involve monsters or non-human threats (aliens, robots, supernatural, metaphorical)? Return JSON: { title, hasMonsters: boolean, monsterType: string | null }',
  items: movies.result
});

// Step 3: Filter to interesting subset
const monsterMovies = classified.filter(m => m.hasMonsters);

// Step 4: Deep synthesis with reason (workhorse model)
const analysis = await reason({
  prompt: "Analyze these monster movies. Group by type, identify decade trends, pick the best metaphorical use. Return JSON.",
  data: monsterMovies,
  model: "workhorse"
});

return analysis.result;
```

200 movies processed, but only the final synthesis enters your context window.

### Example: Content Intelligence Pipeline

```js
// Classify hundreds of crawled pages
const pages = await dataset_query({
  dataset: "crawl-results",
  query: '*[_type == "page"][0...500] { _id, url, title, textContent }'
});

// Fan out: which pages have specific pricing information?
const classified = await comprehend({
  prompt: 'Does this page contain specific pricing (dollar amounts, tiers, rate cards)? Return JSON: { url, hasPricing: boolean, confidence: "high"|"medium"|"low" }',
  items: pages.result
});

// Filter to high-confidence hits
const pricingPages = classified.filter(p => p.hasPricing && p.confidence === "high");

// Refetch full content for just the relevant pages (by ID)
const fullContent = [];
for (const p of pricingPages) {
  const doc = await dataset_query({
    dataset: "crawl-results",
    query: `*[url == "${p.url}"][0]`
  });
  fullContent.push(doc.result[0]);
}

// Deep extraction with workhorse
const pricing = await reason({
  prompt: "Extract all pricing into a structured comparison. Return JSON: { competitors: [{ name, plans: [{ name, price, features }] }] }",
  data: fullContent,
  model: "workhorse"
});

return pricing.result;
```

## Combining with Workers

For large-scale processing, run the pipeline in a worker to keep your context window completely clean:

```js
spawn_worker({
  description: "Classify and analyze 1000 crawled pages",
  prompt: `
    1. Fetch all pages from "crawl-results" dataset
    2. Use comprehend() to classify each page by type
    3. Filter to pricing pages
    4. Use reason() with workhorse to extract structured pricing
    5. Write results to /analysis/pricing-report.json on the board
  `,
  model: "workhorse"
});
```

The worker runs the full pipeline in the background. You review the clean output when it's done.

## Key Design Principles

1. **comprehend for breadth, reason for depth** — don't run reason() on 200 items or comprehend() for synthesis
2. **comprehend output is noisy and that's fine** — it's recall-optimized. Filter before going deep.
3. **Your context window is precious** — these tools exist so raw data never enters it. Only clean results surface.
4. **Compose in pipelines** — comprehend -> filter -> reason is the standard pattern
5. **Use background execute for large pipelines** — `execute({ background: true })` keeps the conversation responsive while processing hundreds of items
6. **Workers can use these too** — a worker running a comprehend->reason pipeline is the cheapest way to process large datasets
