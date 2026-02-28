# Tools & CLI Reference

## Table of Contents
- [Agent Tools](#agent-tools)
- [CLI Commands](#cli-commands)
- [Custom Commands](#custom-commands)
- [JSONL Session Distillation](#jsonl-session-distillation)

## Agent Tools

Source: `src/tools.ts`

All tools use `@sinclair/typebox` for parameter schemas and `stringEnum()` from `openclaw/plugin-sdk` for enum params.

### Core Tools (Always Enabled)

#### memory_recall
Search through long-term memories using hybrid retrieval.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | ✅ | — | Search query |
| `limit` | number | ❌ | 5 | Max results (1-20) |
| `scope` | string | ❌ | agent's accessible scopes | Specific scope to search |
| `category` | enum | ❌ | all | `preference/fact/decision/entity/other` |

Returns: List of memories with scores and source indicators (vector, BM25, reranked).

#### memory_store
Save information in long-term memory.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | ✅ | — | Information to remember |
| `importance` | number | ❌ | 0.7 | Score 0-1 |
| `category` | enum | ❌ | `other` | Category classification |
| `scope` | string | ❌ | agent's default scope | Target scope |

Pre-storage checks:
1. Scope access validation
2. Noise filter (`isNoise()`) — rejects greetings, boilerplate, meta-questions
3. Duplicate detection — cosine similarity > 0.98 = skip
4. Embedds via `embedPassage()`

#### memory_forget
Delete memories by ID or search query.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | ❌ | — | Search query to find memory |
| `memoryId` | string | ❌ | — | Direct memory ID (UUID or 8+ prefix) |
| `scope` | string | ❌ | all accessible | Scope filter |

Behavior:
- If `memoryId` provided → direct delete
- If `query` provided with 1 high-confidence match (>0.9) → auto-delete
- Otherwise → returns candidate list with short IDs

#### memory_update
Update an existing memory in-place (preserves original timestamp).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `memoryId` | string | ✅ | — | ID (UUID, prefix, or search query) |
| `text` | string | ❌ | — | New text (triggers re-embedding) |
| `importance` | number | ❌ | — | New importance score |
| `category` | enum | ❌ | — | New category |

Special behavior:
- If `memoryId` doesn't look like a UUID → treats as search query
- If text is updated, runs noise filter + re-embedding
- Returns candidate list if multiple matches found

### Management Tools (Optional, `enableManagementTools: true`)

#### memory_stats
Get statistics about memory usage, scopes, and categories.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `scope` | string | ❌ | all accessible | Scope filter |

Returns: total count, scope/category breakdowns, retrieval config, FTS status.

#### memory_list
List recent memories with optional filtering.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `limit` | number | ❌ | 10 | Max results (1-50) |
| `scope` | string | ❌ | all accessible | Scope filter |
| `category` | enum | ❌ | all | Category filter |
| `offset` | number | ❌ | 0 | Pagination offset |

## CLI Commands

Registered via `openclaw memory-pro <command>`. Source: `cli.ts`.

```bash
# List memories
openclaw memory-pro list [--scope global] [--category fact] [--limit 20] [--json]

# Search memories (uses full hybrid retrieval)
openclaw memory-pro search "query" [--scope global] [--limit 10] [--json]

# View statistics
openclaw memory-pro stats [--scope global] [--json]

# Delete a memory by ID (supports 8+ char prefix)
openclaw memory-pro delete <id>

# Bulk delete with filters
openclaw memory-pro delete-bulk --scope global [--before 2025-01-01] [--dry-run]

# Export (JSONL without vectors)
openclaw memory-pro export [--scope global] [--output memories.json]

# Import (re-embeds all entries)
openclaw memory-pro import memories.json [--scope global] [--dry-run]

# Re-embed from source DB to target DB (A/B testing)
openclaw memory-pro reembed --source-db /path/to/old-db [--batch-size 32] [--skip-existing]

# Migrate from built-in memory-lancedb
openclaw memory-pro migrate check [--source /path]
openclaw memory-pro migrate run [--source /path] [--dry-run] [--skip-existing]
openclaw memory-pro migrate verify [--source /path]

# Print plugin version
openclaw memory-pro version
```

### Re-embed Command Details

- Reads source DB → re-embeds all text with current model → writes to target (current) DB
- **Safety**: Refuses in-place re-embedding unless `--force` is passed
- Uses `importEntry()` which preserves original `id` and `timestamp`
- Batch processing with configurable batch size (default: 32)
- `--skip-existing`: checks `hasId()` before importing

### Migration (from legacy memory-lancedb)

Legacy paths searched:
- `~/.openclaw/memory/lancedb`
- `~/.claude/memory/lancedb`

Migration converts `createdAt` → `timestamp`, adds `scope` field, preserves vectors.

## Custom Commands

Custom slash commands are NOT built into the plugin. They are defined at the Agent/system-prompt level.

### Example: `/lesson` command
Add to agent system prompt:
```markdown
## /lesson command
When the user sends `/lesson <content>`:
1. Use memory_store to save as category=fact (the raw knowledge)
2. Use memory_store to save as category=decision (actionable takeaway)
3. Confirm what was saved
```

### Example: `/remember` command
```markdown
## /remember command
When the user sends `/remember <content>`:
1. Use memory_store to save with appropriate category and importance
2. Confirm with the stored memory ID
```

## JSONL Session Distillation

Two approaches for creating memories from session logs:

### Approach 1: `/new` Pipeline (Recommended, 2026-02+)

Non-blocking pipeline triggered by `/new` command:
1. `command:new` hook enqueues JSON task file (fast, no LLM calls)
2. User-level systemd worker watches inbox
3. Worker runs Gemini Map-Reduce on session JSONL
4. Produces 0-20 atomic lessons → imported via `openclaw memory-pro import`
5. Keywords include `Keywords (zh)` with entity taxonomy
6. Example files in `examples/new-session-distill/`

### Approach 2: Hourly Cron Distiller (Legacy)

Script: `scripts/jsonl_distill.py`

Workflow:
1. `python3 scripts/jsonl_distill.py init` — initialize cursor (mark existing files as read)
2. `python3 scripts/jsonl_distill.py run` — incremental read, produce batch JSON
3. Agent reads batch, calls `memory_store` for selected memories
4. `python3 scripts/jsonl_distill.py commit --batch-file <file>` — advance cursor

Key details:
- Cursor stored at `~/.openclaw/state/jsonl-distill/cursor.json`
- Batch files at `~/.openclaw/state/jsonl-distill/batches/`
- Skips `*.reset.*` files and distiller agent self (`memory-distiller`)
- `OPENCLAW_JSONL_DISTILL_ALLOWED_AGENT_IDS` env var for allowlist
- Safe: never modifies session logs
