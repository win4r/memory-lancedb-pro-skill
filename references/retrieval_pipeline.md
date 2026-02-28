# Retrieval Pipeline Deep Dive

## Table of Contents
- [Pipeline Overview](#pipeline-overview)
- [Hybrid vs Vector-Only Mode](#hybrid-vs-vector-only-mode)
- [RRF Fusion Strategy](#rrf-fusion-strategy)
- [Cross-Encoder Reranking](#cross-encoder-reranking)
- [Scoring Stages](#scoring-stages)
- [MMR Diversity Filter](#mmr-diversity-filter)
- [Noise Filtering](#noise-filtering)
- [Adaptive Retrieval](#adaptive-retrieval)
- [Configuration Reference](#configuration-reference)

## Pipeline Overview

Full hybrid retrieval pipeline (in order):

```
Query → embedQuery() ─┐
                       ├─→ RRF Fusion → Rerank → Recency Boost → Importance Weight
Query → BM25 FTS ─────┘    → Length Norm → Time Decay → Hard Min Score → Noise Filter → MMR
```

Source: `src/retriever.ts` → `MemoryRetriever.hybridRetrieval()`

Both `hybridRetrieval()` and `vectorOnlyRetrieval()` share the same post-fusion pipeline.

## Hybrid vs Vector-Only Mode

- `mode: "hybrid"` (default): Uses both vector search and BM25 full-text search
- `mode: "vector"`: Falls back to vector-only when FTS is unavailable or explicitly disabled
- Auto-fallback: If `store.hasFtsSupport === false`, hybrid mode silently degrades to vector-only

## RRF Fusion Strategy

**NOT traditional RRF.** Uses vector score as base, BM25 as confirmatory bonus.

```typescript
// If entry appears in both vector AND BM25 results:
fusedScore = vectorScore + (1 * 0.15 * vectorScore)  // 15% boost

// If entry appears ONLY in BM25 (keyword exact match, weak semantic):
fusedScore = max(bm25NormalizedScore, 0.5)

// If entry appears ONLY in vector results:
fusedScore = vectorScore
```

**Rationale**: BM25 hit confirms relevance (keyword match). BM25-only results get floor of 0.5 so exact keyword matches (e.g., "JINA_API_KEY") aren't buried by vector distance.

Source: `fuseResults()` in `src/retriever.ts`

## Cross-Encoder Reranking

Supports 3 provider formats via `rerankProvider`:

| Provider | Auth Header | Documents Format | Response Path |
|----------|-------------|-----------------|---------------|
| `jina` (default) | `Authorization: Bearer` | `string[]` | `results[].relevance_score` |
| `siliconflow` | Same as jina | Same | Same |
| `pinecone` | `Api-Key` | `[{text}]` | `data[].score` |

**Blending formula**: `60% cross-encoder score + 40% original fused score`

**Timeout**: 5 seconds via `AbortController`. On timeout/failure, falls back to cosine similarity reranking.

**Cosine fallback**: `70% original score + 30% cosine(queryVector, entryVector)`

**Unreturned candidates** (when reranker returns fewer results than input): kept with `0.8×` their original score.

Source: `rerankResults()` in `src/retriever.ts`

### Adding a New Rerank Provider

1. Add name to `RerankProvider` type union
2. Add case in `buildRerankRequest()`:
   ```typescript
   case "newprovider":
     return {
       headers: { "Content-Type": "application/json", "Authorization": `Bearer ${apiKey}` },
       body: { model, query, documents, top_n: topN },
     };
   ```
3. Add case in `parseRerankResponse()`:
   ```typescript
   case "newprovider": {
     const items = data.results as Array<{ index: number; score: number }>;
     return items?.map(r => ({ index: r.index, score: r.score })) ?? null;
   }
   ```
4. Add to `rerankProvider` enum in `openclaw.plugin.json`

## Scoring Stages

All stages use `clamp01()` to keep scores in [0, 1] range.

### 1. Recency Boost (additive)
```
boost = exp(-ageDays / halfLife) * weight
score = score + boost
```
- Default: `halfLife=14 days`, `weight=0.10`
- Effect: 0 days → +0.10, 14 days → +0.05, 28 days → +0.025
- Set `recencyHalfLifeDays: 0` to disable

### 2. Importance Weight (multiplicative)
```
factor = 0.7 + 0.3 * importance
score *= factor
```
- `importance=1.0` → ×1.0 (no change)
- `importance=0.7` → ×0.91
- `importance=0.5` → ×0.85
- `importance=0.0` → ×0.7

### 3. Length Normalization (multiplicative)
```
logRatio = log2(max(charLen / anchor, 1))
factor = 1 / (1 + 0.5 * logRatio)
score *= factor
```
- Default anchor: 500 chars
- At anchor (500) → 1.0, 800 → ~0.75, 1000 → ~0.67, 2000 → ~0.50
- No penalty for entries shorter than anchor
- Set `lengthNormAnchor: 0` to disable

### 4. Time Decay (multiplicative)
```
factor = 0.5 + 0.5 * exp(-ageDays / halfLife)
score *= factor
```
- Default: `halfLife=60 days`
- At 0 days → 1.0×, 60 days → ~0.68×, 120 days → ~0.59×, 240 days → ~0.52×
- **Floor at 0.5×** — old entries never lose more than half their score
- Set `timeDecayHalfLifeDays: 0` to disable

### 5. Hard Minimum Score
```
if (score < hardMinScore) → discard
```
- Default: `0.35`
- Applied AFTER all scoring stages

## MMR Diversity Filter

Greedy selection to prevent near-duplicate results:

```typescript
for each candidate (sorted by score desc):
  if any selected result has cosine(selected.vector, candidate.vector) > 0.85:
    defer candidate (appended at end)
  else:
    select candidate
```

- Threshold: 0.85 cosine similarity
- Deferred items are NOT removed — just deprioritized to end of list
- Handles LanceDB Arrow Vector objects via `Array.from()` conversion
- Source: `applyMMRDiversity()` in `src/retriever.ts`

## Noise Filtering

Post-retrieval noise removal via `filterNoise()` from `src/noise-filter.ts`:

**Filtered categories:**
- Agent denial responses: "I don't have any information", "I'm not sure about", etc.
- Meta-questions about memory: "do you remember", "can you recall", etc.
- Session boilerplate: "hi", "hello", "HEARTBEAT", "fresh session"
- Very short text (< 5 chars)

Applied at BOTH:
- Retrieval time (in scoring pipeline)
- Storage time (in `memory_store` tool via `isNoise()`)

## Adaptive Retrieval

`shouldSkipRetrieval()` in `src/adaptive-retrieval.ts` determines if a query needs memory lookup.

**Skipped (returns true):**
- Greetings: "hi", "hello", "good morning"
- Slash commands: anything starting with `/`
- Shell commands: "run", "build", "test", "git", "npm", etc.
- Simple affirmations: "yes", "no", "ok", "sure", etc.
- Continuations: "go ahead", "continue", "proceed"
- Pure emoji
- System: "HEARTBEAT", "[System"
- Very short non-question messages (< 15 chars English, < 6 chars CJK)

**Forced retrieve (returns false, overrides skip):**
- Memory keywords: "remember", "recall", "forgot", "memory"
- Temporal references: "last time", "before", "previously"
- Personal info: "my name", "my email", "my preference"
- CJK equivalents: "你记得", "之前", "上次"

**Normalization**: Strips OpenClaw `[cron:...]` prefix and `Conversation info (untrusted metadata):` header before pattern matching.

## Configuration Reference

```typescript
export const DEFAULT_RETRIEVAL_CONFIG: RetrievalConfig = {
  mode: "hybrid",
  vectorWeight: 0.7,        // (used in docs, but RRF fusion uses different strategy)
  bm25Weight: 0.3,          // (used in docs, but RRF fusion uses 15% boost)
  minScore: 0.3,            // Pre-rerank minimum score
  rerank: "cross-encoder",  // "cross-encoder" | "lightweight" | "none"
  candidatePoolSize: 20,    // Candidates fetched before fusion
  recencyHalfLifeDays: 14,
  recencyWeight: 0.10,
  filterNoise: true,
  rerankModel: "jina-reranker-v3",
  rerankEndpoint: "https://api.jina.ai/v1/rerank",
  lengthNormAnchor: 500,
  hardMinScore: 0.35,
  timeDecayHalfLifeDays: 60,
};
```

> **Note**: `vectorWeight` and `bm25Weight` are declared in config but the actual fusion in `fuseResults()` uses a different strategy (vector as base + 15% BM25 bonus). These fields exist for future configurability or documentation purposes.
