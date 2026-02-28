# Scope System

## Table of Contents
- [Scope Types](#scope-types)
- [Access Control Logic](#access-control-logic)
- [ScopeManager API](#scopemanager-api)
- [Configuration](#configuration)
- [Utility Functions](#utility-functions)

## Scope Types

Source: `src/scopes.ts`

Built-in scope patterns:

| Pattern | Example | Description |
|---------|---------|-------------|
| `global` | `global` | Shared knowledge across all agents |
| `agent:<id>` | `agent:discord-bot` | Agent-private scope |
| `custom:<name>` | `custom:work` | User-defined scope |
| `project:<id>` | `project:myapp` | Project-specific scope |
| `user:<id>` | `user:john` | User-specific scope |

Scope format validation: `^[a-zA-Z0-9._:-]+$`, max 100 chars.

## Access Control Logic

### Default Behavior (no explicit config)

When no `agentAccess` is configured for an agent:
- Agent gets access to: `["global", "agent:<agentId>"]`
- Agent scope is only included if it exists as a built-in scope pattern

### Explicit Access Control

When `agentAccess` is configured:
- Agent gets ONLY the explicitly listed scopes
- Example: `"discord-bot": ["global", "agent:discord-bot", "custom:shared"]`

### Default Scope for New Memories

When an agent stores a memory without specifying scope:
1. If agent has access to its own `agent:<id>` scope → uses that
2. Otherwise → uses the global default scope (configured via `scopes.default`, defaults to `"global"`)

### No Agent Context

When no `agentId` is available (e.g., CLI operations):
- `getAccessibleScopes()` → returns ALL defined scopes
- `getDefaultScope()` → returns config default (`"global"`)
- `isAccessible()` → returns `true` for any valid scope

## ScopeManager API

```typescript
interface ScopeManager {
  getAccessibleScopes(agentId?: string): string[];
  getDefaultScope(agentId?: string): string;
  isAccessible(scope: string, agentId?: string): boolean;
  validateScope(scope: string): boolean;
  getAllScopes(): string[];
  getScopeDefinition(scope: string): ScopeDefinition | undefined;
}
```

### Management Methods

```typescript
class MemoryScopeManager {
  addScopeDefinition(scope, definition): void;
  removeScopeDefinition(scope): boolean;  // Cannot remove "global"
  setAgentAccess(agentId, scopes): void;
  removeAgentAccess(agentId): boolean;
  exportConfig(): ScopeConfig;
  importConfig(config): void;
  getStats(): { totalScopes, agentsWithCustomAccess, scopesByType };
}
```

### Validation

On initialization, `validateConfiguration()`:
1. Verifies default scope exists in definitions
2. Warns (but doesn't error) if agent access references undefined scopes
3. Global scope is always ensured to exist

## Configuration

```json
{
  "scopes": {
    "default": "global",
    "definitions": {
      "global": { "description": "Shared knowledge across all agents" },
      "agent:discord-bot": { "description": "Discord bot private memories" },
      "custom:work": { "description": "Work-related memories" }
    },
    "agentAccess": {
      "discord-bot": ["global", "agent:discord-bot"],
      "code-agent": ["global", "agent:code-agent", "custom:work"]
    }
  }
}
```

## Utility Functions

```typescript
// Create scope identifiers
createAgentScope("main")     // "agent:main"
createCustomScope("work")    // "custom:work"
createProjectScope("myapp")  // "project:myapp"
createUserScope("john")      // "user:john"

// Parse scope identifier
parseScopeId("agent:main")   // { type: "agent", id: "main" }
parseScopeId("global")       // { type: "global", id: "" }

// Access checks
isScopeAccessible("global", ["global", "agent:main"])  // true
filterScopesForAgent(scopes, agentId, scopeManager)     // filtered array
```

## How Scopes Flow Through the System

1. **Storage**: `memory_store` tool → validates scope access → stores with scope field
2. **Retrieval**: `memory_recall` / auto-recall → gets accessible scopes → passes as `scopeFilter`
3. **Deletion**: `memory_forget` → validates scope access → checks before delete
4. **CLI**: No agent context → all scopes accessible (admin mode)
5. **Auto-capture**: Uses default scope for the current agent
6. **BM25/Vector search**: Scope filter applied via SQL WHERE clause
