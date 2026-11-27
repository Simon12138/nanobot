---
id: requirement-knowledge-index
title: 工单知识索引
source: workitem:10009,10011,10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 工单知识索引

本目录用于存放已通过人工门禁或已验收工单的知识沉淀，一个工单一个文件。

## 当前条目

| 工单 | 主题 | 已沉淀流程:步骤 | 更新时间 | 状态 |
| --- | --- | --- | --- | --- |
| [10009](10009-nanobot部署与webui访问.md) | nanobot 部署与 WebUI 访问 | 10013:10060（设计） | 2026-11-26 | active |

## 命名与结构约定

- 文件名：`<工单号>-<中文摘要>.md`；一工单一个文件，六节固定结构，幂等键见各文件 front-matter 的 `distilledFlows`。
- front-matter 必须包含 `id`、`title`、`source`、`updated`、`owner`、`status`、`confidence`，并额外维护 `distilledFlows`（`<来源流程ID>:<来源步骤ID>` 列表）。
- 章节结构存在口径冲突，待知识库管理员裁决（见 `../_templates/index.md`）。

## 幂等

- 文件已存在且 `distilledFlows` 覆盖来源阶段时跳过，不重复沉淀。
