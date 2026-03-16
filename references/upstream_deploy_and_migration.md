# Upstream Deployment + Memory Migration Runbook

This runbook documents a safe cutover from a local/custom `memory-lancedb-pro` checkout
to an upstream checkout while preserving LanceDB memory data.

## Scope

- Service: `memory-lancedb-pro-mcp.service` (systemd user unit)
- Runtime: Node.js HTTP MCP server (`src/mcp/server-http.mjs`)
- Config: `~/.openclaw/openclaw.json`
- Data: LanceDB folder configured by `plugins.entries.memory-lancedb-pro.config.dbPath`

## Preconditions

1. Upstream repo is already cloned and on a clean branch.
2. Current service is healthy (`systemctl --user status memory-lancedb-pro-mcp.service`).
3. You have a rollback backup path ready.

## Paths (example)

- Old code path: `/home/austin/Development/research/memory-lancedb-pro`
- New upstream code path: `/home/austin/Development/research/memory-lancedb-pro-upstream`
- Old DB path: `/home/austin/.openclaw/workspace/memory/lancedb-pro`
- New DB path: `/home/austin/.openclaw/workspace/memory/lancedb-pro-upstream`

## Procedure

1. Stop service:
   - `systemctl --user stop memory-lancedb-pro-mcp.service`
2. Backup old DB:
   - `cp -a /home/austin/.openclaw/workspace/memory/lancedb-pro /home/austin/.openclaw/workspace/memory/lancedb-pro.backup-$(date +%Y%m%d-%H%M%S)`
3. Copy DB into new location:
   - `mkdir -p /home/austin/.openclaw/workspace/memory/lancedb-pro-upstream`
   - `cp -a /home/austin/.openclaw/workspace/memory/lancedb-pro/. /home/austin/.openclaw/workspace/memory/lancedb-pro-upstream/`
4. Update `~/.openclaw/openclaw.json`:
   - `plugins.load.paths[0]` -> upstream code path
   - `plugins.entries.memory-lancedb-pro.config.dbPath` -> new DB path
5. Update systemd user unit:
   - `WorkingDirectory` -> upstream code path
   - `ExecStart` -> upstream `src/mcp/server-http.mjs`
6. Reload and restart:
   - `systemctl --user daemon-reload`
   - `systemctl --user restart memory-lancedb-pro-mcp.service`

## Validation

1. Service status:
   - `systemctl --user status memory-lancedb-pro-mcp.service --no-pager`
2. Health endpoint:
   - `curl -fsS http://127.0.0.1:3099/health`
3. MCP initialize (stdio or HTTP adapter path in your client).
4. Functional smoke:
   - run one `memory_recall` query against known memory content
   - confirm returned IDs and text are from migrated dataset

## Rollback

1. Stop service.
2. Restore previous systemd unit (`WorkingDirectory`/`ExecStart`).
3. Restore `openclaw.json` old plugin path and old `dbPath`.
4. Restart service and verify health.

