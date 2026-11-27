---
id: project-index
title: 项目知识索引
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 项目知识索引

本目录记录 nanobot 的代码事实：定位、架构、模块、数据模型、运行手册与术语。基线为 `c360a10e8ca8afd18916b73b7575f015941a33d9`。

| 文档 | 文件 | 内容 | 更新时间 | 来源 | 状态 | 置信度 |
| --- | --- | --- | --- | --- | --- | --- |
| 项目概览 | [overview.md](overview.md) | 定位、基本事实、运行形态、仓库结构、技术栈 | 2026-11-26 | `README.md`、`pyproject.toml`、`docs/concepts.md` | active | high |
| 系统架构 | [architecture.md](architecture.md) | 分层、核心链路、组合根、启动序列、并发与设计约束 | 2026-11-26 | `docs/architecture.md`、核心代码 | active | high |
| 数据模型 | [data-model.md](data-model.md) | 存储全景、SQLite schema、会话语义、记忆/cron/config 数据 | 2026-11-26 | `nanobot/session/*`、`nanobot/cron/*` | active | high |
| 运行手册 | [runbook.md](runbook.md) | 安装、配置、运行、网关运维、容器部署、开发验证、故障排查 | 2026-11-26 | `docs/*`、部署资产 | active | high |
| 术语表 | [glossary.md](glossary.md) | 中英术语与本仓库标识符对照 | 2026-11-26 | `docs/*`、核心代码 | active | high |
| 模块知识 | [modules/index.md](modules/index.md) | 10 个模块的职责、关键文件、配置、扩展点与测试 | 2026-11-26 | 代码与文档 | active | high |

## 阅读路径

- 新人入门：`overview.md` → `glossary.md` → `architecture.md`。
- 排障：`runbook.md` → 对应模块文档。
- 改数据结构/迁移：`data-model.md` → `modules/sessions-memory.md`。
- 接入模型或平台：`modules/providers.md` / `modules/channels.md`。

## 边界

- 本目录只记录事实，不包含部署决策或未裁决的推断；不确定内容在各文档“待人工确认的疑点”中列出。
- 代码基线变化后需重新核对条目并更新 `updated`；修订保留原结论与裁决来源。
