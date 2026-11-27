---
id: module-webui-tui
title: 模块：WebUI 与终端 TUI
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：WebUI 与终端 TUI

## 职责

提供两个共享同一 Python 运行时的客户端：浏览器工作台（WebUI）与原生终端界面（TUI）。客户端只负责展示与交互，执行与策略保留在网关。

## WebUI

### 技术栈与构建

- React 18 + Vite 5 + Tailwind + i18next + Radix UI；Markdown/代码高亮使用 `streamdown`、`shiki`、`remark-*`（`webui/package.json`）。
- 构建：`npm run build` / `bun run build` 执行 `tsc -p tsconfig.build.json && vite build`，产物写入 `nanobot/web/dist/`，随后由 hatch 打进 wheel（`webui/package.json:6-21`、`pyproject.toml` 注释）。
- 测试：Vitest 单测、Playwright 浏览器测试；lint 由 ESLint 覆盖 `webui/src`、`webui/browser` 与通道自带前端（`webui/package.json`、`.agent/gotchas.md:7-11`）。

### 服务方式

- WebUI 由 WebSocket 通道提供，默认 `http://127.0.0.1:8765`；网关健康端点是另一个端口（18790）（`docs/architecture.md:112-119`、`nanobot/channels/websocket/runtime.py:181-182`）。
- `nanobot webui` 准备本地 WebUI 设置、按需启动共享网关并打开浏览器（`docs/cli-reference.md:159-175`）。
- 开发模式：`nanobot webui --dev` 同时启动网关与 Vite（`http://127.0.0.1:5173`），Vite 代理 `/api`、`/webui`、`/auth`，WebSocket 直连网关（`.agent/gotchas.md:11`）。

### 访问控制

- 默认仅绑定回环地址；远程访问前必须设置 `NANOBOT_WEB_TOKEN` / `channels.websocket.tokenIssueSecret`（`docs/deployment.md:12,69-74`）。
- 可选 Cloudflare Access 集成：要求回环直连且存在 `Cf-Access-Jwt-Assertion`，并拒绝普通 `X-Forwarded-*`（`docs/deployment.md:99-123`）。
- WebUI 侧工具策略：`tools.webuiAllowLocalServiceAccess`（默认 true）、`tools.webuiAllowRemotePackageInstall`（默认 false）（`nanobot/config/schema.py:440-455`）。

### 线协议与投影

- 出站投影、流缓冲、转录策略与投递轮次完成由 `nanobot/webui/outbound_delivery.py` 负责；WebSocket 传输只做接收者快照、帧排队、顺序与背压（`.agent/design.md:9`）。
- 共享客户端事件契约位于 `packages/client-events/`，客户端初始化依赖必须无环（`.agent/design.md:15-21`）。
- 附件与二进制传输：`nanobot/channels/websocket/attachment_http.py`、`binary_http.py`、`attachment_store.py`。

### 主要功能面

会议话/主题、工作区选择、模型预设、Apps（Agent Plugin/CLI App/MCP）、Skills、Automations、Goals、文件预览与 diff、上下文用量展示等。用户侧说明见 `docs/webui.md`、`docs/concepts.md`。

## 终端 TUI

- TypeScript + Bun + `@opentui/core`；`bun run build` 产物与原生可执行文件随平台 wheel 分发（`tui/package.json`、`docs/cli-reference.md:151-153`）。
- 与 WebUI 共用本地网关的 WebSocket 协议，不维护第二套 Agent 循环；没有网关时按需启动（`docs/cli-reference.md:145`）。
- 经典提示符回退：`nanobot --classic`；非交互输入、`--logs`、`--no-markdown` 自动保留经典提示符（`docs/cli-reference.md:126-155`）。
- TUI 内命令：`/sessions`、`/new-chat`、`/context`、`/branch`、`/diff`、`/detach` 等（`docs/cli-reference.md:131,145-157`）。

## 与 Desktop 共存

- Python 包与 Nanobot Desktop 使用独立运行时、配置、工作区与状态；交互式裸 `nanobot`/`nanobot webui` 可能在认证后询问使用哪个安装（`docs/cli-reference.md:39-51`）。
- Desktop 终端凭据是短期 WebSocket/API token，通过匿名管道传给 TUI，不走命令行参数或环境变量（`docs/cli-reference.md:49`）。
- 跨版本 WebUI/网关仅保证同版本组合；排障先对齐版本（`AGENTS.md`）。

## 关键配置

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `channels.websocket.host` | `127.0.0.1` | WebUI/WS 绑定地址 |
| `channels.websocket.port` | `8765` | WebUI/WS 端口 |
| `channels.websocket.publicWsUrl` | 空 | 反代后的公开 WS 地址 |
| `channels.websocket.tokenIssueSecret` | 空 | WebUI 登录密钥 |
| `channels.websocket.trustedProxyAuth` | 空 | 可信代理断言配置 |
| `tools.webuiAllowLocalServiceAccess` | `true` | 允许 Full Access Shell 检查本机服务 |
| `tools.webuiAllowRemotePackageInstall` | `false` | 允许非本机客户端安装依赖/技能 |

出处：`nanobot/config/schema.py:440-455`、`nanobot/channels/websocket/runtime.py:158-184`、`docs/deployment.md:99-123`。

## 扩展点

- 新 UI 行为：WebUI 源码在 `webui/`，按 `webui/README.md#change-a-behavior` 的变更地图定位组件与浏览器测试。
- 通道自带前端：放在 `nanobot/channels/<channel>/webui/`，同时纳入 WebUI lint/test 配置。
- 新终端行为：`tui/src/`，按 `tui/README.md#source-layout` 定位。

## 测试

- `cd webui && bun run test`、`bun run build`、`bun run test:browser`。
- `cd tui && bun test`；交互行为还需要真实终端验证（`.agent/navigation.md:31-34`）。
- CI 包含 WebUI 静态检查、生产构建、单测与 4 分片浏览器测试（`.github/workflows/ci.yml:300-439`）。

## 待人工确认的疑点

1. WebUI 与网关的协议兼容只保证同版本；灰度/回滚时的版本对齐流程需人工确认。
2. `webuiAllowRemotePackageInstall` 打开后的安全边界需要按部署环境评估。

## 来源

- `docs/architecture.md:102-125,229-248`
- `docs/cli-reference.md:39-51,126-175`
- `docs/concepts.md:41-63`
- `docs/deployment.md:12,69-123`
- `webui/package.json`、`tui/package.json`
- `nanobot/channels/websocket/runtime.py:158-184`
- `nanobot/config/schema.py:440-455`
- `.agent/design.md:9,15-21`、`.agent/gotchas.md:7-11`、`.agent/navigation.md:31-34`
- `.github/workflows/ci.yml:300-439`
