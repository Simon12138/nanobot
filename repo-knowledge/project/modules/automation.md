---
id: module-automation
title: 模块：自动化与后台任务
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：自动化与后台任务

## 职责

在网关运行期间让 Agent 自主行动：定时任务（cron）、本地触发器（local trigger）与心跳检查（heartbeat），以及记忆整合任务 Dream。

## 三类自动化

| 类型 | 创建方式 | 适用场景 | 交付位置 |
| --- | --- | --- | --- |
| 定时任务 | 在目标会话中直接要求 nanobot 创建（时间/间隔/cron） | 提醒、日报、一次性未来任务 | 默认回到创建它的会话 |
| 本地触发器 | 目标会话发送 `/trigger <name>`，随后脚本执行 `nanobot trigger <id> [message]` | CI、脚本、webhook 事件 | 绑定的会话 |
| 心跳 | 编辑 `<workspace>/HEARTBEAT.md` | 安静的后台监控，仅在有事项时提醒 | 最近活跃会话 |

出处：`docs/automations.md:5-13,89-101`、`docs/concepts.md:137-145`。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/cron/types.py` | 任务/调度/负载/状态数据结构 |
| `nanobot/cron/service.py` | 任务加载、调度、执行、运行记录与审计 |
| `nanobot/cron/binding.py`、`bound_runner.py`、`session_delivery.py`、`session_turns.py` | 会话绑定、绑定执行、投递与自动化轮次 |
| `nanobot/triggers/` | 本地触发器存储、队列、投递与会话轮次 |
| `nanobot/agent/cron_turns.py`、`nanobot/agent/automation_turns.py` | 自动化轮次协调 |
| `nanobot/session/automation_spec.py` | 自动化元数据规范（隐藏历史标记） |
| `nanobot/agent/dream.py`、`nanobot/agent/memory.py` | Dream 整合 |
| `nanobot/cron/webui_metadata.py` | WebUI Automations 视图元数据 |

## 定时任务

- 存储：`<workspace>/cron/jobs.json`，`CronStore {version:1, jobs:[...]}`（`nanobot/cron/types.py:197-201`、`nanobot/cli/gateway_runtime.py:549-550`）。
- 调度：`at`（一次性时间戳）、`every`（固定间隔）、`cron`（表达式 + 可选 IANA 时区）（`nanobot/cron/types.py:23-44`）。
- 负载：`system_event` 或 `agent_turn`，含消息、投递字段、`sessionKey`、来源信息与绑定版本（`nanobot/cron/types.py:47-81`）。
- 未绑定会话的 Agent 任务会自动禁用（`nanobot/cron/service.py:196-251`）。
- 系统任务 Dream 与 Heartbeat 由网关启动时注册，并在 WebUI 中显示为只读受保护任务（`nanobot/cli/gateway_runtime.py:837-865`、`docs/automations.md:105`）。
- 运行状态保留最近 20 次；另有 `action.jsonl` 变更审计与 `runs/` 运行记录（`nanobot/cron/types.py:116-147`、`nanobot/cron/service.py:208-209,428`）。

## 本地触发器

- 先在目标会话 `/trigger <name>` 创建，返回形如 `trg_8K4P2Q9X` 的 ID 与命令（`docs/automations.md:64-80`）。
- 投递写入工作区本地持久队列；网关离线或目标会话忙时等待；每个工作区只应有一个消费者（`docs/cli-reference.md:229`）。
- 语义为**至少一次**：进程在运行中重启可能重放同一投递，外部脚本需容忍重复（`docs/automations.md:117`）。
- 每次投递在 `<workspace>/triggers/runs` 写审计记录（`docs/cli-reference.md:229`）。
- nanobot 不提供公网 webhook 服务；外部系统需自建接收器再调用 CLI（`docs/automations.md:82-84`）。

## 心跳

- 网关运行期间默认启用，间隔由 `gateway.heartbeat.interval_s`（默认 1800 秒）控制；仅在需要关注时发消息（`docs/automations.md:89-101`、`nanobot/config/schema.py:395-399`）。
- 内容来自 `<workspace>/HEARTBEAT.md`；模板说明其会被受保护的 heartbeat cron 任务读取（`nanobot/templates/HEARTBEAT.md`）。
- 支持自定义评估提示（`docs/configuration.md:2515-2518`）。

## 可靠性语义

| 场景 | 行为 | 出处 |
| --- | --- | --- |
| 网关停机错过定时点 | 不补跑；周期任务等下一次，过去的一次性任务不自动执行 | `docs/automations.md:111` |
| 精确墙钟时间 | 用 cron 表达式；间隔会在每次运行结束后重算 | `docs/automations.md:112` |
| 发送前通道不可达 | 自动重试最多 1 小时，不重新运行模型 | `docs/automations.md:113` |
| 发送中途超时 | 不自动重试，避免重复消息 | `docs/automations.md:113` |
| 本地触发器 | 持久队列 + 至少一次投递 | `docs/automations.md:117` |

## 配置

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `gateway.heartbeat.enabled` | `true` | 心跳开关 |
| `gateway.heartbeat.interval_s` | `1800` | 心跳间隔 |
| `agents.defaults.dream.enabled` | `true` | Dream 开关 |
| `agents.defaults.dream.intervalH` | `2` | Dream 间隔 |
| `agents.defaults.dream.cron` | `null` | 旧版 cron 覆盖 |

出处：`nanobot/config/schema.py:71-98,395-399`。

## 扩展点

- 新的自动化来源：实现自动化 spec（参考 `nanobot/session/automation_spec.py` 与 `nanobot/triggers/local_session_turns.py`），把隐藏元数据 `_automation_turn` 写入历史，供客户端识别。
- 新的 cron 负载类型：扩展 `CronPayload` 与绑定执行逻辑，注意保留未绑定自动禁用规则。

## 测试

- `tests/cron/`、`tests/triggers/`、`tests/agent/` 中的自动化轮次与恢复测试。

## 待人工确认的疑点

1. 定时任务的“创建会话”绑定在会话删除后的行为未在本条目固化，需结合 `cron/binding.py` 验证。
2. 心跳“最近活跃会话”的选择规则由运行时维护，跨通道的优先级需人工确认。

## 来源

- `docs/automations.md`
- `docs/concepts.md:137-145`
- `docs/cli-reference.md:219-250`
- `nanobot/cron/types.py:23-81,116-201`
- `nanobot/cron/service.py:196-251,208-209,428`
- `nanobot/cli/gateway_runtime.py:549-550,837-865`
- `nanobot/session/automation_spec.py`
- `nanobot/config/schema.py:71-98,395-399`
- `nanobot/templates/HEARTBEAT.md`
