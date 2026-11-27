---
id: module-agent-runtime
title: 模块：Agent 运行时
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：Agent 运行时

## 职责

把一条入站消息变成一次完整对话：确定会话与工作区、构建上下文、驱动 Provider 与工具循环、处理流式与取消、投递答复，并提供子代理与目标执行能力。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/agent/loop.py` | `AgentLoop`：通道侧编排、会话/工作区解析、上下文、钩子、命令路由、出站投递 |
| `nanobot/agent/runner.py` | `AgentRunner`：Provider 调用、流式增量、工具执行、迭代上限、上下文治理 |
| `nanobot/agent/context.py` | `ContextBuilder`：项目指令 + Agent 身份 + 记忆 + 技能段落 |
| `nanobot/agent/context_governance.py` | 历史总结、预算裁减与模型请求状态 |
| `nanobot/agent/autocompact.py` | 空闲会话自动压缩 |
| `nanobot/agent/pending_inputs.py` | 运行内输入批次与终止等待 |
| `nanobot/agent/session_inbox.py` | 会话执行串行化与关闭语义 |
| `nanobot/agent/execution_policy.py` | 执行范围与取消策略 |
| `nanobot/agent/turn_delivery.py` | 轮次出站路由与投递 |
| `nanobot/agent/subagent.py` | 子代理管理、并发限制、状态 |
| `nanobot/agent/goal_permission.py`、`nanobot/goal/` | 持续目标的权限与执行 |
| `nanobot/command/` | 斜杠命令路由与内置命令 |

## 一轮对话的处理顺序（概述）

1. `AgentLoop` 消费 `MessageBus` 入站消息，解析 `session_key`、工作区作用域与模型运行时（`nanobot/agent/loop.py:1383-1394` 起）。
2. 通过 `ContextBuilder` 组装系统提示与历史：项目 `AGENTS.md`、Agent 工作区 `SOUL.md`/`USER.md`、记忆、技能摘要（`nanobot/agent/context.py`、`docs/architecture.md:166-176`）。
3. 交给 `AgentRunner`：调用 Provider，处理文本/推理流，执行工具调用，把结果回填模型；达到迭代上限后停止并给出运行时提示（`nanobot/agent/runner.py`、`docs/architecture.md:58`）。
4. 生成 `OutboundMessage`，经 `TurnDelivery` 与 `MessageBus` 交给通道发送（`nanobot/agent/turn_delivery.py`、`nanobot/channels/manager.py:902-1005`）。

## 上下文构建

- Agent 工作区负责：会话命名空间、`SOUL.md`、`USER.md`、记忆与自定义技能。
- 项目工作区负责：项目 `AGENTS.md`、工具相对路径与 Shell 工作目录。
- `ContextBuilder` 合并项目指令与 Agent 侧身份/记忆；文件与搜索工具以项目为边界，并对内置/Agent 技能与历史文件提供明确只读能力（`docs/architecture.md:166-176`）。
- 项目 `AGENTS.md` 若与内置默认相同会被跳过；不实现逐级祖先目录指令链（`.agent/gotchas.md:28`）。

## 上下文预算

- 当文件读取使请求超预算时：先总结模型已接收的历史；仍不满足时，仅把必要的未读文本结果替换为“读取更小范围”的错误并保存用于重放（`docs/architecture.md:60-62`）。
- 空闲自动压缩阈值由 `agents.defaults.session_ttl_minutes`（别名 `idleCompactAfterMinutes`）与检查间隔控制（`nanobot/config/schema.py:210-219`）。

## 工具执行

- 工具调用参数在注册表中按 JSON Schema 校验（不做类型强制转换），错误以可操作的工具错误返回给模型（`docs/architecture.md:145-147`、`nanobot/agent/tools/registry.py`）。
- 工具结果长度上限 `max_tool_result_chars`（默认 16000）用于控制上下文（`nanobot/config/schema.py:193`）。
- 工具清单与扩展方式见 `project/modules/tools-skills.md`。

## 子代理与目标

- 子代理由父会话派生，受 `max_concurrent_subagents`（默认 4）限制；后台子任务回报到发起会话，网关重启时未完成子任务标记为中断（`nanobot/config/schema.py:192`、`nanobot/agent/subagent.py`、`docs/concepts.md:125-135`）。
- Goal 通过 `create_goal`/`update_goal` 工具与 `/goal` 命令管理，状态保存在会话 metadata（`nanobot/goal/`、`nanobot/session/goal_state.py`）。

## 命令与取消

- 内置命令：`/stop`、`/restart`、`/status`、`/new`、`/compact`、`/model`、`/trigger`、`/dream*`、`/skill`、`/help`、`/pairing`，以及目标相关 `/goal`、`/cancel_goal`（`nanobot/command/builtin.py:39-87,543-571`、`nanobot/goal/command.py:108-113`）。
- `/stop` 停止当前会话及其子代理，不影响其他会话，也不撤销已发生的文件修改（`docs/concepts.md:130`）。

## 关键配置

| 配置 | 默认 | 作用 |
| --- | --- | --- |
| `agents.defaults.model` | `anthropic/claude-opus-4-5` | 默认模型 |
| `agents.defaults.provider` | `auto` | Provider 选择/自动检测 |
| `agents.defaults.max_tool_iterations` | `200` | 单轮工具迭代上限 |
| `agents.defaults.max_concurrent_subagents` | `4` | 子代理并发 |
| `agents.defaults.max_tool_result_chars` | `16000` | 工具结果字符上限 |
| `agents.defaults.context_window_tokens` | `200000` | 上下文窗口 |
| `agents.defaults.provider_retry_mode` | `standard` | 重试模式（standard/persistent） |
| `agents.defaults.reasoning_effort` | `null` | 推理强度，空表示沿用 Provider 默认 |

出处：`nanobot/config/schema.py:177-247`。

## 扩展点

- 新工具：`nanobot/agent/tools/`（见 tools-skills 模块）。
- 新钩子：实现 `AgentHook`，在组合根传入 `hook_factories`（`nanobot/nanobot.py:136-141`）。
- 新命令：在 `nanobot/command/builtin.py` 注册。

## 测试

- `tests/agent/`（循环并发、取消、投递、上下文构建、技能加载、恢复）、`tests/command/`、`tests/tools/`。

## 待人工确认的疑点

1. `provider_retry_mode="persistent"` 的退避上限与终止条件未在本条目展开，需按线上故障策略确认。
2. 模型预设与 fallback 的优先级细节见 providers 模块；跨 Provider 回退是否保留工具状态需专项验证。

## 来源

- `docs/architecture.md:36-62,145-147,166-199`
- `docs/concepts.md:94-135`
- `nanobot/agent/loop.py`、`nanobot/agent/runner.py`
- `nanobot/agent/context.py`、`nanobot/agent/context_governance.py`、`nanobot/agent/autocompact.py`
- `nanobot/agent/pending_inputs.py`、`nanobot/agent/session_inbox.py`、`nanobot/agent/execution_policy.py`
- `nanobot/agent/turn_delivery.py`、`nanobot/agent/subagent.py`
- `nanobot/config/schema.py:177-247`
- `nanobot/command/builtin.py:39-87,543-571`
- `.agent/gotchas.md:24-30`
