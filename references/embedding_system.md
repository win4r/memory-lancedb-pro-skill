# Embedding System

## Table of Contents
- [Architecture](#architecture)
- [Provider Configuration](#provider-configuration)
- [Task-Aware Embeddings](#task-aware-embeddings)
- [LRU Cache](#lru-cache)
- [Known Model Dimensions](#known-model-dimensions)
- [Error Handling](#error-handling)

## Architecture

Source: `src/embedder.ts`

The `Embedder` class wraps the OpenAI SDK to provide:
- Task-aware embedding (separate `embedQuery` vs `embedPassage`)
- LRU cache with TTL
- Batch embedding support
- Any OpenAI-compatible endpoint

```typescript
class Embedder {
  private client: OpenAI;           // OpenAI SDK instance
  private readonly _cache: EmbeddingCache;  // LRU with TTL
  private readonly _model: string;
  private readonly _taskQuery?: string;     // e.g. "retrieval.query"
  private readonly _taskPassage?: string;   // e.g. "retrieval.passage"
  private readonly _normalized?: boolean;
  private readonly _requestDimensions?: number;
}
```

## Provider Configuration

The plugin works with **any OpenAI-compatible embedding API**:

| Provider | Model | Base URL | Dimensions | Notes |
|----------|-------|----------|------------|-------|
| Jina (recommended) | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 | Supports `task` + `normalized` |
| OpenAI | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 | — |
| Google Gemini | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 | — |
| Ollama (local) | `nomic-embed-text` | `http://localhost:11434/v1` | varies | Set `embedding.dimensions` explicitly |

### Config Example (Jina)
```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  }
}
```

## Task-Aware Embeddings

Some providers (e.g., Jina v5) support task-specific embeddings. The plugin uses:

- `embedQuery(text)` → uses `taskQuery` (e.g., `"retrieval.query"`) for search queries
- `embedPassage(text)` → uses `taskPassage` (e.g., `"retrieval.passage"`) for stored documents

**Backward compatibility**: `embed()` and `embedBatch()` methods delegate to `embedPassage()`/`embedBatchPassage()`.

### How tasks are passed

```typescript
private buildPayload(input: string | string[], task?: string): any {
  const payload: any = { model: this.model, input };
  if (task) payload.task = task;                    // Extra field
  if (this._normalized !== undefined) payload.normalized = this._normalized;  // Extra field
  if (this._requestDimensions > 0) payload.dimensions = this._requestDimensions;
  return payload;
}
```

The `task` and `normalized` fields are NOT in the OpenAI SDK types, hence the `any` cast. Providers that don't recognize these fields will ignore them.

## LRU Cache

```typescript
class EmbeddingCache {
  maxSize = 256;       // Max entries
  ttlMs = 30 * 60000;  // 30 minutes TTL
}
```

- Cache key: SHA-256 hash of `"${task}:${text}"` (first 24 hex chars)
- LRU eviction: on `get()`, entry is moved to end (most recently used)
- On `set()`: oldest entry evicted if cache is full
- Cache stats: `{ size, hits, misses, hitRate }` accessible via `embedder.cacheStats`

**When to clear cache**: Restart the gateway. Cache is in-memory only.

## Known Model Dimensions

Hardcoded lookup in `EMBEDDING_DIMENSIONS`:

```typescript
const EMBEDDING_DIMENSIONS: Record<string, number> = {
  "text-embedding-3-small": 1536,
  "text-embedding-3-large": 3072,
  "text-embedding-004": 768,
  "gemini-embedding-001": 3072,
  "nomic-embed-text": 768,
  "mxbai-embed-large": 1024,
  "BAAI/bge-m3": 1024,
  "all-MiniLM-L6-v2": 384,
  "all-mpnet-base-v2": 768,
  "jina-embeddings-v5-text-small": 1024,
  "jina-embeddings-v5-text-nano": 768,
};
```

If `embedding.dimensions` is set in config, it overrides this lookup.
If model is not in the table AND no dimensions override, initialization throws an error.

### Adding a New Model

Add to `EMBEDDING_DIMENSIONS` map:
```typescript
"new-model-name": 1024,
```

Or let users override via config: `"embedding": { "dimensions": 1024 }`.

## Error Handling

- Empty text throws `"Cannot embed empty text"`
- API failure wraps error with descriptive message
- Dimension mismatch validated after first successful call
- `test()` method embeds the string `"test"` to verify connectivity
- Env var resolution: `${VAR}` in API key is resolved at construction time

### Vector Dimension Mismatch

If existing LanceDB table has vectors of dimension X but config specifies dimension Y:
```
Error: Vector dimension mismatch: table=X, config=Y.
Create a new table/dbPath or set matching embedding.dimensions.
```

Solution: Either change `embedding.dimensions` / `embedding.model` to match, OR use a new `dbPath` and re-embed via CLI (`openclaw memory-pro reembed`).
