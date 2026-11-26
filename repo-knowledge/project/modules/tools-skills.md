---
id: module-tools-skills
title: 模块：工具、技能与 MCP
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：工具、技能与 MCP

## 职责

向模型暴露可调用的工具（文件、Shell、Web、MCP、定时任务、图像生成、子代理、会话检索等），并提供按需加载的技能说明与外部 MCP 工具接入。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/agent/tools/registry.py` | 按名称注册工具、生成 schema、准备与执行调用 |
| `nanobot/agent/tools/loader.py` | 扫描 `agent/tools` 模块与 `nanobot.tools` entry point，按作用域/开关过滤 |
| `nanobot/agent/tools/base.py` | `Tool` 抽象与 `ToolResult` |
| `nanobot/agent/tools/*.py` | 各工具实现与 `*_config.py` 配置模型 |
| `nanobot/agent/tools/mcp.py` | MCP 客户端、工具/资源/提示包装、OAuth |
| `nanobot/agent/skills.py`、`nanobot/agent/skill_metadata.py` | 技能发现、front-matter 校验、渐进加载 |
| `nanobot/skills/` | 内置技能包 |

## 工具注册与执行

- `ToolRegistry` 以 `tool.name` 为键，缓存排序后的定义（内置在前、`mcp_*` 在后），提供 `prepare_call`/`execute`（`nanobot/agent/tools/registry.py:29-224`）。
- `ToolLoader.discover` 导入 `agent/tools` 下除 `_SKIP_MODULES` 外的所有模块并收集 `Tool` 子类，另加载 `nanobot.tools` entry point；`load` 再按作用域与启用状态过滤（`nanobot/agent/tools/loader.py:18-133`）。
- 工具契约：抽象 `name`/`description`/`parameters`/`execute`；标志 `read_only`、`concurrency_safe`、`exclusive`、`config_key`、`_scopes`；类钩子 `enabled`/`create`；`to_schema` 生成模型侧 schema（`nanobot/agent/tools/base.py:75-159`）。
- 调用前按 JSON Schema 校验参数，不做类型强制；MCP 使用服务器原始 input schema；默认 draft 2020-12，不检索外部引用（`docs/architecture.md:145-147`）。

## 内置工具（名称与实现）

| 工具名 | 实现文件 |
| --- | --- |
| `read_file`、`write_file`、`edit_file`、`list_dir` | `filesystem.py` |
| `grep`、`find_files` | `search.py` |
| `rg` | `rg.py` |
| `exec` | `shell.py` |
| `exec_session`、`list_exec_sessions` | `exec_session.py` |
| `web_search`、`web_fetch` | `web.py` |
| `mcp_*`（工具/资源/提示） | `mcp.py` |
| `cron` | `cron.py` |
| `generate_image` | `image_generation.py` |
| `subagent` | `subagent.py` |
| `apply_patch` | `apply_patch.py` |
| `message` | `message.py` |
| `my` | `self.py` |
| `run_cli_app` | `cli_apps.py` |
| `search_sessions`、`read_session` | `sessions.py` |
| `send_session_message` | `session_messages.py` |
| `create_goal`、`update_goal` | `nanobot/goal/assembly.py:18-26` |

工具名、schema 与错误消息属于模型契约，非有意变更不得改动（`docs/architecture.md:145`）。

## 工具配置

工具通过 `config_key` 关联到 `tools.*` 子配置；根 `ToolsConfig` 仅组装，不导入运行时实现（`.agent/design.md:11`）：

| 配置键 | 说明 |
| --- | --- |
| `tools.exec` | Shell 执行、超时、沙箱、工作目录限制 |
| `tools.file` | 文件工具限制与额外根目录 |
| `tools.web` | 搜索/抓取 Provider、代理与限制 |
| `tools.cli_apps` | CLI Apps |
| `tools.my` | 运行时自检工具 |
| `tools.image_generation` | 图像生成 |
| `tools.mcpServers` | MCP 服务器（stdio/sse/streamableHttp、OAuth、超时、`enabledTools`） |
| `tools.restrictToWorkspace` | 工作区限制策略意图 |
| `tools.ssrfWhitelist` | SSRF 例外 CIDR |
| `tools.maxSessionMessagesPerMinute` | 会话消息频率上限（默认 6） |

出处：`nanobot/config/schema.py:427-457`。

## 技能

- 目录来源与优先级：工作区 `skills/` → Agent Plugin 技能 → 内置 `nanobot/skills/`；同名去重，`disabledSkills` 过滤（`nanobot/agent/skills.py:125-159`）。
- 每个技能是一个目录，含 `SKILL.md`；front-matter 要求 `name`（kebab-case，≤64 字符）与 `description`（≤1024 字符）匹配（`nanobot/agent/skill_metadata.py:7-41`）。
- 调用方式：显式 `$skill-name`；`always=true` 的技能会注入；其余按摘要渐进加载（`nanobot/agent/skills.py:211-227,259-315,403-417`）。
- 内置技能目录：`clawhub`、`cron`、`github`、`image-generation`、`memory`、`my`、`nanobot-document`、`skill-creator`、`summarize`、`tmux`、`weather`（`nanobot/skills/`）。
- 禁用配置：`agents.defaults.disabledSkills`（`nanobot/config/schema.py:209`），网关运行时可热切换（`nanobot/cli/gateway_runtime.py:726-729`）。

## MCP

- 连接类型：stdio（`command`/`args`/`env`/`cwd`）与 HTTP（`sse`、`streamableHttp`，`url`/`headers`）；`auth: oauth` 的令牌存储在配置之外（`nanobot/config/schema.py:411-424`）。
- `enabledTools` 默认 `["*"]` 注册全部能力；设置限制后只注册列出的工具，且不注册资源/提示（`nanobot/config/schema.py:424`）。
- HTTP/SSE MCP 受 SSRF 校验约束；本地/私网端点需显式白名单（`.agent/security.md:21`）。
- MCP 生命周期由组合根管理（见 `project/architecture.md`）。

## 扩展点

- 新工具：在 `agent/tools/` 实现 `Tool` 子类并加 `*_config.py`；或提供 `nanobot.tools` entry point。
- 新技能：工作区 `skills/<name>/SKILL.md`、Agent Plugin 或内置目录。
- Agent Plugin：`<workspace>/plugins/` 下的 v1 包，可从 WebUI Apps 启用（`docs/architecture.md:222-224`）。
- 新 MCP：配置 `tools.mcpServers`。

## 测试

- `tests/tools/`、`tests/agent/tools/`、`tests/agent/test_skills_loader.py`、`tests/architecture/test_import_boundaries.py`。

## 待人工确认的疑点

1. 工具在“受限工作区”模式下的精确允许根目录集合需结合 `security` 模块与 `tools.file` 配置逐项确认。
2. 内置技能的实际行为（如 `my`、`cron`）会持续演进，使用前应阅读对应 `SKILL.md`。

## 来源

- `docs/architecture.md:127-147,215-227`
- `nanobot/agent/tools/registry.py:29-224`、`nanobot/agent/tools/loader.py:18-133`、`nanobot/agent/tools/base.py:75-159`
- 工具实现：`nanobot/agent/tools/*.py`（名称与行号见上表对应文件）
- `nanobot/agent/skills.py:125-159,211-227,259-315,403-417`、`nanobot/agent/skill_metadata.py:7-41`
- `nanobot/config/schema.py:209,411-457`
- `nanobot/goal/assembly.py:18-26`
- `.agent/design.md:11`、`.agent/security.md:15-23`、`.agent/gotchas.md:36-38`
