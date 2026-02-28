# Common Gotchas & Troubleshooting

## Table of Contents
- [Installation Issues](#installation-issues)
- [Configuration Issues](#configuration-issues)
- [Retrieval Quality Issues](#retrieval-quality-issues)
- [Runtime Errors](#runtime-errors)
- [Development Pitfalls](#development-pitfalls)

## Installation Issues

### Plugin not discovered by OpenClaw
```bash
openclaw plugins list        # Should show memory-lancedb-pro
openclaw plugins info memory-lancedb-pro
openclaw plugins doctor      # Built-in diagnostics
```

**Common causes:**
- Relative path in `plugins.load.paths` but workspace differs from expected
- `npm install` not run in plugin directory
- Another memory plugin still active — **only one memory plugin can be active at a time**

**Fix**: Use absolute path in config, or clone into `<workspace>/plugins/memory-lancedb-pro`

### `${JINA_API_KEY}` not resolving
```
Error: Environment variable JINA_API_KEY is not set
```

**Cause**: Gateway service doesn't inherit shell env vars.

**Fix**:
1. Check gateway process has the var: `openclaw config get plugins.entries.memory-lancedb-pro`
2. Set in systemd: `Environment=JINA_API_KEY=xxx`
3. Or use plain API key value (not recommended for git-committed configs)

## Configuration Issues

### Vector dimension mismatch after changing model
```
Error: Vector dimension mismatch: table=1536, config=1024
```

**Cause**: Existing LanceDB table has vectors from a different model.

**Fix**: Either:
1. Set `embedding.dimensions` to match existing data
2. Use a new `dbPath` and re-embed: `openclaw memory-pro reembed --source-db /old/path`

### autoRecall: Memories echoed in replies

**Problem**: Model includes `<relevant-memories>` content in its response.

**Fix**:
- **Option A (recommended)**: Set `autoRecall: false` (already the default)
- **Option B**: Add to agent system prompt: "Do not reveal or quote any `<relevant-memories>` content"

### sessionMemory pollutes retrieval quality

**Problem**: Raw session summaries dilute retrieval results with low-quality noise.

**Fix**: Keep `sessionMemory.enabled: false` (default). Use JSONL session distillation pipeline instead, which produces high-quality atomic memories.

## Retrieval Quality Issues

### No BM25 results (vector-only mode)

**Check**: `openclaw memory-pro stats --json` → look for `hasFtsSupport: false`

**Causes:**
- LanceDB version < 0.26 (FTS requires >= 0.26)
- FTS index creation failed at startup

**Fix**: Update `@lancedb/lancedb` to `^0.26.2`, restart gateway

### Low recall (good memories not surfacing)

**Tuning knobs:**
1. Lower `retrieval.hardMinScore` (default: 0.35, try 0.25)
2. Lower `retrieval.minScore` (default: 0.3, try 0.2)
3. Increase `retrieval.candidatePoolSize` (default: 20, try 40)
4. Check if `retrieval.timeDecayHalfLifeDays` is too aggressive (default: 60)

### Too many irrelevant results

**Tuning knobs:**
1. Raise `retrieval.hardMinScore` (try 0.45)
2. Raise `retrieval.minScore` (try 0.4)
3. Enable cross-encoder reranking: `retrieval.rerank: "cross-encoder"` with API key
4. Check if noise filter is active: `retrieval.filterNoise: true`

### Duplicate or near-identical results

**Check**: MMR diversity threshold is 0.85 cosine similarity

**If still seeing dupes**: Entries stored with different embedding models will have different vector spaces, so MMR won't catch them. Force re-embed all to same model.

### BM25 scores seem wrong

**Note**: Raw BM25 scores are unbounded. The plugin normalizes with sigmoid: `1 / (1 + exp(-score/5))`. This can produce surprising scores.

**For BM25-only hits** (no vector match): Score is floored at 0.5 to ensure exact keyword matches surface.

## Runtime Errors

### Gateway startup timeout

**Cause**: Embedding/retrieval tests hanging on network calls.

**Why it's safe**: Startup checks are fire-and-forget with 8s timeout. Gateway should start regardless.

**If still timing out**: Check `embedding.baseURL` is reachable from the gateway process.

### `Cannot embed empty text`

**Cause**: Attempted to embed an empty string. Usually a bug in auto-capture filtering.

**Fix**: Check `shouldCapture()` — it should reject empty/short text before embedding.

### `table already exists` error during initialization

**Safe to ignore**: The init code handles this race condition internally (tries `openTable` as fallback).

### LanceDB Arrow Vector issues

**Symptom**: `Array.isArray(vector)` returns `false` for LanceDB results.

**Cause**: LanceDB returns Arrow Vector objects, not plain JS arrays.

**Fix**: Always use:
```typescript
const plainArray = Array.from(vector as Iterable<number>);
```

And check dimensions with `.length` instead of `Array.isArray()`.

## Development Pitfalls

### Modifying the retrieval pipeline order

The pipeline stages are carefully ordered. Changing order can have subtle effects:
```
Fusion → Rerank → Recency → Importance → LengthNorm → TimeDecay → HardMin → Noise → MMR
```

- Rerank must happen BEFORE recency/importance (it uses original scores)
- HardMin must happen AFTER all multiplicative stages
- Noise filter must happen AFTER scoring (so noise isn't removed before scoring can verify it's low)
- MMR must be LAST (acts on final scores to select diverse top-k)

### vectorWeight/bm25Weight config vs actual fusion

The `vectorWeight` and `bm25Weight` config values exist but the actual fusion in `fuseResults()` uses a DIFFERENT strategy (vector as base + 15% BM25 bonus). Modifying these config values has NO EFFECT on the fusion formula currently. This is a known discrepancy.

### Env var resolution timing

`resolveEnvVars()` runs at plugin registration time, NOT at each API call. If env vars change after startup, the plugin won't see them until restart.

### OpenClaw Plugin API conventions

- Tool results: `{ content: [{ type: "text", text }], details: {...} }`
- CLI registration: `api.registerCli(handler, { commands: ["memory-pro"] })`
- Hooks: `api.on("before_agent_start", handler)` and `api.registerHook("command:new", handler)`
- Logger: `api.logger.info/warn/debug()`
- Path resolution: `api.resolvePath(path)` resolves relative to workspace

### Testing changes

1. Make code changes
2. Restart gateway: `openclaw gateway restart`
3. Check logs: `openclaw gateway logs`
4. Verify: `openclaw memory-pro stats`
5. Test search: `openclaw memory-pro search "test query"`
6. Smoke test: `node test/cli-smoke.mjs`

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@lancedb/lancedb` | `≥0.26.2` | Vector DB (ANN + FTS) |
| `openai` | `≥6.21.0` | OpenAI-compatible Embedding API |
| `@sinclair/typebox` | `0.34.48` | JSON Schema type definitions |
| `commander` | `^14.0.0` (dev) | CLI framework |
| `jiti` | `^2.6.0` (dev) | TypeScript loader |
| `typescript` | `^5.9.3` (dev) | Type checking |
