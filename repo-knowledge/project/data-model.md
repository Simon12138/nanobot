---
id: project-data-model
title: 数据模型与持久化
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 数据模型与持久化

## 存储全景

| 数据 | 位置 | 形式 | 出处 |
| --- | --- | --- | --- |
| 实例配置 | `<config-dir>/config.json`（默认 `~/.nanobot/config.json`） | JSON，camelCase 序列化 | `nanobot/config/loader.py:29-163` |
| 会话历史 | `<config-dir>/sessions/<workspace-id>/sessions.sqlite3` | SQLite（WAL） | `nanobot/session/sqlite_store.py:174-175`、`nanobot/session/location.py:26-54` |
| 工作区标识 | `<workspace>/.nanobot/workspace-id`（32 位十六进制） | 文本 | `nanobot/session/location.py:19-21,48-54` |
| 长期记忆 | `<workspace>/memory/MEMORY.md`、`memory/history.jsonl`、`memory/.cursor`、`memory/.dream_cursor` | Markdown + JSONL | `nanobot/agent/memory.py:63-83` |
| 记忆 Git 仓库 | `SOUL.md`、`USER.md`、`memory/MEMORY.md`、`memory/wiki`、`memory/.dream_cursor` | Git 跟踪 | `nanobot/utils/gitstore.py:77-83` |
| 定时任务 | `<workspace>/cron/jobs.json` | JSON（`version: 1`） | `nanobot/cron/types.py:197-201`、`nanobot/cli/gateway_runtime.py:549-550` |
| 本地触发器 | `<workspace>/triggers/`（投递队列与 `runs/` 审计） | 文件队列 | `docs/cli-reference.md:229` |
| WebUI 状态与旧历史 | `<config-dir>/webui/` | JSON/JSONL/其它 | `nanobot/config/paths.py:45-47`、`docs/cli-reference.md:133-143` |
| 日志 / 媒体 | `<config-dir>/logs/`、`<config-dir>/media/[<channel>]` | 文件 | `nanobot/config/paths.py:29-47` |

配置目录默认是 `config.json` 所在目录（`nanobot/config/paths.py:19-26`）。会话库被强制放在 agent 工作区之外，否则启动报错（`nanobot/session/location.py:34-38`）。

## SQLite 会话库

### 迁移机制

- 版本保存在 `PRAGMA user_version`；迁移函数按顺序组成 `_SCHEMA_MIGRATIONS`（当前 v1–v6），`initialize()` 应用剩余迁移并写回版本号（`nanobot/session/sqlite_store.py:149-159,270-313`）。
- 每个连接启用 `foreign_keys`、WAL、`synchronous=FULL`；写事务使用 `BEGIN IMMEDIATE`；数据库文件 chmod 0600 并拒绝符号链接（`nanobot/session/sqlite_store.py:219-232,242-313`）。

### 表结构（按迁移版本）

| 版本 | 表 / 索引 | 关键列 |
| --- | --- | --- |
| v1 | `storage_meta` | `key` PK、`value` |
| v1 | `sessions` | `key` PK、`created_at`、`updated_at`、`metadata`、`provider_state`、`last_archived`、`revision`、`generation`、`title`、`preview`、`visible_updated_at` |
| v1 | `messages` | `session_key` FK→`sessions(key)`、`position`、`record`；PK(`session_key`,`position`) |
| v1 | `checkpoints` | `session_key` PK/FK、`record` |
| v1 | `pending_inputs` | `session_key` FK、`kind`、`position`、`record`；PK(`session_key`,`kind`,`position`) |
| v1 | `sessions_updated` | `sessions(updated_at DESC)` |
| v2 | `history_heads` | `session_key` PK/FK、`revision`、`updated_at_ms`、`preview`、`user_preview` |
| v2 | `history_turns` | `session_key` FK、`ordinal`、`record_count`、`byte_count`、`user_count`、`completed`；PK(`session_key`,`ordinal`) |
| v2 | `history_entries` | `session_key`、`turn_ordinal`、`position`、`kind`、`record`；PK 三元组；FK 级联到 `history_turns` |
| v3 → v4 | `session_archives` | `key` PK/FK→`sessions(key)`（延迟约束）、`archived_at` |
| v5 | `history_input_identity` | 唯一部分索引：`json_extract(record,'$.content.input_id') WHERE kind='input'` |
| v6 | `sessions_subagent_*` | 子代理来源索引，基于 `metadata.$.subagent_task` |

以上定义见 `nanobot/session/sqlite_store.py:62-145`。

### 两套逻辑历史

- `sessions` + `messages` + `checkpoints` + `pending_inputs` 是模型重放用的规范历史与运行输入。
- `history_heads` + `history_turns` + `history_entries` 是展示历史：`kind` 包含 `input`、`answer`、`answer_segment`、`reasoning*`、`progress`、`tool_activity`、`file_activity`、`completion`、`fork_boundary`、`compaction`、`model_request`（`nanobot/session/display_history.py:17-31`）。展示事实不进入 Provider 上下文或会话工具历史（`docs/architecture.md:180-184`）。
- 一个 turn 在 `COMPLETION` 记录处闭合（`nanobot/session/sqlite_store.py:709`）。

## 会话身份与并发

- 会话键默认 `f"{channel}:{chat_id}"`，可由 `session_key_override` 覆盖（`nanobot/bus/events.py:45-48`）；通道可追加线程维度（例如 Telegram topic）。
- 内部键：`heartbeat`、`dream:` 前缀（`nanobot/session/keys.py:3-13`）。
- `Session` 携带 `revision:int` 与 `generation:uuid4().hex`；`last_consolidated` 是 `last_archived` 的别名（`nanobot/session/manager.py:99-142`）。
- `save` 以 `(revision, generation)` 做乐观并发检查，不匹配抛 `SessionConflictError`；generation 同时保护历史/检查点写入（`nanobot/session/sqlite_store.py:420-437,453-454,681-683`）。

## 删除、重置、派生

| 操作 | 语义 | 出处 |
| --- | --- | --- |
| `delete` | 递归删除 `parent_session_key` 子会话；写入 tombstone；存储层级联删除行 | `nanobot/session/state.py:1055-1102`、`nanobot/session/sqlite_store.py:778-784` |
| `reset` | 构造新 `Session`（新 generation），清理保留/WebUI 元数据，先删再提交 | `nanobot/session/state.py:1166-1188` |
| `fork` | 复制第 N 条可见用户消息之前的公共消息，剔除易变元数据，在单事务内复制 `history_*` 前缀并写 `FORK_BOUNDARY` | `nanobot/session/manager.py:363-403`、`nanobot/session/state.py:1019-1052`、`nanobot/session/sqlite_store.py:726-749` |

## 旧 JSONL 迁移

- 一次性迁移从 0.3.6 起可用，计划在 0.5.0 移除（`nanobot/session/jsonl_migration.py:1-6,29-31`）。
- 来源：会话库目录下 `*.jsonl`、`<workspace>/sessions/*.jsonl`，以及可选 `<name>.checkpoint.json`（`nanobot/session/jsonl_migration.py:131-165`）。
- 行格式：`_type="metadata"` 头、`_type="provider_state"`、其余为带 `role` 的消息（`nanobot/session/jsonl_migration.py:98-114`）。
- 规则：重复会话键内容必须逐字节一致，否则中止；目标 SQLite 已存在同键则中止；成功后写 `storage_meta['jsonl_import']='complete'`，工作区来源备份到 `<db dir>/jsonl-backup/` 后删除（`nanobot/session/sqlite_store.py:298-310`、`nanobot/session/jsonl_migration.py:169-177,233-289`）。
- `nanobot/session/history.py` 不是存储格式，而是对已持久化历史做全文检索的 `SessionHistoryReader`（`nanobot/session/history.py:157-368`）。

## 长期记忆

- `memory/history.jsonl` 记录形如 `{"cursor": int, "timestamp": "%Y-%m-%d %H:%M", "content": str, "session_key"?: str}`；`.cursor` 保存最新游标，`.dream_cursor` 保存已处理游标（`nanobot/agent/memory.py:295-309,463-470`）。
- 普通追加在锁内完成但无显式 fsync；整文件重写走原子替换并 fsync 文件与目录（`.agent/gotchas.md:40-42`）。
- Dream 在 `dream_lock` 下读取未处理日志，用临时会话键 `dream:<ts>-<hex8>` 运行，完成后推进游标、自动提交记忆 diff、压缩日志并清理 Dream 会话（`nanobot/agent/dream.py:24-67`、`nanobot/agent/memory.py:681-684`）。
- 记忆模板包含 User Information / Preferences / Project Context / Important Notes（`nanobot/templates/memory/MEMORY.md`）。

## 定时任务数据

- `CronStore { version: 1, jobs: [...] }`（`nanobot/cron/types.py:197-201`）。
- Job 字段：`id`、`name`、`enabled`、`schedule`、`payload`、`state`、`createdAtMs`、`updatedAtMs`、`deleteAfterRun`（`nanobot/cron/types.py:150-194`）。
- `CronSchedule.kind ∈ {at, every, cron}`；`CronPayload.kind ∈ {system_event, agent_turn}`，含消息、投递字段、`sessionKey`、来源信息与 `bindingVersion`（`nanobot/cron/types.py:23-81`）。
- `CronJobState` 记录下次/上次运行与状态（`ok|error|skipped`），运行历史上限 20 条（`nanobot/cron/types.py:116-147`、`nanobot/cron/service.py:191`）；另有 `action.jsonl` 审计与 `runs/` 运行记录（`nanobot/cron/service.py:208-209,428`）。

## 配置数据

- 根配置分节：`agents`、`channels`、`transcription`、`providers`、`network`、`gateway`、`tools`、`modelPresets`（`nanobot/config/schema.py:498-514`）。
- 同时接受 camelCase 与 snake_case，保存时使用 camelCase 别名（`nanobot/config_base.py:12-15`、`docs/architecture.md:164`）。
- 环境变量前缀 `NANOBOT_`、嵌套分隔符 `__`（`nanobot/config/schema.py:764-767`）；配置值支持 `${VAR}` 引用（`nanobot/config/loader.py:182-231`）。
- 保存为原子替换，并剔除 OAuth 凭据（`nanobot/config/loader.py:134-163`）。

## 目标与自动化元数据

- Goal 状态保存在会话 metadata 的 `goal_state` 键，含 `status`（active/blocked/cancelled）、`objective`、`ui_summary`（`nanobot/session/goal_state.py:7-47`）。
- 自动化轮次通过隐藏历史元数据 `_automation_turn` 标记，cron 与本地触发器各自提供 spec（`nanobot/session/automation_spec.py:7-49`、`nanobot/cron/session_turns.py:25-38`、`nanobot/triggers/local_session_turns.py:27-37`）。

## 数据安全与运维注意

- 会话库、配置与工作区应一起备份；仅复制运行中的 `sessions.sqlite3` 可能漏掉最近写入（`docs/cli-reference.md:143`）。
- 升级/降级受数据格式约束：旧版本无法读取新格式 WebUI 历史（`docs/cli-reference.md:143`）。
- 跨实例共享同一配置目录不受支持，更新前需停止所有相关进程（`docs/cli-reference.md:66`）。

## 待人工确认的疑点

1. `workspace-id` 与工作区路径的映射关系只存在于本机文件中，仓库内没有对照表；多工作区/多实例场景的运维约定需人工确认。
2. 旧 JSONL 迁移窗口（0.5.0 移除）与实际安装版本分布是否已覆盖，需发布侧确认。

## 来源

- `nanobot/session/sqlite_store.py:62-159,174-175,219-313,420-437,726-749,778-784`
- `nanobot/session/state.py:40-93,1019-1102,1166-1188`
- `nanobot/session/manager.py:99-142,363-403`
- `nanobot/session/location.py:19-54`
- `nanobot/session/display_history.py:17-31`
- `nanobot/session/jsonl_migration.py:1-6,29-31,98-114,131-165,169-177,233-289`
- `nanobot/session/history.py:157-368`
- `nanobot/agent/memory.py:63-83,295-309,463-470,681-684`
- `nanobot/agent/dream.py:24-67`
- `nanobot/cron/types.py:23-81,116-147,150-201`
- `nanobot/cron/service.py:191,208-209,428`
- `nanobot/config/schema.py:498-514,764-767`
- `nanobot/config/paths.py:19-47`
- `nanobot/session/goal_state.py:7-47`
- `docs/cli-reference.md:133-143`、`docs/architecture.md:166-199`
