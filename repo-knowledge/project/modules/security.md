---
id: module-security
title: 模块：安全边界
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：安全边界

## 职责

约束 Agent 的文件、Shell、网络与访问权限；把安全当作功能行为的一部分，任何影响这些路径的变更都必须保留既有守卫。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/security/workspace_access.py` | 工作区访问判定 |
| `nanobot/security/workspace_policy.py` | 工作区策略：允许/拒绝路径、额外根目录 |
| `nanobot/execution/files.py` | 带作用域的文件操作（读/写/列目录/打补丁） |
| `nanobot/execution/sandbox.py` | 构建沙箱参数（bwrap/seatbelt） |
| `nanobot/security/sandbox_launcher.py` | 解析受信宿主启动器 |
| `nanobot/agent/tools/shell.py` | Shell 工具与 `restrict_to_workspace` 应用级守卫 |
| `nanobot/security/network.py` | SSRF/URL 目标校验与 DNS 验证 |
| `nanobot/agent/tools/web.py` | Web 抓取/搜索，遵循网络守卫 |
| `nanobot/agent/tools/shell_config.py` | Shell/沙箱配置模型 |

## 工作区限制

- 文件工具与原生文件搜索使用 `nanobot/execution/files.py` 的作用域操作；在打开、列目录或写补丁前应用 `workspace_policy.py` 的有效请求工作区。编辑活动快照使用授权写目标，而不是工具参数中的路径（`.agent/security.md:5-7`）。
- 媒体上传目录在受限模式下是内部额外只读根（`.agent/security.md:7`）。
- 额外文件根必须按能力细分：`extra_read_allowed_dirs`（只读）、`extra_write_allowed_dirs`（仅当工具有意允许写）、精确文件白名单（`.agent/security.md:9`）。
- Shell 也遵循 `restrict_to_workspace`：启用时，工作目录在工作区外会在执行前被拒绝，并检查明显的逃逸文本；这不是进程级隔离（`.agent/security.md:11`）。

## Shell 沙箱

- 后端：Linux `bwrap`（bubblewrap）、macOS `seatbelt`（`sandbox-exec`）；Windows 配置后端时仅告警并保留应用级守卫（`.agent/security.md:25-29`）。
- 在 Unix 上，已配置后端无法启动必须失败，不能静默退化为无隔离执行（`.agent/security.md:27`）。
- Seatbelt 不得暴露共享宿主临时目录，临时文件留在工作区（`.agent/security.md:27`）。
- 配置项：`tools.exec.sandbox`、`sandboxLauncher`、`sandboxRoBinds`、`sandboxRwBinds`、`maxRuntime`、`idleTimeout`、`allowPatterns`/`denyPatterns`、`allowedEnvKeys`（`nanobot/agent/tools/shell_config.py:9-30`）。
- Docker 内使用 bwrap 需叠加 `docker-compose.bwrap.yml` 并依赖宿主非特权用户命名空间（`docs/deployment.md:77-81`）。

## SSRF 与出站网络

- 所有 Agent 工具的 HTTP 请求必须经过 `security/network.py` 的 `validate_url_target`/`resolve_url_target`；默认阻断回环、RFC1918 私网、CGNAT、链路本地与云元数据端点（含 `169.254.169.254`）（`.agent/security.md:15-17`）。
- 唯一直接例外是 `configure_ssrf_whitelist(cidrs)`，来自 `config.tools.ssrf_whitelist`（`.agent/security.md:19`、`nanobot/config/schema.py:457`）。
- 共享网络代理（`network.proxy`，由 `providers.<name>.useProxy` 启用）是用户授权的独立信任边界；代理返回的图片下载仍拒绝畸形 URL 与可识别的私网目标（`.agent/security.md:19`）。
- HTTP/SSE MCP 同样受 SSRF 约束；本地/私网 MCP 端点只能通过显式白名单；stdio MCP 不在 HTTP SSRF 路径（`.agent/security.md:21`）。
- 规则：不要在工具中直接调用 `httpx.get`/`requests.get`，应走既有抓取工具或复用 URL 校验（`.agent/security.md:23`）。

## 访问控制与凭据

- 聊天通道：pairing 或 `allowFrom` 白名单（`docs/configuration.md:2393-2485`）。
- WebUI：默认仅回环；远程暴露前设置 `NANOBOT_WEB_TOKEN` / `tokenIssueSecret`；Cloudflare Access 模式要求回环直连与断言头（`docs/deployment.md:12,69-123`）。
- Provider `apiKey` 字段 `repr=False`，保存配置时剔除 OAuth 凭据（`nanobot/config/schema.py:263`、`nanobot/config/loader.py:134-163`）。
- 内部元数据键（如 `_runtime_control`、`_user_shell`）只能由受信传输铸造，不得从不可信客户端原样接受（`nanobot/bus/events.py:16-26`）。

## 配置汇总

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `tools.restrictToWorkspace` | `false` | 工作区限制策略意图 |
| `tools.ssrfWhitelist` | `[]` | SSRF 例外 CIDR |
| `tools.exec.sandbox` | `""` | 沙箱后端（bwrap/seatbelt） |
| `tools.exec.maxRuntime` | `3600` | 单次 Shell 最长运行秒数 |
| `tools.exec.maxRuntimeLimit` | `86400` | 运行时长上限 |
| `tools.exec.idleTimeout` | `300` | 空闲超时 |
| `tools.webuiAllowLocalServiceAccess` | `true` | 允许本机服务检查 |
| `tools.webuiAllowRemotePackageInstall` | `false` | 允许远程客户端安装依赖 |

出处：`nanobot/config/schema.py:439-457`、`nanobot/agent/tools/shell_config.py:9-30`。

## 扩展点

- 新沙箱后端：实现 `execution/sandbox.py` 的后端契约并注册到 `_BACKENDS`，保留读写根处理与“配置后端启动失败必须失败”的语义（`.agent/security.md:29`）。
- 新文件能力：请求作用域文件操作，不要在适配器里直接做路径授权或文件副作用（`.agent/security.md:13`）。
- 新出站请求：复用 `validate_url_target` 或既有抓取工具（`.agent/security.md:23`）。

## 测试

- `tests/security/`、`tests/agent/test_workspace_scope.py`、`tests/tools/test_sandbox.py`、`tests/tools/test_web_fetch_security.py`、`tests/providers/test_image_generation_security.py`。

## 待人工确认的疑点

1. `restrictToWorkspace` 默认关闭；生产环境是否应默认开启需人工裁决（属于产品安全策略，不在知识库自行修改）。
2. Windows 不提供进程级沙箱（仅应用级守卫），对外承诺需确认。

## 来源

- `.agent/security.md`
- `docs/architecture.md:201-213`
- `nanobot/security/workspace_access.py`、`nanobot/security/workspace_policy.py`、`nanobot/execution/files.py`
- `nanobot/execution/sandbox.py`、`nanobot/security/sandbox_launcher.py`、`nanobot/agent/tools/shell.py`
- `nanobot/security/network.py`、`nanobot/agent/tools/web.py`
- `nanobot/agent/tools/shell_config.py:9-30`
- `nanobot/config/schema.py:263,439-457`
- `nanobot/bus/events.py:16-26`
- `docs/deployment.md:12,69-123`
