<div align="center">

# 🧠 memory-lancedb-pro Skill

**专为 AI 编码助手设计的 [memory-lancedb-pro](https://github.com/win4r/memory-lancedb-pro) 插件维护技能**

让 AI 全方位理解插件架构、检索管线、配置体系，从而高效地维护和升级这个 OpenClaw 长期记忆插件。

[![OpenClaw Plugin](https://img.shields.io/badge/OpenClaw-Plugin-blue)](https://github.com/openclaw/openclaw)
[![LanceDB](https://img.shields.io/badge/LanceDB-Vectorstore-orange)](https://lancedb.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**简体中文** | [English](README_EN.md)

</div>

---

## 这是什么？

这是一个 **Agent Skill**（AI 编码助手技能包），专门为维护和升级 [memory-lancedb-pro](https://github.com/win4r/memory-lancedb-pro) OpenClaw 插件而设计。

当 AI 编码助手加载这个 Skill 后，它将获得对插件的全方位理解能力，包括：

- 🏗️ **插件架构** — 12 个源文件的职责、导出和相互关系
- 🔍 **检索管线** — RRF 融合、交叉编码器 Reranking、6 个评分阶段的精确数学公式
- 💾 **存储层** — LanceDB schema、FTS 索引、CRUD 操作实现
- 🔐 **作用域系统** — 5 种作用域类型、访问控制逻辑
- 🛠️ **开发工作流** — 7 个常见开发场景的步骤指南
- 🐛 **故障排除** — 安装、配置、检索质量调优、开发陷阱

## 文件结构

```
memory-lancedb-pro-skill/
├── SKILL.md                                  # 主技能文件（架构、工作流、设计决策）
├── references/
│   ├── retrieval_pipeline.md                 # 检索管线深度解析
│   ├── storage_and_schema.md                 # 存储层与数据模型
│   ├── embedding_system.md                   # 嵌入系统（提供商、缓存、Task-aware）
│   ├── plugin_lifecycle.md                   # 插件生命周期与配置
│   ├── scope_system.md                       # 多作用域隔离系统
│   ├── tools_and_cli.md                      # Agent 工具与 CLI 命令
│   └── troubleshooting.md                    # 常见问题与故障排除
├── README.md                                 # 本文件
└── README_EN.md                              # English README
```

## 如何使用

### 方式一：作为 Antigravity Agent Skill（推荐）

将整个文件夹放到 Antigravity 的 skills 目录下：

```bash
# 克隆到 Antigravity skills 目录
git clone https://github.com/win4r/memory-lancedb-pro-skill.git \
  ~/.gemini/antigravity/skills/memory-lancedb-pro
```

Skill 会在满足以下条件时自动触发：

1. 开发 memory-lancedb-pro 的新功能或修复 bug
2. 修改检索管线（向量搜索、BM25、RRF 融合、Reranking）
3. 添加或更换嵌入提供商
4. 更新作用域/访问控制逻辑
5. 修改 Agent 工具或 CLI 命令
6. 排查记忆质量问题（噪声、重复、低召回率）
7. 开发 JSONL 会话蒸馏管线
8. 在不同记忆后端之间迁移数据
9. 理解插件架构以规划改进

### 方式二：作为独立参考文档

直接阅读 `SKILL.md` 和 `references/` 目录下的文档，获取插件的完整技术细节。

## 涵盖的知识深度

| 领域 | 内容 |
|------|------|
| **检索管线** | RRF 融合公式、3 种 Rerank 提供商适配器、Recency Boost / Importance Weight / Length Norm / Time Decay / Hard Min / MMR 6 个评分阶段的精确公式 |
| **存储层** | LanceDB 表 schema、FTS 索引创建与竞态处理、向量/BM25 搜索实现、所有 CRUD 操作的 API 签名 |
| **嵌入系统** | 4 种提供商配置（Jina/OpenAI/Gemini/Ollama）、Task-aware API、LRU 缓存（256 条目，30 分 TTL）、模型维度映射表 |
| **插件生命周期** | 组件初始化顺序、3 个生命周期 Hook 的实现（auto-recall/auto-capture/session memory）、服务注册、日常备份 |
| **作用域系统** | 5 种作用域类型、默认 vs 显式访问控制、ScopeManager 完整 API |
| **工具与 CLI** | 6 个 Agent 工具参数表、所有 CLI 命令示例、JSONL 蒸馏的两种方案 |
| **故障排除** | 12 个常见问题的排查方法、检索质量调优旋钮、开发陷阱（Arrow Vector、配置不一致、环境变量时机） |

## 设计理念

本 Skill 遵循 **渐进式加载** 原则：

- **SKILL.md**（~10KB）作为总览和路由，始终加载
- **7 个 references 文件**（~45KB）按需加载，只在 AI 需要深入某个子系统时读取
- 总计约 55KB 的结构化技术文档，涵盖 1400+ 行精炼知识

## 许可证

MIT

