---
id: project-runbook
title: 运行手册
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 运行手册

本手册只描述仓库已有的命令与配置事实；部署脚本、CI 与云资源不在知识库运维的修改范围内。

## 环境要求

- Python 3.11+（`pyproject.toml` `requires-python`）。
- 可访问的模型服务：云 API、OAuth 订阅或本地服务（Ollama/vLLM/LM Studio）。
- 构建 WebUI 需要 Node.js/npm（Docker 使用 `node:24-bookworm-slim`）；构建 TUI 与仓库脚本使用 Bun。
- 容器运行需要 Docker / Docker Compose；Linux 上可选用 bubblewrap 做 shell 沙箱。

## 安装

| 方式 | 命令 / 说明 | 出处 |
| --- | --- | --- |
| 官方安装脚本 | `curl -fsSL https://raw.githubusercontent.com/HKUDS/nanobot/main/scripts/install.sh \| sh`（Windows 用 `install.ps1`） | `docs/quick-start.md:18-34` |
| 源码开发安装 | `pip install -e ".[dev]"` | `CONTRIBUTING.md:70-79` |
| 容器镜像 | 使用本仓库 `Dockerfile` 构建；Docker Hub 第三方镜像不受维护 | `docs/deployment.md:76-95` |

## 初始化与配置

```bash
nanobot onboard              # 创建 ~/.nanobot/config.json 与 ~/.nanobot/workspace/
nanobot onboard --refresh    # 非交互补齐缺失默认字段，保留已有值
nanobot onboard --wizard     # 交互式向导（无浏览器/SSH 场景）
nanobot status               # 校验配置/工作区与 Provider/模型就绪度，不调用模型
```

- 配置支持 `${VAR}` 引用；缺失变量在启动时抛 `ConfigLoadError`（`docs/configuration.md:107-215`、`nanobot/config/loader.py:182-231`）。
- 也可用环境变量覆盖：前缀 `NANOBOT_`、嵌套分隔符 `__`（`nanobot/config/schema.py:764-767`）。
- 多实例使用 `--home <dir>`（全局选项，位于子命令之前），或显式 `--config` / `--workspace`（`docs/cli-reference.md:25-37`）。

## 运行

| 场景 | 命令 |
| --- | --- |
| 单次问答 | `nanobot -m "Hello!"` / `nanobot agent -m "Hello!"` |
| 终端交互 | `nanobot`（`--classic` 使用 Python 提示符；`--session <id>` 复用会话） |
| 浏览器 | `nanobot webui`（`--detach` 常驻、`--dev` Vite 开发、`--port` 指定 WebUI 端口） |
| 无头网关 | `nanobot gateway`（`--background` 后台、`--verbose` 日志、`--port` 健康端口） |
| 本地触发器 | 先在会话里 `/trigger <name>`，再 `nanobot trigger <id> "message"` |

默认端口：健康端点 `http://127.0.0.1:18790/health`；WebUI/WebSocket `http://127.0.0.1:8765`（`docs/cli-reference.md:211-217`、`nanobot/channels/websocket/runtime.py:181-182`）。WebUI 由 WebSocket 通道提供，不由健康端点提供（`docs/architecture.md:112-119`）。

## 网关运维

```bash
nanobot gateway status
nanobot gateway logs            # --no-follow 只打印
nanobot gateway restart
nanobot gateway stop
nanobot gateway install-service --manager systemd --dry-run
nanobot gateway install-service --manager systemd
nanobot gateway uninstall-service --manager systemd
```

- macOS 使用 `--manager launchd`；日志位于 `~/.nanobot/logs/`（`docs/deployment.md:132-180`）。
- 健康检查：`curl --fail http://127.0.0.1:18790/health`；返回 200 表示就绪，503 表示未就绪（`nanobot/cli/gateway_runtime.py:780-836`）。
- `gateway stop/status/logs` 在安装不完整时仍可用；其余命令受安装锁保护（`nanobot/cli/entry.py:85-104`）。

## 容器部署

```bash
docker compose up -d
docker compose logs -f nanobot-gateway
docker compose run --rm nanobot-cli agent -m "Hello!"
curl --fail http://127.0.0.1:18790/health
docker compose down
```

- 首次启动会生成 `~/.nanobot/config.json` 并在日志中打印随机 WebUI 密码；可用 `NANOBOT_WEB_TOKEN` 预先指定（`docs/deployment.md:40-61`、`docker-compose.yml`）。
- 预装通道依赖：`NANOBOT_CHANNELS=telegram,slack docker compose build`（`Dockerfile:42-45`）。
- 容器以 UID 1000 运行；宿主机 `~/.nanobot` 需可写，权限问题用 `sudo chown -R 1000:1000 ~/.nanobot`（`docs/deployment.md:74-76`、`entrypoint.sh:15-26`）。
- `entrypoint.sh` 以 root 启动时会 chown 后通过 `setpriv` 降权；降权失败则拒绝运行（`entrypoint.sh:4-13`）。
- bwrap 沙箱需叠加 `docker-compose.bwrap.yml`（`docs/deployment.md:77-81`）。

## Render 部署

- `render.yaml` 定义单个 Docker Web 服务、健康检查路径 `/`、端口 `PORT=8765`、1GB 持久盘挂载 `/home/nanobot/.nanobot`，默认关闭自动部署（`render.yaml:1-34`）。
- 必需环境变量：`ANTHROPIC_API_KEY`、`NANOBOT_WEB_TOKEN`（`sync: false`，在控制台填写）。
- 首次启动由 `entrypoint.sh` 从 `/app/render-config.json` 生成 `config.json`（`entrypoint.sh:28-39`）。

## 开发与验证

```bash
# Python
pytest
ruff check nanobot/
python -m scripts.check_import_boundaries

# 严格类型检查（与 CI 对齐）
uv sync --all-extras --dev
uv run --no-sync python -m scripts.install_channel_dependencies --all-channels
uv run --no-sync basedpyright

# WebUI
cd webui && bun run test && bun run build
cd webui && bun run test:browser

# TUI
cd tui && bun test
```

- 覆盖率下限 75%，测试路径包含 `tests` 与 `nanobot/channels`（`pyproject.toml` `[tool.pytest.ini_options]`、`[tool.coverage.report]`）。
- 不要运行全局 `ruff format`；上游约定只格式化本次有意修改的文件（`AGENTS.md`、`CONTRIBUTING.md:84-88`）。
- CI 由 `scripts/ci_scope.py` 按受影响契约选择 Python/WebUI/TUI/Docker/原生验收任务（`.github/workflows/ci.yml:19-43`）。

## 构建产物

- Python 包由 hatchling 构建；`hatch_build.py` 负责在打包时准备 WebUI 产物（`pyproject.toml` `[build-system]`、`hatch_build.py`）。
- WebUI 构建输出到 `nanobot/web/dist/`，该目录被 git 忽略但会打进 wheel/sdist（`pyproject.toml` 注释、`.gitignore`）。
- TUI 有独立的 Bun 构建与发布工作流（`tui/package.json`、`.github/workflows/tui-release.yml`）。

## 数据备份与升级

- 升级前停止所有共享同一配置目录的进程，并整体备份配置目录与工作区（`docs/cli-reference.md:66,143`）。
- `nanobot update` 针对 PyPI 稳定版；`--dev` 更新源码；`--plan` 只生成计划不执行（`docs/cli-reference.md:55-74`）。
- 就地升级无自动回滚；失败会留下未完成标记，需在同一环境重试（`docs/cli-reference.md:72`）。
- 降级需要 `--allow-downgrade` 且数据兼容；旧版本不能读取新格式 WebUI 历史（`docs/cli-reference.md:70,143`）。
- 自更新实现集中在 `nanobot/update*.py`，发布元数据与信任在 `nanobot/release_feed.py`、`nanobot/release_trust.py`、`nanobot/update_contract.json`。

## 常见故障排查

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| `nanobot status` 显示配置缺失 | 未执行初始化 | `nanobot onboard` |
| Provider/model 未就绪 | 未配置 API Key 或预设 | WebUI **Settings → Models** 或 `nanobot onboard --wizard` |
| 启动报地址被占用 | 已有网关在运行 | `nanobot gateway stop`，或换端口 |
| Docker 内 Permission denied | 宿主机挂载目录属主不符 | `sudo chown -R 1000:1000 ~/.nanobot` |
| 网关以 root 启动失败 | setpriv 降权不可用 | 修复容器权限或显式指定运行用户（`entrypoint.sh:4-13`） |
| 旧历史导入失败 | JSONL 损坏或身份冲突 | 按启动错误修复后重启；源文件在失败时保持不变 |
| WebUI/TUI 与网关异常 | 版本不一致 | 先对齐版本；跨版本无兼容保证（`AGENTS.md`） |

## 待人工确认的疑点

1. `docs/deployment.md` 推荐 Render 暴露 8765，而健康检查 18790 通常只绑本机；生产环境的端口暴露与反代约定需人工确认。
2. 官方安装脚本指向上游 `HKUDS/nanobot`，本仓库为绑定 fork；实际发布渠道与镜像来源需确认。
3. CI 覆盖 Python 3.14，但 classifiers 未声明；对外支持口径需确认。

## 来源

- `docs/quick-start.md`
- `docs/cli-reference.md`
- `docs/deployment.md`
- `docs/configuration.md:107-215`
- `CONTRIBUTING.md:70-101`
- `.agent/gotchas.md`
- `pyproject.toml`、`Dockerfile`、`docker-compose.yml`、`entrypoint.sh`、`render.yaml`
- `nanobot/cli/entry.py:85-104`
- `nanobot/cli/gateway_runtime.py:780-836`
- `nanobot/channels/websocket/runtime.py:181-182`
- `nanobot/config/schema.py:764-767`
- `.github/workflows/ci.yml:19-43`、`scripts/ci_scope.py`
