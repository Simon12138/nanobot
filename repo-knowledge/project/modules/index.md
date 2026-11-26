---
id: project-modules-index
title: 模块知识索引
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块知识索引

| 模块 | 文件 | 覆盖范围 | 更新时间 | 来源 | 状态 | 置信度 |
| --- | --- | --- | --- | --- | --- | --- |
| CLI 与网关 | [cli-gateway.md](cli-gateway.md) | 命令分派、网关生命周期、健康端点、容器启动 | 2026-11-26 | `nanobot/cli/*`、`nanobot/gateway/*` | active | high |
| Agent 运行时 | [agent-runtime.md](agent-runtime.md) | Loop/Runner、上下文、子代理、目标、命令 | 2026-11-26 | `nanobot/agent/*`、`docs/architecture.md` | active | high |
| Provider 与模型 | [providers.md](providers.md) | Provider 注册/选择、配置、回退、Responses API | 2026-11-26 | `nanobot/providers/*` | active | high |
| 通道与总线 | [channels.md](channels.md) | 通道发现/生命周期、访问控制、消息流 | 2026-11-26 | `nanobot/channels/*`、`nanobot/bus/*` | active | high |
| 工具、技能与 MCP | [tools-skills.md](tools-skills.md) | 工具注册/执行、技能加载、MCP | 2026-11-26 | `nanobot/agent/tools/*`、`nanobot/agent/skills.py` | active | high |
| 会话与记忆 | [sessions-memory.md](sessions-memory.md) | 会话存储/恢复、长期记忆、Dream | 2026-11-26 | `nanobot/session/*`、`nanobot/agent/memory.py` | active | high |
| WebUI 与 TUI | [webui-tui.md](webui-tui.md) | 浏览器工作台、终端客户端、端口与访问控制 | 2026-11-26 | `webui/*`、`tui/*`、`nanobot/webui/*` | active | high |
| 自动化与后台任务 | [automation.md](automation.md) | cron、本地触发器、心跳、可靠性 | 2026-11-26 | `nanobot/cron/*`、`nanobot/triggers/*` | active | high |
| 安全边界 | [security.md](security.md) | 工作区限制、Shell 沙箱、SSRF、访问控制 | 2026-11-26 | `nanobot/security/*`、`.agent/security.md` | active | high |
| 自更新与发布 | [update-release.md](update-release.md) | 更新命令、备份/恢复、发布工作流 | 2026-11-26 | `nanobot/update*.py`、`.github/workflows/*` | active | medium |

## 维护说明

- 模块边界以代码职责为准，不按目录长度切分；新增模块文档需同步更新本索引与根索引。
- 涉及用户可见行为（工具名、schema、错误消息、端口、配置键）的结论必须附代码出处。
- 置信度为 `medium` 的条目（如发布链路）在用于对外承诺前需人工复核。
