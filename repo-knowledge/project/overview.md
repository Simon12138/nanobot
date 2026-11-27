---
id: project-overview
title: 项目概览
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 项目概览

## 一句话定位

nanobot 是一个用 Python 编写的轻量级、可自托管的个人 AI 助手框架：同一个 Agent 运行时同时服务于 WebUI、终端 TUI 和各聊天平台，具备工具调用、长期记忆、MCP、定时任务与多种模型接入能力（`README.md`、`docs/concepts.md`）。

## 基本事实

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 发行包名 | `nanobot-ai` | `pyproject.toml` `[project].name` |
| 当前代码版本 | `0.4.2` | `pyproject.toml` `[project].version`；`nanobot/__init__.py:42-50` |
| 许可证 | MIT | `pyproject.toml`、`LICENSE` |
| Python 要求 | `>=3.11`（classifiers 列 3.11/3.12；CI 覆盖 3.11 与 3.14） | `pyproject.toml`；`.github/workflows/ci.yml` |
| CLI 入口 | `nanobot = nanobot.cli.entry:main`、`nanobot-desktop-tui = nanobot.cli.desktop_tui:main` | `pyproject.toml` `[project.scripts]` |
| 构建后端 | hatchling + 自定义构建钩子 `hatch_build.py` | `pyproject.toml` `[build-system]` |
| 默认配置 | `~/.nanobot/config.json` | `nanobot/config/loader.py:29-33`、`docs/concepts.md` |
| 默认工作区 | `~/.nanobot/workspace/` | `nanobot/config/paths.py:50-53` |
| 网关健康端口 | `18790`（`GET /health`） | `nanobot/config/schema.py:402-408`、`nanobot/cli/gateway_runtime.py:780-836` |
| WebUI / WebSocket 端口 | `8765` | `nanobot/channels/websocket/runtime.py:181-182`、`docs/concepts.md` |
| 内置通道包 | 19 个（另有一个 `tests/` 目录） | `nanobot/channels/` |
| 内置 Provider 规格 | 48 个 `ProviderSpec` | `nanobot/providers/registry.py` |
| 内置技能 | 11 个目录 | `nanobot/skills/` |

## 运行形态

| 形态 | 命令 | 说明 |
| --- | --- | --- |
| 单次 CLI | `nanobot -m "Hello!"` 或 `nanobot agent -m "Hello!"` | 一次请求后退出，适合脚本与排障 |
| 交互终端 | `nanobot` / `nanobot agent` | 原生 TypeScript TUI，也可 `--classic` 回退到 Python 提示符 |
| WebUI | `nanobot webui` | 准备本地 WebUI 配置并按需拉起共享网关，再打开浏览器 |
| 网关 | `nanobot gateway` | 无头常驻：启用通道、cron、Dream、heartbeat、健康端点 |
| 进程内 SDK | `from nanobot import Nanobot` | 通过 `Nanobot.from_config()` 以编程方式运行 Agent |

命令行入口与子命令的完整定义见 `nanobot/cli/entry.py`、`nanobot/cli/commands.py`；入口分派细节见 `project/modules/cli-gateway.md`。

## 仓库结构

| 路径 | 内容 |
| --- | --- |
| `nanobot/` | Python 核心：CLI、网关、Agent、通道、Provider、工具、会话、记忆、安全、自更新 |
| `webui/` | React 18 + Vite + Tailwind 浏览器工作台；构建产物写入 `nanobot/web/dist/` |
| `tui/` | TypeScript + Bun + OpenTUI 原生终端客户端 |
| `packages/client-events/` | WebUI/TUI 共享的客户端事件契约（TypeScript） |
| `tests/` | Python 测试；通道测试也可与通道包同目录（`nanobot/channels/*/tests/`） |
| `docs/` | 用户与架构文档（产品文档与代码同仓库） |
| `scripts/` | 安装、CI 选择、发布、Docker smoke、导入边界检查等脚本 |
| `.agent/` | 面向编码 Agent 的导航、设计、安全、陷阱说明 |
| `Dockerfile`、`docker-compose.yml`、`render.yaml`、`entrypoint.sh` | 容器与 Render 部署资产 |
| `.github/workflows/` | CI、原生验收、PyPI 发布、TUI 发布、候选版本与发布审批 |

## 核心概念

| 概念 | 含义 | 详见 |
| --- | --- | --- |
| AgentLoop | 面向通道的一轮对话编排：会话/工作区解析、上下文构建、钩子、出站投递 | `project/modules/agent-runtime.md` |
| AgentRunner | 面向模型的多轮循环：调用 Provider、流式增量、执行工具、回填结果 | `project/modules/agent-runtime.md` |
| Session | 一段对话的持久化历史；键为 `channel:chat_id`，带 `generation`/`revision` 并发控制 | `project/data-model.md` |
| Memory / Dream | 工作区长期记忆与周期性整合任务 | `project/modules/sessions-memory.md` |
| Channel | 把外部平台消息转换为 `InboundMessage`/`OutboundMessage` 的适配器 | `project/modules/channels.md` |
| Provider | 大模型后端适配器，统一到 `LLMProvider.chat()` | `project/modules/providers.md` |
| Tool / Skill / MCP | 可调用工具、按需加载的说明型技能、外部 MCP 工具服务 | `project/modules/tools-skills.md` |
| Gateway | 常驻进程，承载通道、cron、heartbeat、健康检查与 WebUI 后端 | `project/modules/cli-gateway.md` |
| Automation | cron 定时任务、本地 trigger、heartbeat 三类自主执行 | `project/modules/automation.md` |

## 技术栈

- 运行时：Python 3.11+、asyncio、Typer、Pydantic v2 / pydantic-settings、loguru。
- 持久化：标准库 `sqlite3`（会话库，WAL + `synchronous=FULL`）、JSON/JSONL 文件（配置、cron、记忆日志）。
- 网络与协议：`aiohttp`、`httpx[socks]`、`websockets`、MCP SDK。
- 前端：React 18、Vite 5、Tailwind、i18next；TUI 使用 Bun 与 `@opentui/core`。
- 构建与交付：hatchling、uv（容器）、npm 构建 WebUI、GitHub Actions、sigstore（发布信任）。

## 文档与代码的关系

本知识库以仓库当前 HEAD（`c360a10e8ca8afd18916b73b7575f015941a33d9`）为准；`docs/` 中声明“跟随最新开发分支”，可能与已安装的稳定版存在差异，涉及行为判断时以代码为准并在条目中标注出处。`AGENTS.md` 与 `.agent/*` 是仓库对内部开发者的约定，其中“不手工换行 Markdown 段落”等规则仅约束上游仓库贡献，不改变本知识库的写入规范。

## 待人工确认的疑点

1. 绑定远端为 `git@github.com:Simon12138/nanobot.git`，而 `README.md` 与 badge 指向上游 `HKUDS/nanobot`；本仓库与上游的发布/同步关系未在仓库内说明，需人工确认以哪个发布渠道为准。
2. `pyproject.toml` classifiers 仅声明 3.11/3.12，但 CI 在 3.14 上运行完整测试；对外支持矩阵口径需确认。
3. `docs/` 与当前 `0.4.2` 代码之间是否存在个别未同步描述，需按“以代码为准”原则持续巡检。

## 来源

- `pyproject.toml`
- `README.md`
- `docs/concepts.md`
- `docs/cli-reference.md`
- `nanobot/config/schema.py:177-247,402-408`
- `nanobot/config/paths.py:50-53`
- `nanobot/cli/gateway_runtime.py:780-836`
- `nanobot/channels/websocket/runtime.py:181-182`
- `nanobot/__init__.py:42-50`
