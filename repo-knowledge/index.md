---
id: repo-knowledge-index
title: nanobot 知识库总索引
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# nanobot 知识库总索引

本知识库由「知识库运维」按“老项目知识初始化”流程生成，描述绑定仓库 `nanobot` 的当前代码事实。

| 项目 | 值 |
| --- | --- |
| 仓库 | `nanobot`（远程 `git@github.com:Simon12138/nanobot.git`） |
| 基线提交 | `c360a10e8ca8afd18916b73b7575f015941a33d9` |
| 代码版本 | `0.4.2`（`pyproject.toml`） |
| 初始化分支 | `knowledge/init-20261126` |
| 生成日期 | 2026-11-26 |

## 目录

| 目录 | 用途 | 索引 |
| --- | --- | --- |
| `project/` | 代码知识：概览、架构、模块、数据模型、运行手册、术语 | [project/index.md](project/index.md) |
| `history/` | 历史文档原件（只读输入） | [history/index.md](history/index.md) |
| `business-knowledge/` | 从历史文档提炼的业务流程、领域规则、术语 | [business-knowledge/index.md](business-knowledge/index.md) |
| `requirement-knowledge/` | 已确认工单的知识沉淀 | [requirement-knowledge/index.md](requirement-knowledge/index.md) |
| `_audit/` | 巡检报告与裁决记录 | [_audit/index.md](_audit/index.md) |
| `_templates/` | front-matter 与文档模板约定 | [_templates/index.md](_templates/index.md) |

## 文件清单

### project/

| 文件 | 内容 | 更新时间 | 来源 | 状态 | 置信度 |
| --- | --- | --- | --- | --- | --- |
| [index.md](project/index.md) | 项目知识索引与阅读路径 | 2026-11-26 | 代码与文档 | active | high |
| [overview.md](project/overview.md) | 定位、基本事实、运行形态、仓库结构、技术栈 | 2026-11-26 | `README.md`、`pyproject.toml` | active | high |
| [architecture.md](project/architecture.md) | 分层、核心链路、组合根、启动序列、设计约束 | 2026-11-26 | `docs/architecture.md`、核心代码 | active | high |
| [data-model.md](project/data-model.md) | 存储全景、SQLite schema、会话/记忆/cron/config 数据 | 2026-11-26 | `nanobot/session/*`、`nanobot/cron/*` | active | high |
| [runbook.md](project/runbook.md) | 安装、配置、运行、网关运维、部署、验证、排障 | 2026-11-26 | `docs/*`、部署资产 | active | high |
| [glossary.md](project/glossary.md) | 中英术语与本仓库标识符对照 | 2026-11-26 | `docs/*`、核心代码 | active | high |
| [modules/index.md](project/modules/index.md) | 模块知识索引 | 2026-11-26 | 代码与文档 | active | high |
| [modules/cli-gateway.md](project/modules/cli-gateway.md) | CLI 分派、网关生命周期、健康端点、容器启动 | 2026-11-26 | `nanobot/cli/*`、`nanobot/gateway/*` | active | high |
| [modules/agent-runtime.md](project/modules/agent-runtime.md) | Loop/Runner、上下文、子代理、目标、命令 | 2026-11-26 | `nanobot/agent/*` | active | high |
| [modules/providers.md](project/modules/providers.md) | Provider 注册/选择、配置、回退、Responses API | 2026-11-26 | `nanobot/providers/*` | active | high |
| [modules/channels.md](project/modules/channels.md) | 通道发现/生命周期、访问控制、消息流 | 2026-11-26 | `nanobot/channels/*`、`nanobot/bus/*` | active | high |
| [modules/tools-skills.md](project/modules/tools-skills.md) | 工具注册/执行、技能加载、MCP | 2026-11-26 | `nanobot/agent/tools/*` | active | high |
| [modules/sessions-memory.md](project/modules/sessions-memory.md) | 会话存储/恢复、长期记忆、Dream | 2026-11-26 | `nanobot/session/*`、`nanobot/agent/memory.py` | active | high |
| [modules/webui-tui.md](project/modules/webui-tui.md) | 浏览器工作台、终端客户端、端口与访问控制 | 2026-11-26 | `webui/*`、`tui/*`、`nanobot/webui/*` | active | high |
| [modules/automation.md](project/modules/automation.md) | cron、本地触发器、心跳、可靠性语义 | 2026-11-26 | `nanobot/cron/*`、`nanobot/triggers/*` | active | high |
| [modules/security.md](project/modules/security.md) | 工作区限制、Shell 沙箱、SSRF、访问控制 | 2026-11-26 | `nanobot/security/*`、`.agent/security.md` | active | high |
| [modules/update-release.md](project/modules/update-release.md) | 更新命令、备份/恢复、发布工作流 | 2026-11-26 | `nanobot/update*.py`、`.github/workflows/*` | active | medium |

### 其他目录

| 文件 | 内容 | 更新时间 | 状态 | 置信度 |
| --- | --- | --- | --- | --- |
| [history/index.md](history/index.md) | 历史文档索引（当前无历史文档） | 2026-11-26 | active | high |
| [business-knowledge/index.md](business-knowledge/index.md) | 业务知识索引（无历史输入，未生成） | 2026-11-26 | active | high |
| [requirement-knowledge/index.md](requirement-knowledge/index.md) | 工单知识索引（暂无条目） | 2026-11-26 | active | high |
| [_audit/index.md](_audit/index.md) | 巡检报告索引（暂无） | 2026-11-26 | active | high |
| [_templates/index.md](_templates/index.md) | front-matter、文档骨架与章节口径 | 2026-11-26 | active | high |

## 本次生成与跳过

| 项 | 结果 |
| --- | --- |
| `project/` 概览与架构 | 已生成 |
| `project/modules/<模块>` | 已生成 10 个模块 |
| `project/data-model` / `runbook` / `glossary` | 已生成 |
| `business-knowledge/` | 跳过：`repo-knowledge/history/` 无历史文档（`history/index.md`） |
| 工单沉淀、巡检 | 不在本次初始化范围 |
| 业务代码、部署脚本、CI | 未修改 |

## 已知疑点（待人工裁决/确认）

1. 绑定远端为 `Simon12138/nanobot` fork，`README`/badge 指向上游 `HKUDS/nanobot`；发布与同步关系未在仓库内说明。
2. 无历史文档，`business-knowledge/` 为空；是否以 `docs/` 产品文档补充业务知识需管理员决定。
3. Python 支持矩阵口径不一致：classifiers 为 3.11/3.12，CI 覆盖 3.11 与 3.14。
4. WebUI/网关仅保证同版本兼容，灰度与回滚的版本对齐流程未在仓库内定义。
5. 工单知识章节存在六节（运维职责）与五节（`knowledge-curate` 技能）的口径冲突。
6. `workspace-id` 与工作区路径的映射只存在本机，多实例迁移/共享方案未定义。

## 维护约定

- 只写 `repo-knowledge/**`；不修改业务代码、部署脚本、CI 与云资源。
- 写前查重、查矛盾、核对来源；未裁决不覆盖旧结论。
- 新增/改名/删除文件须在同一提交更新所在目录索引与根索引。
- 修订必须保留“原结论 / 修订原因 / 裁决来源”，旧文件置 `status: superseded`。
