# chuks_vector

Embedded vector store for Chuks — in-memory and persistent, no external infrastructure needed. The whole thing ships in your single binary.

Vector search is embarrassingly parallel — comparing a query against all stored vectors is a perfect `spawn` workload. On a 16-core machine, `chuks_vector` scans 16 partitions simultaneously.

## Why Embedded?

Most RAG applications use Pinecone, Weaviate, or Qdrant — external services with latency, cost, and operational overhead. For many use cases (small-medium datasets, edge deployment, single-binary apps) an embedded vector store is better:

- **Zero infrastructure** — no database to provision, no docker to run
- **Zero latency** — vectors live in the same process
- **Optional persistence** — writes to a single file, auto-loads on restart
- **Parallel search** — `spawn` + channels across all CPU cores

## Installation

Add to your `chuks.json`:

```bash
chuks add @chuks/vector
```

## Quick Start

```chuks
import { vector } from "pkg/@chuks/vector"

// Create a store — vectors live in your process
var store = vector.create({
    dimensions: 1536,
    metric:     "cosine",
    persist:    "./vectors.db"     // optional: auto-save/load
})

// Index documents
store.upsert("doc1", embedding1, { title: "Chuks intro" })
store.upsert("doc2", embedding2, { title: "Chuks concurrency" })

// Search — parallel comparison across all vectors
var results = await store.search(queryEmbedding, { topK: 5 })

var i: int = 0
while (i < results.length) {
    println(results[i].id + " — score: " + toString(results[i].score))
    i = i + 1
}
```

## Creating a Store

```chuks
var store = vector.create({
    dimensions: 1536,       // vector dimensionality (required)
    metric:     "cosine",   // "cosine" | "euclidean" | "dot" (default: "cosine")
    persist:    "./my.db",  // persistence file path (optional)
    workers:    16,          // parallel search workers (default: 8)
})
```

| Option       | Type   | Default    | Description                |
| ------------ | ------ | ---------- | -------------------------- |
| `dimensions` | int    | _required_ | Vector dimensionality      |
| `metric`     | string | `"cosine"` | Distance metric            |
| `persist`    | string | none       | File path for persistence  |
| `workers`    | int    | 8          | Parallel search goroutines |

## CRUD Operations

### Upsert

```chuks
// Insert or update (same ID replaces existing)
store.upsert("doc1", embeddingVector, { title: "My doc", category: "tutorial" })
```

### Batch Upsert

```chuks
store.upsertBatch([
    { id: "d1", vector: vec1, metadata: { title: "A" } },
    { id: "d2", vector: vec2, metadata: { title: "B" } },
    { id: "d3", vector: vec3, metadata: { title: "C" } },
])
```

### Get

```chuks
var item = store.get("doc1")
// item.id       → "doc1"
// item.vector   → [0.12, -0.34, ...]
// item.metadata → { title: "My doc" }
```

### Delete

```chuks
var deleted = store.delete("doc1")  // true if found and removed
```

### Has

```chuks
if (store.has("doc1")) {
    println("exists")
}
```

### Count

```chuks
println(store.count())  // number of stored vectors
```

### List

```chuks
// All items
var all = store.list(null)

// Paginated
var page = store.list({ offset: 100, limit: 50 })
```

### Clear

```chuks
store.clear()
```

## Search — Parallel Vector Comparison

```chuks
var results = await store.search(queryVector, {
    topK:     10,       // results to return (default: 10)
    minScore: 0.7,      // minimum similarity threshold
    filter: function(meta: any): bool {
        return meta.category == "tutorial"
    }
})

// results is []SearchResult, ordered by score descending
// results[0].id        — most similar document ID
// results[0].score     — similarity score
// results[0].metadata  — attached metadata
```

### How Parallel Search Works

```
10,000 vectors ÷ 8 workers = 8 partitions

Each worker (real OS goroutine via spawn):
  → Scans its 1,250 vectors
  → Computes cosine/euclidean/dot scores
  → Sends results to channel

Main thread:
  → channel.receive × 8
  → Merge, sort, return topK
```

For stores under 500 vectors, search runs sequentially (spawn overhead > benefit).

### Metadata Filtering

Filter runs _after_ similarity computation. For best performance, keep the filter function cheap:

```chuks
var results = await store.search(queryVec, {
    topK: 5,
    filter: function(meta: any): bool {
        return meta.language == "en"
    }
})
```

## Distance Metrics

| Metric        | Range     | Best For                                          |
| ------------- | --------- | ------------------------------------------------- |
| `"cosine"`    | -1 to 1   | General use. Direction matters, magnitude doesn't |
| `"euclidean"` | 0 to 1    | When absolute distance matters (1/(1+dist))       |
| `"dot"`       | unbounded | Pre-normalized vectors (fastest)                  |

## Persistence

When `persist` is set, call `store.save()` to write to disk. On next `vector.create()` with the same path, data auto-loads:

```chuks
// Session 1: create and populate
var store = vector.create({ dimensions: 1536, persist: "./vectors.db" })
store.upsert("doc1", vec1, { title: "Hello" })
store.upsert("doc2", vec2, { title: "World" })
store.save()

// Session 2: auto-loads from file
var store = vector.create({ dimensions: 1536, persist: "./vectors.db" })
println(store.count())  // 2
```

Data is stored as JSON — human-readable and debuggable.

## Vector Math Utilities

The module object also provides standalone math functions:

```chuks
import { vector } from "pkg/@chuks/vector"

// Cosine similarity
var score = vector.similarity(vecA, vecB)

// Euclidean similarity
var dist = vector.euclidean(vecA, vecB)

// Dot product
var dot = vector.dotProduct(vecA, vecB)

// Normalize to unit length (enables faster dot-product comparisons)
var norm = vector.normalize(vec)
```

## Full RAG Example

```chuks
import { embed } from "pkg//embed"
import { vector } from "pkg/@chuks/vector"

// Create the store
var store = vector.create({
    dimensions: 1536,
    metric: "cosine",
    persist: "./knowledge.db"
})

// Index documents
var documents: []string = [
    "Chuks compiles to native Go code via AOT transpilation",
    "The spawn keyword creates real OS goroutines",
    "Channels provide typed, buffered message passing",
]

// Embed and store
var embedResult = await embed.batch(documents, { model: "text-embedding-3-small" })
var i: int = 0
while (i < documents.length) {
    store.upsert("doc" + toString(i), embedResult.vectors[i], { text: documents[i] })
    i = i + 1
}
store.save()

// Search
var queryVec = await embed.single("how does concurrency work?", { model: "text-embedding-3-small" })
var results = await store.search(queryVec, { topK: 2 })

var j: int = 0
while (j < results.length) {
    var doc: any = store.get(results[j].id)
    println(doc.metadata.text + " (score: " + toString(results[j].score) + ")")
    j = j + 1
}
// Output:
// The spawn keyword creates real OS goroutines (score: 0.92)
// Channels provide typed, buffered message passing (score: 0.84)
```

## API Reference

### `vector.create(config)` → `VectorStore`

Create a new embedded vector store.

### `store.upsert(id, vec, metadata)` → `void`

Insert or replace a vector. Validates dimension match.

### `store.upsertBatch(items)` → `void`

Insert multiple vectors. Each item: `{ id, vector, metadata }`.

### `store.get(id)` → `any`

Get vector by ID. Returns `{ id, vector, metadata }` or `null`.

### `store.delete(id)` → `bool`

Remove a vector. Returns true if found.

### `store.has(id)` → `bool`

Check if an ID exists.

### `store.count()` → `int`

Number of stored vectors.

### `store.stats()` → `StoreStats`

Returns `{ count, dimensions, metric, persistent }`.

### `store.list(config)` → `[]any`

List all entries. Config: `{ offset, limit }` for pagination.

### `store.search(query, config)` → `Task<[]any>`

Parallel similarity search. Config: `{ topK, minScore, filter }`. Returns `[{ id, score, metadata }]`.

### `store.save()` → `bool`

Persist to disk. Requires `persist` path in create config.

### `store.clear()` → `void`

Remove all vectors.

### `vector.similarity(a, b)` → `any`

Cosine similarity between two vectors.

### `vector.euclidean(a, b)` → `any`

Euclidean similarity (1/(1+distance)).

### `vector.dotProduct(a, b)` → `any`

Raw dot product.

### `vector.normalize(vec)` → `[]any`

Normalize to unit length.
