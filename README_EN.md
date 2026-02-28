<div align="center">

# 🧠 memory-lancedb-pro Skill

**An AI Coding Assistant Skill for maintaining and upgrading the [memory-lancedb-pro](https://github.com/win4r/memory-lancedb-pro) plugin**

Give your AI assistant deep understanding of the plugin's architecture, retrieval pipeline, and configuration system — enabling efficient maintenance and feature development of this OpenClaw long-term memory plugin.

[![OpenClaw Plugin](https://img.shields.io/badge/OpenClaw-Plugin-blue)](https://github.com/openclaw/openclaw)
[![LanceDB](https://img.shields.io/badge/LanceDB-Vectorstore-orange)](https://lancedb.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[简体中文](README.md) | **English**

</div>

---

## What Is This?

This is an **Agent Skill** — a structured knowledge package designed for AI coding assistants to maintain and upgrade the [memory-lancedb-pro](https://github.com/win4r/memory-lancedb-pro) OpenClaw plugin.

When an AI coding assistant loads this skill, it gains comprehensive understanding of the plugin, including:

- 🏗️ **Plugin Architecture** — Responsibilities, exports, and relationships of all 12 source files
- 🔍 **Retrieval Pipeline** — RRF fusion, cross-encoder reranking, exact math formulas for 6 scoring stages
- 💾 **Storage Layer** — LanceDB schema, FTS indexing, CRUD operation implementations
- 🔐 **Scope System** — 5 scope types, access control logic
- 🛠️ **Development Workflows** — Step-by-step guides for 7 common development scenarios
- 🐛 **Troubleshooting** — Installation, configuration, retrieval quality tuning, development pitfalls

## File Structure

```
memory-lancedb-pro-skill/
├── SKILL.md                                  # Main skill file (architecture, workflows, design decisions)
├── references/
│   ├── retrieval_pipeline.md                 # Retrieval pipeline deep dive
│   ├── storage_and_schema.md                 # Storage layer & data model
│   ├── embedding_system.md                   # Embedding system (providers, caching, task-aware)
│   ├── plugin_lifecycle.md                   # Plugin lifecycle & configuration
│   ├── scope_system.md                       # Multi-scope isolation system
│   ├── tools_and_cli.md                      # Agent tools & CLI commands
│   └── troubleshooting.md                    # Common issues & troubleshooting
├── README.md                                 # 中文 README
└── README_EN.md                              # This file
```

## How to Use

### Option A: As an Antigravity Agent Skill (Recommended)

Clone this repo into the Antigravity skills directory:

```bash
git clone https://github.com/win4r/memory-lancedb-pro-skill.git \
  ~/.gemini/antigravity/skills/memory-lancedb-pro
```

The skill auto-triggers when you work on:

1. Developing new features or fixing bugs in memory-lancedb-pro
2. Modifying the retrieval pipeline (vector search, BM25, RRF fusion, reranking, scoring stages)
3. Adding or changing embedding providers
4. Updating scope/access control logic
5. Modifying agent tools or CLI commands
6. Troubleshooting memory quality issues (noise, duplicates, low recall)
7. Working on the JSONL session distillation pipeline
8. Migrating data between memory backends
9. Understanding the plugin's architecture to plan enhancements

### Option B: As Standalone Reference Documentation

Read `SKILL.md` and the files under `references/` directly for complete technical details about the plugin.

## Knowledge Coverage

| Domain | Coverage |
|--------|----------|
| **Retrieval Pipeline** | RRF fusion formula, 3 rerank provider adapters, exact formulas for Recency Boost / Importance Weight / Length Norm / Time Decay / Hard Min / MMR scoring stages |
| **Storage Layer** | LanceDB table schema, FTS index creation with race condition handling, vector/BM25 search impl, full CRUD API signatures |
| **Embedding System** | 4 provider configs (Jina/OpenAI/Gemini/Ollama), task-aware API, LRU cache (256 entries, 30min TTL), model dimension lookup table |
| **Plugin Lifecycle** | Component init order, 3 lifecycle hook implementations (auto-recall/auto-capture/session memory), service registration, daily backup |
| **Scope System** | 5 scope types, default vs explicit access control, complete ScopeManager API |
| **Tools & CLI** | 6 agent tool parameter tables, all CLI command examples, 2 JSONL distillation approaches |
| **Troubleshooting** | 12 common issues with solutions, retrieval quality tuning knobs, development pitfalls (Arrow Vectors, config inconsistencies, env var timing) |

## Design Philosophy

This skill follows the **progressive disclosure** principle:

- **SKILL.md** (~10KB) serves as overview and router — always loaded
- **7 reference files** (~45KB) loaded on-demand — only when the AI needs a deep dive into a specific subsystem
- Total: ~55KB of structured technical documentation covering 1,400+ lines of distilled knowledge

## License

MIT

