---
id: project-glossary
title: 术语表
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 术语表

| 术语 | 英文 / 标识符 | 定义 | 出处 |
| --- | --- | --- | --- |
| Agent 循环 | AgentLoop | 面向通道的一轮对话编排者：解析会话/工作区、构建上下文、装配钩子、发布出站消息 | `nanobot/agent/loop.py`、`docs/architecture.md:38-45` |
| Agent 运行器 | AgentRunner | 面向模型的多轮循环：调用 Provider、处理流式增量、执行工具并回填结果 | `nanobot/agent/runner.py`、`docs/architecture.md:46-52` |
| 会话 | Session | 一段对话的持久化历史，键为 `channel:chat_id`，带 revision/generation | `nanobot/session/manager.py:99-119` |
| 轮次 | Turn | 一次输入到完成（`COMPLETION`）的展示历史单元 | `nanobot/session/sqlite_store.py:709`、`nanobot/session/display_history.py:17-31` |
| 会话代际 | generation | 会话的随机身份（`uuid4().hex`），用于拒绝过期写入 | `nanobot/session/manager.py:119` |
| 修订号 | revision | 乐观并发版本号，与 generation 一起校验保存 | `nanobot/session/sqlite_store.py:420-437` |
| 通道 | Channel | 外部平台适配器，产出/消费入站与出站消息 | `nanobot/channels/base.py`、`docs/architecture.md:87-100` |
| 通道插件 | ChannelPlugin | 通道包的描述符：运行时类、设置页、依赖、默认启用等 | `nanobot/channels/plugin.py:21-40` |
| 消息总线 | MessageBus | 入站/出站消息与运行时事件的队列与分发 | `nanobot/bus/queue.py`、`nanobot/bus/events.py` |
| 入站/出站消息 | InboundMessage / OutboundMessage | 通道与核心之间的标准消息结构 | `nanobot/bus/events.py:29-82` |
| 模型提供方 | Provider | 大模型后端适配器，统一实现 `LLMProvider.chat()` | `nanobot/providers/base.py`、`docs/architecture.md:64-86` |
| 模型预设 | Model Preset | 命名的 provider + model + 生成参数组合，用于快速切换 | `nanobot/config/schema.py:156-174` |
| 回退模型 | Fallback | 主模型失败时按配置切换的候选模型 | `nanobot/providers/fallback_provider.py:113-131` |
| 工具 | Tool | 模型可调用的能力单元，带名称、描述、JSON Schema 参数与执行体 | `nanobot/agent/tools/base.py:75-159` |
| 工具注册表 | ToolRegistry | 按名称保存工具、生成 schema、准备与执行调用 | `nanobot/agent/tools/registry.py:29-224` |
| MCP | Model Context Protocol | 外部工具服务协议，支持 stdio/sse/streamableHttp 与 OAuth | `nanobot/config/schema.py:411-424`、`.agent/security.md:21` |
| 技能 | Skill | 以 `SKILL.md`（YAML front-matter）组织的按需说明，可用 `$skill-name` 调用 | `nanobot/agent/skills.py:80-159` |
| Agent 插件 | Agent Plugin | 打包技能和/或 MCP 服务器的可安装单元 | `docs/concepts.md:104-115` |
| CLI 应用 | CLI App | 由 Apps 管理的本地命令行工具适配器 | `nanobot/agent/tools/cli_apps.py`、`docs/concepts.md:104-115` |
| 工作区 | Workspace | Agent 的持久状态目录，默认 `~/.nanobot/workspace/` | `nanobot/config/paths.py:50-53` |
| Agent 工作区 | Agent workspace | 承载 SOUL/USER、记忆与自定义技能的身份工作区 | `docs/concepts.md:41-52` |
| 项目工作区 | Project workspace | WebUI 中为某个话题选择的项目目录，提供 AGENTS.md 与相对路径基准 | `docs/concepts.md:41-52` |
| 身份文件 | SOUL.md / USER.md | Agent 人格与用户画像，位于 Agent 工作区 | `docs/architecture.md:192-198` |
| 项目指令 | AGENTS.md | 项目级指令文件，来自项目工作区 | `docs/architecture.md:166-176` |
| 心跳文件 | HEARTBEAT.md | 心跳任务读取的检查清单，仅在需要时打扰用户 | `docs/concepts.md:137-145` |
| 长期记忆 | Memory | 工作区内的 MEMORY.md、记忆日志与 wiki 页面 | `docs/memory.md`、`nanobot/agent/memory.py` |
| 记忆整合 | Dream | 周期性读取未处理日志并更新长期记忆的后台任务 | `nanobot/agent/dream.py:24-67` |
| 压缩 | Compaction | 在上下文预算内总结/截断历史；含空闲自动压缩 | `nanobot/agent/autocompact.py`、`nanobot/agent/context_governance.py` |
| 网关 | Gateway | 常驻进程：通道、cron、Dream、heartbeat、健康端点、WebUI 后端 | `docs/cli-reference.md:177-217` |
| 健康端点 | `/health` | 网关就绪检查，返回 200/503 JSON | `nanobot/cli/gateway_runtime.py:780-836` |
| WebUI | WebUI | React 浏览器工作台，由 WebSocket 通道在 8765 提供 | `docs/architecture.md:102-125` |
| TUI | TUI | TypeScript + Bun + OpenTUI 的原生终端客户端 | `tui/package.json`、`docs/cli-reference.md:117-157` |
| 自动化 | Automation | 定时 cron、本地 trigger、heartbeat 三类自主执行 | `docs/concepts.md:137-145` |
| 定时任务 | Cron job | `jobs.json` 中的 at/every/cron 任务，负载可为系统事件或 Agent 轮次 | `nanobot/cron/types.py:23-81` |
| 本地触发器 | Trigger | 由 `/trigger` 创建、`nanobot trigger` 投递的工作区本地队列 | `docs/cli-reference.md:219-250` |
| 目标 | Goal | 会话 metadata 中的持续目标状态（active/blocked/cancelled） | `nanobot/session/goal_state.py:7-47` |
| 子代理 | Subagent | 由父会话派生、并发受限的后台执行单元 | `nanobot/agent/subagent.py`、`docs/concepts.md:125-135` |
| 沙箱 | Sandbox | Shell 执行隔离后端：Linux `bwrap`、macOS `seatbelt` | `nanobot/execution/sandbox.py`、`.agent/security.md:25-29` |
| SSRF 防护 | SSRF guard | 出站 HTTP 目标校验，默认阻断回环/私网/CGNAT/链路本地/云元数据 | `nanobot/security/network.py`、`.agent/security.md:15-23` |
| 工作区限制 | restrictToWorkspace | 将文件与 Shell 访问限制在工作区的应用级策略 | `nanobot/security/workspace_policy.py`、`.agent/security.md:5-13` |
| 配对 | Pairing | 聊天平台侧的访问准入机制，与 `allowFrom` 白名单配合 | `docs/configuration.md:2393-2485` |
| WebUI 密钥 | tokenIssueSecret / NANOBOT_WEB_TOKEN | WebUI 登录凭据；远程暴露前必须设置 | `docs/deployment.md:12`、`render.yaml` |
| 会话导入 | JSONL migration | 0.3.6–0.4.x 的旧会话导入，0.5.0 移除 | `nanobot/session/jsonl_migration.py:1-6` |
| 更新计划 | Update plan | `nanobot update --plan` 生成的评审文件，不直接执行 | `docs/cli-reference.md:57-64` |

## 来源

- `docs/concepts.md`、`docs/architecture.md`、`docs/cli-reference.md`、`docs/configuration.md`、`docs/memory.md`
- `nanobot/bus/events.py`、`nanobot/agent/loop.py`、`nanobot/agent/runner.py`
- `nanobot/session/manager.py`、`nanobot/session/sqlite_store.py`、`nanobot/session/display_history.py`
- `nanobot/channels/plugin.py`、`nanobot/channels/base.py`
- `nanobot/providers/base.py`、`nanobot/providers/fallback_provider.py`
- `nanobot/agent/tools/base.py`、`nanobot/agent/tools/registry.py`、`nanobot/agent/skills.py`
- `nanobot/cron/types.py`、`nanobot/session/goal_state.py`
- `.agent/security.md`、`docs/deployment.md`、`render.yaml`
