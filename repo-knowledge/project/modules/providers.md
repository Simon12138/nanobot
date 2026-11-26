---
id: module-providers
title: 模块：Provider 与模型接入
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 模块：Provider 与模型接入

## 职责

把不同厂商的模型服务统一为 `LLMProvider` 接口，负责 Provider 选择、鉴权、端点/模型规则、请求字段适配、流式解析、错误映射与回退。

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `nanobot/providers/registry.py` | 集中式 `ProviderSpec` 元数据（关键字、API Key 前缀、默认 base、是否本地/OAuth/网关） |
| `nanobot/providers/base.py` | `LLMProvider` 抽象、`LLMResponse`、`ToolCallRequest`、`ProviderCallContext` |
| `nanobot/providers/factory.py` | 按后端类型构造客户端；存在 fallback 时包装 `FallbackProvider` |
| `nanobot/providers/routing.py` | 路由解析：校验配置并选择后端，不创建客户端 |
| `nanobot/providers/fallback_provider.py` | 主 Provider 失败时的回退链 |
| `nanobot/providers/openai_compat_provider.py` | OpenAI 兼容后端（多数托管/本地 Provider） |
| `nanobot/providers/anthropic_provider.py` | Anthropic Messages 后端 |
| `nanobot/providers/openai_responses/` | OpenAI Responses API：请求构造、SDK/HTTP/SSE/WebSocket、解析与重放 |
| `nanobot/providers/azure_openai_provider.py`、`bedrock_provider.py` | Azure OpenAI、AWS Bedrock |
| `nanobot/providers/openai_codex_provider.py`、`xai_grok_provider.py`、`github_copilot_provider.py` | OAuth 订阅类 Provider |
| `nanobot/providers/langfuse.py` | Langfuse 追踪 |

## Provider 选择顺序

`docs/architecture.md:66-80` 描述的选择依据：

1. 显式 `agents.defaults.provider` 或当前模型预设指定的 Provider。
2. Provider 注册表中的关键字匹配。
3. API Key 前缀与 API Base URL 提示。
4. 配置了 `apiBase` 时的本地 Provider 回退。
5. 可路由多模型族的网关型 Provider 回退。

实现侧：`Config._match_provider`（`nanobot/config/schema.py:589-714`）、`resolve_provider_route`（`nanobot/providers/routing.py:93-163`）、`make_provider`（`nanobot/providers/factory.py:55-158`）。设计要求自动检测必须能从工厂追溯到具体 Provider 类（`.agent/design.md:39-41`）。

## 内置 Provider 概况

- `PROVIDERS` 共 48 个 `ProviderSpec`（`nanobot/providers/registry.py:239-954`）。
- 绝大多数托管与本地服务走 OpenAI 兼容后端：`openrouter`、`deepseek`、`groq`、`zhipu`、`dashscope`、`moonshot`、`mistral`、`siliconflow`、`volcengine`、`ollama`、`vllm`、`lm_studio`、`openai` 等。
- 专用后端：`anthropic`、`azure_openai`、`bedrock`、`openai_codex`（OAuth）、`xai_grok`（OAuth）、`github_copilot`（OAuth），以及使用 Anthropic 后端的 `kimi_coding`、`minimax_anthropic`。
- `assemblyai` 仅用于转写，不作为对话 Provider（`nanobot/providers/registry.py:935`）。
- 自定义 OpenAI 兼容端点使用 `providers.custom`，或在 `providers` 下新增未被内置名占用的键（`nanobot/config/schema.py:321-392`）。

精确的 Provider 名称与端点规则以 `registry.py` 为准；此处不逐项复制，避免与代码漂移。

## 配置

| 配置 | 说明 |
| --- | --- |
| `providers.<name>.apiKey` | 密钥，支持 `${VAR}` 引用；`repr=False` 不随日志输出 |
| `providers.<name>.apiBase` | 自定义端点；本地 Provider 用于启用判断 |
| `providers.<name>.api` | `supportedApis` 与 `preferredApi`（Responses/兼容/Anthropic Messages 等） |
| `providers.<name>.extraHeaders/extraBody/extraQuery` | 厂商特有请求字段 |
| `providers.<name>.useProxy` | 使用共享网络代理（独立信任边界） |
| `providers.<name>.thinkingStyle` | 思考字段注入风格（`thinking_type`/`enable_thinking`/`reasoning_split`） |
| `providers.bedrock.region/profile` | Bedrock 区域与共享配置 profile |
| `modelPresets.<name>` | 命名预设：`provider`、`model`、`maxTokens`、`contextWindowTokens`、`temperature`、`reasoningEffort` 等 |
| `agents.defaults.fallbackModels` | 字符串预设名或内联 fallback 配置 |

出处：`nanobot/config/schema.py:132-174,256-392,510-514`、`docs/configuration.md:280-1615`。

## 请求与响应路径

- `LLMProvider` 暴露 `chat()`、`chat_stream()` 及上下文/重试变体；`LLMResponse` 携带文本、推理、工具调用、用量与 finish reason（`nanobot/providers/base.py:82,316,382,743,1088,1732`）。
- Responses API 的 SDK、SSE、WebSocket 事件共用同一解析器；Codex 启用 WebSocket 与 beta header，其他适配器保留 HTTP 传输（`docs/architecture.md:78-80`）。
- 自动 response-ID 续接当前只在活动 WebSocket 连接上使用；完整重放状态保存在本地并按端点与模型隔离（`docs/architecture.md:80`）。
- 失败回退：`FallbackProvider` 在主 Provider 报错时按候选顺序切换（`nanobot/providers/fallback_provider.py:113-131`）。

## 扩展点

1. 在 `providers/registry.py` 增加 `ProviderSpec`。
2. 在 `config/schema.py` 的 `ProvidersConfig` 增加配置字段（如需要）。
3. 通用 OpenAI 兼容后端足够时不写新实现；否则实现 `LLMProvider` 并在 `factory.py` 接入。
4. 更新 `docs/providers.md` 与 `docs/configuration.md`。

出处：`docs/architecture.md:217-227`、`docs/development.md#adding-an-llm-provider`。

## 测试

- `tests/providers/`（路由、回退、图像生成安全、Responses 解析等）；涉及网络的行为需模拟/回放。

## 待人工确认的疑点

1. 48 个 Provider 中哪些支持 Responses API、哪些仅兼容 Chat Completions，未在本条目逐项固化；接入前需查 `registry.py` 的 `api` 元数据。
2. OAuth Provider 的凭据存储位置不在配置文件中（保存时剔除），实际路径需按安装环境确认。

## 来源

- `docs/architecture.md:64-86,217-227`
- `docs/providers.md`、`docs/configuration.md:280-1615`
- `nanobot/providers/registry.py:239-954`
- `nanobot/providers/base.py:82,316,382,743,1088,1732`
- `nanobot/providers/factory.py:55-158,196-224`
- `nanobot/providers/routing.py:93-163`
- `nanobot/providers/fallback_provider.py:113-131`
- `nanobot/config/schema.py:132-174,256-392,510-514,589-714`
- `.agent/design.md:39-41`
