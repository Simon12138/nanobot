---
id: requirement-10009
title: 10009 nanobot 部署与 WebUI 访问
source: workitem:10009
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: medium
distilledFlows: ["10013:10060"]
---

# 10009 nanobot 部署与 WebUI 访问

## 需求结论

待沉淀。需求澄清结论（#10009 评论 10054，人工确认 D1–D7 与验收要点）属需求阶段材料，本文件由设计阶段沉淀单 #10011 首次建立，未写入该章节；待需求阶段沉淀单补齐。

## 设计决策

### 设计方案（设计阶段，已人工批准）

1. **部署形态**：Alinux 3 主机，以专用系统用户 `nanobot` 运行；从仓库 main（`c360a10e8`）源码**非 editable** 构建安装；由 systemd 用户服务 + linger 守护并开机自启。
2. **构建期生成 WebUI**：hatch 构建钩子在非 editable 安装时构建并打包 WebUI；editable 安装会跳过 WebUI 构建（`hatch_build.py:74-81`）。补充事实：钩子优先使用 bun，缺失时回退 npm，两者皆无时自动准备 bun，`--bun` 构建不依赖 Node（`hatch_build.py:20-23`）。
3. **监听与端口**：WebUI 监听 `0.0.0.0:8765`，网关健康检查为 `127.0.0.1:18790`（`nanobot/config/schema.py:406`、`docs/concepts.md:63`）。绑定 `0.0.0.0` 必须配置 `token` 或 `tokenIssueSecret`，否则网关拒绝绑定（`docs/webui.md:211`）。
4. **服务单元与密钥注入**：`nanobot gateway install-service` 生成的 systemd 单元**不含** `EnvironmentFile`，`[Service]` 段仅有 `Environment=PYTHONUNBUFFERED=1`，并含 `Restart=on-failure`、`RestartSec=10`、`WantedBy=default.target`（`nanobot/gateway/service.py:253-277`）；因此需以 drop-in 注入 `NANOBOT_MAAS_API_KEY`。仓库部署文档同样推荐用 `EnvironmentFile=` 加载仅部署用户可读的密钥文件（`docs/configuration.md:171-176`），权限 600。
5. **配置解析严格性**：配置中 `${VAR}` 引用按严格模式解析，被引用变量缺失时抛出 `ConfigLoadError(kind="missing_env")`，启动直接失败（`nanobot/config/loader.py:184-205`）；环境注入缺失不是降级而是启动失败。
6. **模型接入**：使用自定义 provider `providers.aliyunMaas`（`providers` 支持以额外字段声明自定义 provider，`nanobot/config/schema.py:322-390`），并以 `modelPresets.primary` 指向 MaaS 的 `qwen4-max`（`nanobot/config/schema.py:510-514`）；API Key 仅经 600 权限 EnvironmentFile 注入，不写入工单与文档。
7. **运行时基线**：要求 Python >= 3.11（`pyproject.toml:6`）。
8. **影响面**：零代码改动（仓库只读，不提交不推送），仅新增主机实体：专用用户、目录、systemd 单元、端口与 python3.11。
9. **验证映射**：设计已对 AC1–AC9 逐条映射；其中 AC5（浏览器登录）/AC6（真实模型对话）/AC7（重启自启）需外部条件配合验证。AC 明细见设计文档 `evidence/design.md`（sha256 `d06eb8ea06b9cc82401578126740af25f649e05739c93db382ed86800b5c889b`），该原文不在知识库维护工作区，本节只能引用摘要级结论（confidence: low）。

### 设计阶段已知风险与未决项

- `qwen4-max` 在 MaaS 端点是否可用：部署时需探测 `/v1/models`，不可用则暂停回报（confidence: low，无实测证据）。
- 外部可达性与云安全组放行未知：本机仅能验证内网可达，外部客户端连通性未确认（confidence: low）。
- Node 20 构建 WebUI 的兼容性：构建钩子可优先走 bun、`--bun` 构建不依赖 Node，可能规避该风险，但未实测（confidence: low）。
- 密钥注入主方案与备选尚未最终确定（confidence: low）。
- `maxTokens` / `contextWindowTokens` 待探测确定（confidence: low）。

> 来源：#10009 设计阶段（流程 10013 / 步骤 10060），沉淀单 #10011，人工批准见 #10009 评论 10059，2026-11-26。

## 开发实现

待沉淀。本阶段无内容：来源工单 #10009 当前状态为「开发中」，尚无开发阶段过门禁材料；待开发阶段沉淀单写入。

## 测试结论

待沉淀。尚无测试阶段过门禁材料；待测试阶段沉淀单写入。

## 部署与验收事实

待沉淀。尚无部署与验收阶段过门禁材料；待部署验收阶段沉淀单写入。

## 遗留与风险

待沉淀。设计阶段的已知风险与未决项已记录在「设计决策」小节；本节的最终风险汇总待后续阶段补齐。
