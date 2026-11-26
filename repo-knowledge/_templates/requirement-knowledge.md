---
id: template-requirement-knowledge
title: 工单知识文件模板
source: workitem:10011
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 工单知识文件模板

复制下方代码块到 `requirement-knowledge/<工单号>-<中文摘要>.md`，替换占位符后按阶段逐节写入。

```markdown
---
id: requirement-<工单号>
title: <工单号> <主题>
source: workitem:<工单号>
updated: <YYYY-MM-DD>
owner: knowledge-manager
status: active        # active | superseded
confidence: high      # high | medium | low
distilledFlows: []    # 已沉淀的 "<来源流程ID>:<来源步骤ID>" 列表
---

# <工单号> <主题>

## 需求结论

<已过门禁的需求结论；推断内容标 low 并写明依据>

> 来源：<工单号> 需求阶段（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。

## 设计决策

<已批准的设计方案、影响面、关键取舍、已知未决项>

> 来源：<工单号> 设计阶段（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。

## 开发实现

<实现范围、关键改动、与设计的差异>

> 来源：<工单号> 开发阶段（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。

## 测试结论

<测试范围、结果、未覆盖项>

> 来源：<工单号> 测试阶段（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。

## 部署与验收事实

<部署环境、发布版本、验收结果>

> 来源：<工单号> 部署验收阶段（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。

## 遗留与风险

<遗留问题、风险、后续动作>

> 来源：<工单号> <阶段>（流程 <ID> / 步骤 <ID>），<YYYY-MM-DD>。
```

约束：六节固定，缺材料的章节写「待沉淀」及缺失原因，不空写、不编造；每节末尾标注来源；同一幂等键重复沉淀时只更新对应章节与 `updated`。
