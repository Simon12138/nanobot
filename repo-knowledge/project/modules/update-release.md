---
id: module-update-release
title: 模块：自更新与发布
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: medium
---

# 模块：自更新与发布

## 职责

在不破坏已安装环境的前提下更新 nanobot（PyPI 稳定版或源码），并以受审查的流程完成候选版本、审批与发布。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/update.py` | `nanobot update` 命令入口 |
| `nanobot/update_plan.py` | 生成可审查的更新计划 |
| `nanobot/update_jobs.py` | 保存/恢复的更新作业 |
| `nanobot/update_worker.py` | 独立更新 worker（文件较大，承载主要变更逻辑） |
| `nanobot/update_manager.py` | 读取包管理器意图与安装来源 |
| `nanobot/update_profiles.py` | 安装配置/环境画像 |
| `nanobot/update_uv.py` | uv 安装路径 |
| `nanobot/update_service.py`、`update_restart.py`、`update_stop.py` | 服务停止/重启与恢复 |
| `nanobot/update_backup.py`、`update_recovery.py`、`update_fallback.py` | 备份、恢复与回退 |
| `nanobot/update_handoff.py`、`update_guidance.py` | 显式交接与升级指引 |
| `nanobot/release_feed.py`、`release_trust.py`、`releases.py` | 发布元数据、信任校验与版本记录 |
| `nanobot/update_contract.json` | 更新契约 |
| `nanobot/installation_lock.py` | 安装/数据租约 |

## 命令与行为

| 命令 | 行为 |
| --- | --- |
| `nanobot update --check` | 查询最新稳定 PyPI 版本，不安装 |
| `nanobot update` | 在普通 Python venv 中安装最新稳定版 |
| `nanobot update --dev` | 更新源码并准备匹配接口；已有可编辑检出会快进到上游 |
| `nanobot update --plan` | 保存精确评审计划，不安装、不重启 |
| `nanobot update --job <plan-id>` | 检查已保存的 WebUI 更新作业 |
| `nanobot update --allow-downgrade` | 允许降级（需数据兼容） |

出处：`docs/cli-reference.md:55-74`。

## 运行约束

- 更新前备份配置、工作区与聊天历史，并停止共享同一 Python 环境的所有进程；新版本可能迁移数据（`docs/cli-reference.md:66`）。
- 就地更新无自动回滚；失败或中断会留下未完成标记，需在同一环境重试（`docs/cli-reference.md:72`）。
- 源码更新需要 Git；`--dev` 会在 `~/.nanobot/src/` 创建可编辑检出并跟踪官方 `main`；需要时下载固定版本 Bun，不要求 Node（`docs/cli-reference.md:68`）。
- 从源码切回发行版需在独立环境安装并在检出面外启动；不支持就地源码↔发行版切换（`docs/cli-reference.md:68`）。
- CLI 检查的是安装，不是运行中进程；更新后需重启并验证运行中网关的版本与行为（`docs/cli-reference.md:70`）。
- WebUI **Settings → About** 使用独立的审查式更新/重启流程；源码更新仅限 CLI（`docs/cli-reference.md:74`）。
- `nanobot update` 与 `update --job` 在 `entry.py` 中走独立路径，以避免安装锁与自替换冲突（`nanobot/cli/entry.py:85-104,120-128`）。

## 发布与 CI

| 工作流 | 作用 |
| --- | --- |
| `.github/workflows/ci.yml` | 按路径选择 Python/WebUI/TUI/Docker/原生验收 |
| `.github/workflows/native-acceptance.yml` | 原生包验收 |
| `.github/workflows/release-candidate.yml` | 候选版本构建/计划 |
| `.github/workflows/release-approve.yml` | 发布审批 |
| `.github/workflows/publish-pypi.yml`、`trigger-pypi.yml` | PyPI 发布与触发 |
| `.github/workflows/tui-release.yml` | TUI 原生发布 |

相关脚本：`scripts/prepare_pypi_release.py`、`reconcile_pypi_release.py`、`release_candidate.py`、`release_plan.py`、`verify_release_approval.py`、`verify_installed_release.py`、`verify_packaged_update.py`、`release_install_guide.py`。发布文档见 `docs/releasing.md`、`docs/updating.md`。

## 构建与产物

- Python 包由 hatchling 构建；`hatch_build.py` 在构建时准备 WebUI 产物（`pyproject.toml`、`hatch_build.py`）。
- Wheel 包含 `nanobot/web/dist/**`（WebUI）、`nanobot/templates/**`、`nanobot/skills/**`、通道前端资源；sdist 额外包含 `docs/**/*.md`（`pyproject.toml` `[tool.hatch.build]`）。
- TUI 平台 wheel 内含匹配的原生可执行文件；源码分发的 launcher 可下载版本匹配的 GitHub release，`NANOBOT_TUI_NO_DOWNLOAD=1` 可禁用该回退（`docs/cli-reference.md:151-153`）。

## 信任与安全

- 依赖中固定 `sigstore==4.5.0`（`pyproject.toml`），发布信任逻辑在 `nanobot/release_trust.py`。
- 更新契约在 `nanobot/update_contract.json`；安装锁与数据租约在 `nanobot/installation_lock.py`、`nanobot/data_access.py`。
- 更新过程不得打印/记录凭据；配置保存会剔除 OAuth 凭据（`nanobot/config/loader.py:134-163`）。

## 扩展点

- 新安装管理器：在 `update_manager.py` 增加来源识别，在 `update_profiles.py`/`update_worker.py` 接入执行路径；保留备份、恢复与未完成标记语义。
- 新发布渠道：同步 `release_feed.py`、`update_contract.json` 与发布工作流，避免只改一端。

## 测试

- `tests/test_update*.py`、`tests/test_release_*.py`、`tests/test_prepare_pypi_release.py`、`tests/test_verify_*` 等；CI 在 Windows 上另跑真实隔离更新兼容性任务（`.github/workflows/ci.yml:209-299`）。

## 待人工确认的疑点

1. 本仓库为 fork，发布工作流是否实际发布到 PyPI/上游渠道需人工确认（见 `project/overview.md` 疑点 1）。
2. `release_trust.py` 的具体信任根与签名验证范围未在本条目展开，安全评审时需单独核对。

## 来源

- `docs/cli-reference.md:55-74,151-153`
- `docs/releasing.md`、`docs/updating.md`
- `nanobot/update*.py`、`nanobot/release_feed.py`、`nanobot/release_trust.py`、`nanobot/releases.py`
- `nanobot/update_contract.json`、`nanobot/installation_lock.py`、`nanobot/data_access.py`
- `nanobot/cli/entry.py:85-104,120-128`
- `pyproject.toml`、`hatch_build.py`
- `.github/workflows/*.yml`、`scripts/*release*.py`
