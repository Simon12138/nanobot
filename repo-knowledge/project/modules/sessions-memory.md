---
id: module-sessions-memory
title: 模块：会话与长期记忆
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：会话与长期记忆

## 职责

持久化对话历史（规范历史 + 展示历史）、提供恢复/派生/检索能力，并维护跨会话的长期记忆与 Dream 整合。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/session/manager.py` | 离线管理入口：定位存储、创建/保存/删除/fork、列表与元数据读取 |
| `nanobot/session/state.py` | 运行时状态所有者：单写队列、读写快照、reset/delete/fork、metadata 变更 |
| `nanobot/session/sqlite_store.py` | SQLite schema 与事务、迁移、乐观并发 |
| `nanobot/session/location.py` | 会话库定位（工作区外）与 `workspace-id` 管理 |
| `nanobot/session/history.py` | 对已持久化历史做全文检索（`search_sessions`/`read_session`） |
| `nanobot/session/display_history.py` | 展示历史条目种类 |
| `nanobot/session/recovery.py` | 通用恢复检查点 |
| `nanobot/webui/history.py`、`nanobot/session/recovery.py` | 旧 WebUI 历史导入与恢复 |
| `nanobot/session/webui_turns.py` | WebUI 轮次协调 |
| `nanobot/agent/memory.py` | 长期记忆、记忆日志、Git 存储 |
| `nanobot/agent/dream.py` | Dream 整合任务 |

## 会话存储

- 位置：`<config-dir>/sessions/<workspace-id>/sessions.sqlite3`，强制位于 Agent 工作区之外（`nanobot/session/location.py:26-54`）。
- `workspace-id` 为 32 位十六进制，保存在 `<workspace>/.nanobot/workspace-id`（`nanobot/session/location.py:19-21,48-54`）。
- Schema 与迁移细节见 `project/data-model.md`。

## 两套历史

| 历史 | 表 | 用途 |
| --- | --- | --- |
| 规范历史 | `sessions`、`messages`、`checkpoints`、`pending_inputs` | 模型重放、运行输入、压缩检查点 |
| 展示历史 | `history_heads`、`history_turns`、`history_entries` | WebUI/TUI 转录、分页、侧栏投影 |

- 展示历史不会进入 Provider 上下文或会话工具历史（`docs/architecture.md:182-184`）。
- 每次追加都携带执行前准入的 generation；缺失或被替换的会话会拒绝追加，而不是新建行（`docs/architecture.md:186`）。
- 删除/重置级联两套历史；fork 在单个 SQLite 事务内复制规范与展示前缀（`docs/architecture.md:186`）。

## 并发与恢复

- `SessionState` 每键 FIFO、单写执行器、读线程池；`save` 以 `(revision, generation)` 做乐观并发检查（`nanobot/session/state.py:40-93`、`nanobot/session/sqlite_store.py:420-437`）。
- `SessionManager` 的写操作要求离线（`_require_offline_mutation`），用于管理/CLI 场景（`nanobot/session/manager.py:473-477`）。
- WebUI 恢复与轮次协调在 `nanobot/webui/recovery.py`、`nanobot/session/webui_turns.py`；网关启动提供 WebUI 轮次策略（`.agent/design.md:13`）。
- 旧 JSONL 迁移是单次的，0.5.0 移除；导入前会校验重复键一致性与目标冲突（`nanobot/session/jsonl_migration.py`）。

## 长期记忆

| 文件 | 作用 |
| --- | --- |
| `SOUL.md`、`USER.md` | Agent 人格与用户画像（Agent 工作区） |
| `memory/MEMORY.md` | 长期记忆正文（模板含 User Information / Preferences / Project Context / Important Notes） |
| `memory/history.jsonl` | 记忆日志（cursor + timestamp + content + 可选 session_key） |
| `memory/.cursor` | 最新日志游标 |
| `memory/.dream_cursor` | Dream 已处理游标 |
| `memory/wiki/` | 可选的记忆 wiki 页面 |

- 记忆仓库由 `GitStore.for_memory(...)` 初始化并跟踪上述文件；`sync_workspace_templates` 创建模板与空日志（`nanobot/utils/helpers.py:794-833`、`nanobot/utils/gitstore.py:77-83`）。
- Dream：在 `dream_lock` 下读取未处理日志，以临时会话键运行整合，成功后推进游标、提交记忆 diff、压缩日志并清理 Dream 会话（`nanobot/agent/dream.py:24-67`）。
- 记忆日志普通追加无显式 fsync；整文件重写使用原子替换并 fsync 文件与目录（`.agent/gotchas.md:40-42`）。

## 会话生命周期操作

| 操作 | 入口 | 说明 |
| --- | --- | --- |
| 新建/获取 | `SessionState`/`SessionManager.get_or_create` | 键不存在时创建 |
| 重置 | `/new` 命令、`reset` | 新 generation，清理保留元数据 |
| 删除 | `delete_session` | 递归删除子会话，写 tombstone，级联删行 |
| 派生 | `fork_session_before_user_index` | 在第 N 条可见用户消息前 fork，单事务复制 |
| 检索 | `search_sessions`、`read_session` 工具 | 读取完整持久化历史，含摘要检查点之前的消息 |

出处：`nanobot/session/manager.py:541-579`、`nanobot/session/state.py:1019-1102,1166-1188`、`nanobot/agent/tools/sessions.py`。

## 配置

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `agents.defaults.sessionTtlMinutes` / `idleCompactAfterMinutes` | `15` | 空闲自动压缩阈值，0 关闭 |
| `agents.defaults.idleCompactCheckIntervalSeconds` | `60` | 空闲检查间隔 |
| `agents.defaults.dream.enabled` | `true` | Dream 开关 |
| `agents.defaults.dream.intervalH` | `2` | Dream 间隔小时 |
| `agents.defaults.dream.model` | `null` | Dream 模型覆盖 |

出处：`nanobot/config/schema.py:71-98,210-220`。

## 扩展点

- 恢复策略：通用检查点放 `session/recovery.py`，WebUI 专属放 `webui/recovery.py`（`.agent/design.md:13`）。
- 历史协议：新的展示事实应通过 `SessionState` 的 owner 事务写入，不要直接写文件日志（`docs/architecture.md:186`）。

## 测试

- `tests/session/`、`tests/agent/test_loop_recovery.py`、`tests/webui/`、`tests/test_documentation_build.py` 之外的记忆相关测试。

## 待人工确认的疑点

1. `workspace-id` 与工作区路径的映射仅存本机；多实例共享/迁移方案需人工确认。
2. 旧 WebUI 历史导入失败会阻止启动；历史损坏时的修复责任与流程未在仓库内定义。

## 来源

- `docs/architecture.md:178-199`
- `nanobot/session/manager.py:99-142,363-403,406-421,473-477,541-579`
- `nanobot/session/state.py:40-93,1019-1102,1166-1188`
- `nanobot/session/sqlite_store.py:149-159,174-175,270-313,420-437,726-749`
- `nanobot/session/location.py:19-54`
- `nanobot/session/history.py`、`nanobot/session/display_history.py:17-31`
- `nanobot/agent/memory.py:63-83,295-309,463-470,681-684`、`nanobot/agent/dream.py:24-67`
- `nanobot/utils/helpers.py:794-833`、`nanobot/utils/gitstore.py:77-83`
- `nanobot/config/schema.py:71-98,210-220`
- `.agent/design.md:13`、`.agent/gotchas.md:40-42`
