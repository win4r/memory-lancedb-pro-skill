# Storage & Data Model

## Table of Contents
- [Database Schema](#database-schema)
- [LanceDB Initialization](#lancedb-initialization)
- [FTS Index](#fts-index)
- [Vector Search](#vector-search)
- [BM25 Search](#bm25-search)
- [CRUD Operations](#crud-operations)
- [Important Implementation Details](#important-implementation-details)

## Database Schema

LanceDB table name: `memories`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID v4) | Primary key, generated via `crypto.randomUUID()` |
| `text` | `string` | Memory text content (FTS indexed) |
| `vector` | `float[]` | Embedding vector (dimension matches model config) |
| `category` | `string` | One of: `preference`, `fact`, `decision`, `entity`, `other` |
| `scope` | `string` | Scope identifier (e.g., `global`, `agent:main`, `custom:work`) |
| `importance` | `float` | Score 0–1 (default: 0.7) |
| `timestamp` | `int64` | Creation timestamp in milliseconds (UTC) |
| `metadata` | `string` | JSON string for extensible metadata |

## LanceDB Initialization

Source: `MemoryStore.doInitialize()` in `src/store.ts`

1. Dynamic import: `@lancedb/lancedb` loaded via `loadLanceDB()` singleton promise
2. Connect to DB: `lancedb.connect(dbPath)`
3. Table init: idempotent — tries `openTable("memories")` first, creates only if missing
4. Race handling: If `createTable` fails with "already exists", falls back to `openTable`
5. Schema seed: Creates with dummy `__schema__` entry, then deletes it
6. Vector dimension validation: Checks existing data dimensions match config
7. FTS index creation: Calls `createFtsIndex()` — graceful fallback if unavailable

**Critical**: `doInitialize()` is guarded by `initPromise` to prevent concurrent initialization.

## FTS Index

```typescript
// Check existing indices
const indices = await table.listIndices();
const hasFtsIndex = indices?.some(idx => idx.indexType === "FTS" || idx.columns?.includes("text"));

// Create if missing (LanceDB >= 0.26)
await table.createIndex("text", { config: lancedb.Index.fts() });
```

- `ftsIndexCreated` flag tracks whether FTS is available
- If FTS index creation fails, `hasFtsSupport` returns `false` and retriever falls back to vector-only
- `bm25Search()` returns empty array `[]` when FTS is unavailable

## Vector Search

```typescript
async vectorSearch(vector: number[], limit = 5, minScore = 0.3, scopeFilter?: string[]): Promise<MemorySearchResult[]>
```

- Over-fetches: `min(limit * 10, 200)` rows to handle scope filtering
- Score conversion: `score = 1 / (1 + distance)` (LanceDB returns cosine distance)
- Double-checks scope filter in application layer (defense-in-depth)
- Returns entries sorted by score descending
- `scopeFilter` query: `(scope = 'x' OR scope = 'y') OR scope IS NULL` (NULL for backward compat)

## BM25 Search

```typescript
async bm25Search(query: string, limit = 5, scopeFilter?: string[]): Promise<MemorySearchResult[]>
```

- Uses `table.search(query, "fts")` explicitly specifying FTS query type
- BM25 score normalization: `1 / (1 + exp(-rawScore / 5))` — sigmoid to map unbounded BM25 to [0, 1]
- Raw scores < 0 treated as 0.5 (neutral)
- Catches errors gracefully, returns `[]` on failure

## CRUD Operations

### Store
```typescript
async store(entry: Omit<MemoryEntry, "id" | "timestamp">): Promise<MemoryEntry>
```
- Auto-generates UUID and timestamp
- Defaults `metadata` to `"{}"`

### Import Entry
```typescript
async importEntry(entry: MemoryEntry): Promise<MemoryEntry>
```
- Preserves original `id` and `timestamp`
- Used for re-embedding, migration, A/B testing
- Validates vector dimensions match

### Has ID
```typescript
async hasId(id: string): Promise<boolean>
```
- Simple existence check via SQL WHERE

### Delete
```typescript
async delete(id: string, scopeFilter?: string[]): Promise<boolean>
```
- Supports full UUID or 8+ hex char prefix
- Prefix match: scans up to 1000 entries, errors if ambiguous (>1 match)
- Validates scope permissions before deletion

### Update
```typescript
async update(id: string, updates: {...}, scopeFilter?: string[]): Promise<MemoryEntry | null>
```
- **Implemented as delete + re-add** (LanceDB has no in-place update)
- Preserves original timestamp
- Supports ID prefix matching (same as delete)
- Returns null if not found

### Bulk Delete
```typescript
async bulkDelete(scopeFilter: string[], beforeTimestamp?: number): Promise<number>
```
- Requires at least scope OR timestamp filter (safety guard)
- Counts first, then deletes

### List
```typescript
async list(scopeFilter?, category?, limit = 20, offset = 0): Promise<MemoryEntry[]>
```
- Fetches ALL matching rows (no SQL limit), sorts by timestamp descending in app layer
- **Vectors excluded** from list results for performance (set to `[]`)
- Pagination via `slice(offset, offset + limit)`

### Stats
```typescript
async stats(scopeFilter?): Promise<{ totalCount, scopeCounts, categoryCounts }>
```
- Fetches only `scope` and `category` columns
- Aggregates counts in application layer

## Important Implementation Details

### SQL Injection Prevention
- `escapeSqlLiteral()` escapes single quotes by doubling them
- All user-provided strings go through this function before SQL interpolation

### Arrow Vector Objects
- LanceDB returns Arrow Vector objects, NOT plain JS arrays
- `Array.isArray()` returns `false` for Arrow Vectors
- Use `.length` to check dimensions
- Use `Array.from(vector as Iterable<number>)` for conversion to plain arrays

### Scope NULL Handling
- Legacy data may have `NULL` scope — treated as `"global"`
- SQL queries include `OR scope IS NULL` for backward compatibility
- Application layer defaults `null/undefined` scope to `"global"`

### Database Path
- Default: `~/.openclaw/memory/lancedb-pro`
- Resolved via `api.resolvePath()` in plugin registration
- Can be overridden via `dbPath` config

### Backup System
- Daily JSONL export to `<dbPath>/../backups/`
- File format: `memory-backup-YYYY-MM-DD.jsonl`
- Keeps last 7 backups, auto-purges older ones
- First backup: 60 seconds after startup
- Vectors are NOT included in backups (only text metadata)
