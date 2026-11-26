---
id: module-cli-gateway
title: 模块：CLI 与网关
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：CLI 与网关

## 职责

提供 `nanobot` 命令行入口、子命令分派、网关进程生命周期（前台/后台/系统服务）、健康端点与容器启动引导。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/cli/entry.py` | 控制台脚本入口：安装锁、`update` 特例、Agent 别名、桌面目标选择，最后交给 Typer 应用 |
| `nanobot/cli/commands.py` | 根 Typer 应用与子命令注册（`onboard`、`status`、`trigger`、`webui`、`gateway`、`agent`、`channels`、`plugins`、`provider`、`update`、`remote`） |
| `nanobot/cli/agent.py` | `nanobot agent` 参数与单次/交互运行 |
| `nanobot/cli/gateway.py` | `nanobot gateway` 应用：默认启动 + `status`/`logs`/`stop`/`restart`/`install-service`/`uninstall-service` |
| `nanobot/cli/gateway_runtime.py` | 网关启动装配、运行循环、健康服务、系统 cron 注册 |
| `nanobot/gateway/runtime.py` | 网关进程租约、状态、健康探测、启动命令构造 |
| `nanobot/gateway/service.py` | systemd 用户服务 / macOS LaunchAgent 安装器 |
| `nanobot/cli/docker_bootstrap.py` | 容器首启配置生成 |
| `nanobot/installation_lock.py` | 安装/运行时租约，避免并发升级 |
| `nanobot/cli/process_identity.py` | 进程角色标识（日志与进程标题） |

## 入口分派流程

1. `nanobot` 控制台脚本调用 `nanobot.cli.entry:main`（`pyproject.toml` `[project.scripts]`）。
2. `main()` 先解析 `--home`，对 `update` 子命令走独立路径，其余命令在 `installation_access()` 安装锁内执行（`nanobot/cli/entry.py:85-104`）。
3. `_dispatch()` 处理：shell completion、`update` 专用应用、无参数时桌面目标探测、裸参数/`agent` 的原生 TUI 候选，最后落到 `nanobot.cli.commands:app`（`nanobot/cli/entry.py:107-147`）。
4. `nanobot serve` 已被移除；执行会提示改用 `nanobot webui` 并以退出码 2 结束（`nanobot/cli/commands.py:137-151`）。

## 子命令清单

| 命令 | 说明 |
| --- | --- |
| `nanobot onboard` | 初始化/刷新配置与工作区，可选 `--wizard`、`--refresh` |
| `nanobot status` | 显示配置/工作区/模型/Provider 就绪度，不调用模型 |
| `nanobot agent`（或裸 `nanobot`） | 单次或交互式 Agent |
| `nanobot webui` | 准备 WebUI 设置、按需启动网关、打开浏览器 |
| `nanobot gateway` | 前台/后台网关；含服务安装与管理子命令 |
| `nanobot trigger <id> [message]` | 向本地触发器投递消息（省略消息时读 stdin） |
| `nanobot channels status` / `login <channel>` | 通道状态与交互式登录 |
| `nanobot plugins list` / `enable` / `disable` | 可选特性与通道的启停 |
| `nanobot provider login/logout` | OAuth Provider 登录/登出 |
| `nanobot update` | 自更新（`--check`、`--dev`、`--plan`、`--job`） |
| `nanobot remote` | 远程实例相关命令组 |

以上注册见 `nanobot/cli/commands.py:86-582`。

## 网关生命周期

- `nanobot gateway` 默认前台运行；`--background` 通过托管进程启动；`gateway stop/status/logs/restart` 读取记录的进程状态（`nanobot/cli/gateway.py:139-402`、`nanobot/gateway/runtime.py`）。
- 启动前从 `config.gateway.port` 解析健康端口（默认 18790），获取实例锁后装配运行时（`nanobot/cli/gateway_runtime.py:383-429`）。
- 启动内容：已启用通道、已配置的 WebSocket 通道、工作区作用域 cron、Dream 与 heartbeat 系统任务、健康端点（`docs/architecture.md:102-110`）。
- 健康端点只处理 `GET /health`，返回 200/503 JSON，含 `runtime`（instance_id/run_id/version）与 `maintenance`（`nanobot/cli/gateway_runtime.py:780-836`）。
- 系统服务：`install-service --manager systemd|launchd`，生成由 OS 托管的**前台**网关，避免嵌套后台进程（`docs/cli-reference.md:194-209`、`nanobot/gateway/service.py`）。

## 配置项

| 配置 | 默认 | 作用 |
| --- | --- | --- |
| `gateway.host` | `127.0.0.1` | 健康端点绑定地址 |
| `gateway.port` | `18790` | 健康端点端口 |
| `gateway.restart_mode` | `auto` | 重启方式（auto/exec/spawn/exit） |
| `gateway.heartbeat.enabled` | `true` | 心跳系统任务开关 |
| `gateway.heartbeat.interval_s` | `1800` | 心跳间隔秒数 |
| `agents.defaults.workspace` | `~/.nanobot/workspace` | 默认工作区 |

出处：`nanobot/config/schema.py:395-408,177-247`。

## 容器与平台启动

- `entrypoint.sh` 负责：root 启动时 chown 并 `setpriv` 降权、检查 `~/.nanobot` 可写、Render 环境从模板生成配置、Compose 首启执行 `docker_bootstrap`，最后 `exec nanobot "$@"`（`entrypoint.sh:1-55`）。
- `docker-compose.yml` 以 `NANOBOT_DOCKER_BOOTSTRAP=1` 与 `NANOBOT_WEB_TOKEN` 启动网关，暴露 `127.0.0.1:18790` 与 `8765`（`docker-compose.yml:1-47`）。
- Render 使用 `entrypoint.sh gateway`、`PORT=8765` 与持久盘挂载（`render.yaml:1-34`）。

## 扩展点

- 新增 CLI 子命令：在 `nanobot/cli/commands.py` 注册 Typer 命令/子应用。
- 新增网关管理子命令：扩展 `create_gateway_app()`（`nanobot/cli/gateway.py`）。
- 进程托管/健康判断：复用 `nanobot/gateway/runtime.py` 的租约与健康身份，不要另起状态文件格式。

## 测试

- `tests/cli/`（含命令、输入、桌面目标、启动器）、`tests/gateway/`；CI 对 CLI 交互测试有单独串行约束（`.github/workflows/ci.yml:45-78`）。

## 待人工确认的疑点

1. `gateway.restart_mode` 的四个取值在源码中的实际行为差异未在本条目展开，需按运维场景确认默认值是否合适。
2. 健康端点的 `runtime` 身份校验要求同版本；生产多实例灰度时的版本对齐策略需人工确认。

## 来源

- `nanobot/cli/entry.py:85-147`
- `nanobot/cli/commands.py:86-582`
- `nanobot/cli/gateway.py:139-402`
- `nanobot/cli/gateway_runtime.py:383-429,780-836,837-865,939-983`
- `nanobot/gateway/runtime.py`、`nanobot/gateway/service.py`
- `nanobot/config/schema.py:177-247,395-408`
- `entrypoint.sh`、`docker-compose.yml`、`render.yaml`
- `docs/cli-reference.md:177-217`、`docs/architecture.md:102-110`
