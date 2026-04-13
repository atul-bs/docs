# Pinecone Auto-Tracing Instrumentation — Specification

## Summary

First vector database instrumentation in the SDK. Traces 6 Pinecone SDK methods (`@pinecone-database/pinecone` v4-v7) automatically when `Observe.init()` is called. No user code changes required.

**All inputs and outputs are always captured.** Text is captured as-is; vectors are captured as dimension count only (actual floats are too large). There is no `traceContent` flag — everything is always visible for full RAG pipeline observability.

---

## What Gets Traced

| # | SDK Method | Span Name | Inference Mode | Operation |
|---|---|---|---|---|
| 1 | `index.query()` | `pinecone.query` | **Non-inference only** | Vector/ID similarity search — user provides pre-computed embeddings |
| 2 | `index.upsert()` | `pinecone.upsert` | **Non-inference only** | Insert/update vectors — user provides pre-computed embeddings |
| 3 | `index.searchRecords()` | `pinecone.search` | **Both** | Text search (inference) or vector/ID search (non-inference) |
| 4 | `index.upsertRecords()` | `pinecone.upsert_records` | **Inference only** | Insert text records — Pinecone generates embeddings server-side |
| 5 | `pc.inference.embed()` | `pinecone.embed` | **Inference only** | Generate embeddings via Pinecone's hosted models |
| 6 | `pc.inference.rerank()` | `pinecone.rerank` | **Inference only** | Re-score documents against a query for relevance |

### Inference vs Non-Inference

| Mode | What it means | Input type | Index requirement |
|---|---|---|---|
| **Inference** | Pinecone generates embeddings server-side from your text | Text strings | Index must have integrated embedding model configured |
| **Non-inference** | You generate embeddings yourself (via OpenAI, Gemini, etc.) and pass vectors | Float arrays (768-4096 dimensions) | Any dense/sparse index |
| **Both** | `searchRecords()` supports either text or vector input | Text OR float array | Works with both index types |

---

## Data Routing

```
Span
 ├── langfuse.observation.input   → Input panel in UI   (structured JSON)
 ├── langfuse.observation.output  → Output panel in UI  (structured JSON)
 ├── pinecone.* attributes        → Metadata panel      (flat key-value metrics)
 └── db.system / duration_ms / status / error.*  → Common fields
```

---

## Core Principle: Always Capture Everything

| Input type | What we capture | Why |
|---|---|---|
| **Text** (inference) | Actual text string | Essential for debugging retrieval — it's the user's question/document |
| **Vector** (non-inference) | Dimension count only (e.g. `3072`) | Actual floats are 30-50KB — too large, not useful for debugging |
| **ID** (id-based) | The ID string | Lightweight, tells you which vector was used as query |

There is no content gating flag. All text is always captured. All vectors are always dimensions-only. This matches the Langfuse/LangSmith/Braintrust industry standard.

---

## Method Details

### 1. `pinecone.query` — Non-inference only

`query()` always takes a **vector or ID** as input — never text. Users generate embeddings externally (Gemini, OpenAI, etc.) and pass the float array. There is no text-based query variant for this method; for text search, use `searchRecords()` on an inference index.

**Input (`langfuse.observation.input`) — always captured:**

Vector query (user passes pre-computed embedding):
```json
{ "type": "vector", "topK": 3, "namespace": "", "vectorDimensions": 3072, "includeMetadata": true, "includeValues": false }
```

ID-based query (find vectors similar to an existing vector by its ID):
```json
{ "type": "id", "topK": 3, "namespace": "", "id": "chunk-18", "includeMetadata": true, "includeValues": false }
```

**Output (`langfuse.observation.output`) — always captured with full content:**

```json
{
  "matches": [
    { "id": "chunk-18", "score": 0.687, "metadata": { "content": "Cloud computing...", "source": "base" } }
  ],
  "matchesCount": 3
}
```

**Metadata (`pinecone.*`):** `top_k`, `namespace`, `vector_dimensions`, `include_metadata`, `include_values`, `has_filter`, `matches_count`, `usage.read_units`

### 2. `pinecone.upsert` — Non-inference only

`upsert()` takes pre-computed vectors (not text). The input captures each vector's `id` and `metadata` (user-defined labels like source, content preview) but **never the actual float array** — only the dimension count.

**Input (`langfuse.observation.input`) — always captured:**

```json
{
  "vectorCount": 1,
  "vectorDimensions": 3072,
  "vectors": [
    { "id": "test-all-123", "metadata": { "content": "instrumentation test", "source": "test-script" } }
  ]
}
```

Note: `metadata` here is the user-defined key-value pairs stored alongside the vector (not the embedding text). The actual float array (`values`) is **never** captured.

**Output:** Not set (upsertedCount and writeUnits are in metadata).

**Metadata (`pinecone.*`):** `vector_count`, `vector_dimensions`, `namespace`, `upserted_count`, `usage.write_units`

> Note: Pinecone's upsert API currently returns an empty object `{}` — `write_units` is not present in the response. The instrumentation checks for it but it will only appear if Pinecone adds it in a future API version.

### 3. `pinecone.search` — Both inference and non-inference

**Input (`langfuse.observation.input`) — always captured:**

Text search (inference):
```json
{ "type": "text", "topK": 3, "namespace": "__default__", "query": "cloud computing primitives" }
```

Vector search (non-inference):
```json
{ "type": "vector", "topK": 3, "namespace": "__default__", "queryVectorDimensions": 3072 }
```

ID-based search:
```json
{ "type": "id", "topK": 3, "namespace": "__default__", "queryId": "chunk-18" }
```

With filter:
```json
{ "type": "text", "topK": 3, "namespace": "__default__", "query": "cloud computing", "filter": { "source": "base" } }
```

**Output (`langfuse.observation.output`) — always captured:**

```json
[
  { "_id": "doc-1", "_score": 0.824, "fields": { "chunk_text": "Cloud computing delivers IT resources..." } }
]
```

**Metadata (`pinecone.*`):** `namespace`, `top_k`, `query_text_length`, `has_rerank`, `rerank_model`, `rerank_top_n`, `results_count`, `usage.read_units`

### 4. `pinecone.upsert_records` — Inference only

**Input (`langfuse.observation.input`) — always captured:**

```json
{
  "recordsCount": 1,
  "namespace": "__default__",
  "records": [
    { "_id": "doc-1", "chunk_text": "Cloud computing delivers IT resources on demand." }
  ]
}
```

**Output:** Not set (writeUnits is in metadata).

**Metadata (`pinecone.*`):** `records_count`, `namespace`, `usage.write_units`

> Note: Pinecone's upsertRecords API currently returns an empty object `{}` — `write_units` is not present in the response. The instrumentation checks for it but it will only appear if Pinecone adds it in a future API version.

### 5. `pinecone.embed` — Inference only

**Input (`langfuse.observation.input`) — always captured:**

```json
{
  "model": "multilingual-e5-large",
  "inputType": "passage",
  "inputs": ["What is cloud computing?", "How does vector search work?"],
  "inputCount": 2
}
```

**Output:** Not set. **Embedding float arrays are never captured** — only dimension count in metadata.

**Metadata (`pinecone.*`):** `model`, `input_count`, `input_type`, `dimensions`, `usage.total_tokens`

### 6. `pinecone.rerank` — Inference only

**Input (`langfuse.observation.input`) — always captured:**

```json
{
  "model": "bge-reranker-v2-m3",
  "topN": 3,
  "query": "What is cloud computing?",
  "documents": [
    { "text": "Cloud computing delivers IT resources on demand over the internet." },
    { "text": "AWS provides cloud infrastructure services globally." }
  ]
}
```

**Output (`langfuse.observation.output`) — always captured:**

```json
[
  { "index": 0, "score": 0.993, "document": { "text": "Cloud computing delivers..." } },
  { "index": 2, "score": 0.001, "document": { "text": "AWS provides cloud infrastructure..." } }
]
```

**Metadata (`pinecone.*`):** `model`, `documents_count`, `top_n`, `results_count`, `usage.rerank_units`

---

## Common Span Attributes (all 6 methods)

| Attribute | Type | Description |
|---|---|---|
| `db.system` | `"pinecone"` | OTel semantic convention |
| `db.collection.name` | string | Index name/host |
| `duration_ms` | number | Call duration in ms |
| `status` | `"success"` / `"error"` | Outcome |
| `error.type` | string | Error class name (on failure) |
| `error.message` | string | Error message (on failure) |

### Usage / Cost Tracking

| Attribute | Type | Methods | Description |
|---|---|---|---|
| `pinecone.usage.read_units` | number | query, searchRecords | Read units consumed (Pinecone serverless billing) |
| `pinecone.usage.write_units` | number | upsert, upsertRecords | Write units consumed (not currently returned by Pinecone API) |
| `pinecone.usage.total_tokens` | number | embed | Tokens processed by the embedding model |
| `pinecone.usage.rerank_units` | number | rerank | Rerank units consumed |

---

## What is NEVER Captured

- Raw vector float arrays (only dimension count)
- Embedding float arrays (only dimension count)
- Write units for upsert/upsertRecords (Pinecone API returns empty `{}` — no usage data available)

These are too large (30-50KB per vector) and not useful for debugging. Matches Langfuse/LangSmith/Braintrust industry standard.

---

## Summary Table

| Method | Inference Mode | Input type field | Input captured | Output captured | Usage Tracked |
|---|---|---|---|---|---|
| **query** | Non-inference | `type: "vector"` or `type: "id"` | topK, namespace, dims/id, filter, includeMetadata | Full matches: id + score + chunk content | `read_units` |
| **upsert** | Non-inference | — | vectorCount, dims, namespace, vector IDs+metadata | Not set | None (API returns `{}`) |
| **search** | Both | `type: "text"` or `type: "vector"` or `type: "id"` | topK, namespace, query text/dims/id, filter | Full result hits with content | `read_units` |
| **upsertRecords** | Inference | — | recordsCount, namespace, full records | Not set | None (API returns `{}`) |
| **embed** | Inference | — | model, inputType, text inputs, inputCount | Not set | `total_tokens` |
| **rerank** | Inference | — | model, topN, query text, all documents | Ranked results with scores | `rerank_units` |

---

## Compatibility

| Pinecone SDK | Supported | Methods |
|---|---|---|
| v4.x (Oct 2024) | Yes | query, upsert |
| v5.x (Feb 2025) | Yes | query, upsert + possibly more |
| v6.x (May 2025) | Yes | All 6 |
| v7.x (Jan 2026, current) | Yes | All 6 |
| v8.x (not released) | No — silently skips |

---

## Module Resolution

Pinecone is a peer dependency. Uses `createRequire(process.cwd() + '/package.json')` to resolve from the user's app, not from the SDK's own node_modules.

---

## Industry Standards

| Concept | OTel Standard | OpenLLMetry | Our Attribute |
|---|---|---|---|
| Database system | `db.system` | `db.system` | `db.system` — exact match |
| Collection | `db.collection.name` | `db.collection.name` | `db.collection.name` — exact match |
| Read/write units | — | `pinecone.usage.*` | `pinecone.usage.*` — matches |
| Error type | `error.type` | `error.type` | `error.type` — exact match |
| Query input | `gen_ai.retrieval.query.text` | via event | `langfuse.observation.input` JSON |
| Results | `gen_ai.retrieval.documents` | via event | `langfuse.observation.output` JSON |

We use `pinecone.*` namespaced metadata + `langfuse.observation.input/output` for UI. Raw vectors are never captured (matches Langfuse, LangSmith, Braintrust — only Traceloop captures them).

---

## Files

| File | Lines | Purpose |
|---|---|---|
| `src/instrumentations/pinecone/pinecone-instrumentation.ts` | ~640 | Main instrumentation |
| `src/registerInstrument.ts` (line 49, 232) | — | Registration |
| `src/llmFilterSpanProcessor.ts` (line 288-291, 313) | — | Pinecone spans pass LLM filter |


