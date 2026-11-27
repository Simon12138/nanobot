---
id: module-channels
title: 模块：通道与消息总线
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：通道与消息总线

## 职责

把外部平台（WebUI/WebSocket、Telegram、Slack、飞书、微信、邮件等）翻译成统一的 `InboundMessage`，把 `OutboundMessage` 发回平台；同时承载访问控制、流式输出、重试与 outbox。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/channels/registry.py` | 扫描 `nanobot/channels/*/manifest.py` 发现通道包，懒加载插件 |
| `nanobot/channels/plugin.py` | `ChannelPlugin` 描述符与包内目标校验 |
| `nanobot/channels/base.py` | `BaseChannel` 生命周期、入站处理、访问控制、发送/流式钩子 |
| `nanobot/channels/manager.py` | 构建通道、启动/停止、出站分发、重试与 outbox |
| `nanobot/channels/websocket/` | WebUI/WebSocket 通道（默认启用）与附件、二进制 HTTP |
| `nanobot/channels/notification_routes.py` | 通道拥有的纯地址契约，供核心轮次投递使用 |
| `nanobot/bus/events.py`、`nanobot/bus/queue.py` | 消息结构与总线队列 |

## 通道包清单

`dingtalk`、`discord`、`email`、`feishu`、`linear`、`matrix`、`mattermost`、`mochat`、`msteams`、`napcat`、`qq`、`signal`、`slack`、`telegram`、`websocket`、`wecom`、`weixin`、`whatsapp`、`whatsapp_agent`，共 19 个（`nanobot/channels/`；目录中另有 `tests/`）。

其中只有 `websocket` 默认启用，并声明 `always_enabled` 能力（`nanobot/channels/websocket/manifest.py:11-19`）。

## 发现与加载

- `registry._channel_package_names` 遍历带有 `manifest.py` 的子包；`discover_plugins`/`discover_enabled` 懒加载插件（`nanobot/channels/registry.py:18-92`）。
- `load_channel_package` 导入 `nanobot.channels.<name>.manifest.PLUGIN`，并强制运行时/设置目标是包内模块（`nanobot/channels/plugin.py:128-180`）。
- `ChannelPlugin` 字段：`name`、`display_name`、`runtime`、`connector`、`setup`、`management`、`dependencies`、`default_enabled`、`settings_visible`、`capabilities`、`webui`（`nanobot/channels/plugin.py:21-40`）。
- 通道前端代码可位于 `nanobot/channels/*/webui/`，测试可位于 `nanobot/channels/*/tests/`（含 `tests/webui/`），WebUI 的 lint/test 配置会包含这些路径（`.agent/gotchas.md:7-11`）。

## 生命周期与消息流

1. 网关启动时 `ChannelManager` 构建通道；WebSocket 通道获得 WebUI 网关服务（`nanobot/channels/manager.py:197-344`）。
2. `start_all()` 启动各通道与出站分发循环（`nanobot/channels/manager.py:622-640`）。
3. 入站：通道 `_handle_message` 经过白名单/配对检查后 `bus.publish_inbound`（`nanobot/channels/base.py:327-406`）。
4. 出站：`AgentLoop` 发布 `OutboundMessage`；分发循环按通道发送，失败按 `send_max_retries` 重试并记录 outbox（`nanobot/channels/manager.py:902-1005,1147`）。
5. 会话键默认 `channel:chat_id`，通道可提供 `session_key_override` 追加线程维度（`nanobot/bus/events.py:40-48`）。

## 配置

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `channels.sendProgress` | `true` | 发送进度事件 |
| `channels.sendToolHints` | `true` | 发送工具提示 |
| `channels.showReasoning` | `true` | 展示推理（通道支持时） |
| `channels.showCompactionNotices` | `false` | 自动压缩聊天提示 |
| `channels.sendMaxRetries` | `3` | 投递尝试次数（含首次） |
| `channels.transcriptionProvider` | `groq` | 已弃用，改用顶层 `transcription.provider` |
| `channels.<name>.*` | 各通道自定义 | 通道包自行解析 |
| `channels.websocket.host/port` | `127.0.0.1` / `8765` | WebUI/WebSocket 绑定 |

出处：`nanobot/config/schema.py:40-58`、`nanobot/channels/websocket/runtime.py:158-184`。

## 访问控制

- 聊天通道通过 pairing 或 `allowFrom` 白名单限制可对话用户（`docs/concepts.md:117-123`、`docs/configuration.md:2393-2485`）。
- WebUI 远程暴露前需设置 `tokenIssueSecret` / `NANOBOT_WEB_TOKEN`；默认仅绑定 `127.0.0.1`（`docs/deployment.md:12,69-74`）。
- 通道依赖由各包 manifest 声明，CI 通过 `scripts/install_channel_dependencies.py --all-channels` 安装（`.agent/gotchas.md:3-5`）。

## 扩展点

按 `docs/channel-package-guide.md` 新增一个自包含包：提供 `manifest.py` 中的 `PLUGIN`、运行时类（`BaseChannel` 子类），可选设置页与前端资源；不要修改核心循环。

## 测试

- 根 `tests/channels/` 放共享契约测试；具体通道测试位于包内 `tests/`；WebUI/浏览器交互测试同样可随包放置。
- 通道行为的最小验证：通道测试 + `nanobot gateway` 启动路径（`docs/architecture.md:242-248`）。

## 待人工确认的疑点

1. 各聊天通道的依赖安装方式与外部平台版本兼容范围未在本条目逐项固化，接入前需查对应包 manifest 与 `docs/chat-apps.md`。
2. `channels.showReasoning` 的实际展示效果因通道而异，需按平台能力确认。

## 来源

- `nanobot/channels/registry.py:18-92`
- `nanobot/channels/plugin.py:21-40,128-180`
- `nanobot/channels/base.py:93-125,198-220,327-406`
- `nanobot/channels/manager.py:197-344,622-640,902-1005,1147`
- `nanobot/channels/websocket/manifest.py:11-19`、`nanobot/channels/websocket/runtime.py:158-184`
- `nanobot/bus/events.py:29-82`、`nanobot/bus/queue.py`
- `nanobot/config/schema.py:40-58`
- `docs/architecture.md:87-125,242-248`、`docs/concepts.md:117-123`
- `docs/channel-package-guide.md`、`docs/chat-apps.md`
- `.agent/gotchas.md:3-11`
