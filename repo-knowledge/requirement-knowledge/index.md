---
id: requirement-knowledge-index
title: 工单知识索引
source: workitem:10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 工单知识索引

本目录用于存放已通过人工门禁或已验收工单的知识沉淀，一个工单一个文件。

## 当前状态

暂无条目。本次任务为老项目知识初始化，不包含工单沉淀。

## 命名与结构约定

- 文件名：`<工单号>-<中文摘要>.md`。
- front-matter 必须包含 `id`、`title`、`source`、`updated`、`owner`、`status`、`confidence`，并额外维护 `distilledFlows`（`<来源流程ID>:<来源步骤ID>` 列表）。
- 章节结构存在口径冲突，待知识库管理员裁决（见 `../_templates/index.md`）。

## 幂等

- 文件已存在且 `distilledFlows` 覆盖来源阶段时跳过，不重复沉淀。
