---
id: templates-index
title: 知识库模板索引
source: workitem:10011,10015; repo:nanobot@c360a10e8ca8afd18916b73b7575f015941a33d9
updated: 2026-11-26
owner: knowledge-manager
status: active
confidence: high
---

# 知识库模板索引

本目录记录知识文件的最小结构与 front-matter 约定；模板本身不是知识结论。

## 模板文件

| 模板 | 用途 | 更新时间 |
| --- | --- | --- |
| [requirement-knowledge.md](requirement-knowledge.md) | 工单知识文件（六节结构 + front-matter + distilledFlows） | 2026-11-26 |

## front-matter（所有知识文件必填）

```yaml
---
id: project-overview          # 全局唯一，kebab-case
title: 项目概览
source: workitem:10015; repo:nanobot@<commit>
updated: YYYY-MM-DD
owner: knowledge-manager
status: active                # active | superseded
confidence: high              # high | medium | low
---
```

- 工单知识文件额外维护 `distilledFlows`（`<来源流程ID>:<来源步骤ID>` 列表）。
- `confidence: low` 必须在正文写明依据与不确定点。

## 项目知识文档骨架

1. 职责 / 定位
2. 关键文件（带路径，必要时带行号）
3. 处理流程或结构
4. 配置项（默认值与出处）
5. 扩展点
6. 测试与验证
7. 待人工确认的疑点
8. 来源

## 工单知识章节（存在口径冲突，待裁决）

| 来源 | 章节要求 |
| --- | --- |
| 知识库运维职责说明（工单流程） | 六节：需求结论、设计决策、开发实现、测试结论、部署与验收事实、遗留与风险 |
| `knowledge-curate` 技能 | 五节：需求结论、设计决策、测试结论、部署与验收事实、遗留与风险（未列“开发实现”） |

处理约定：本次初始化不生成工单知识，冲突不影响已产出内容；后续沉淀前需由知识库管理员裁决采用六节还是五节，并在 `_audit/` 留下裁决记录。

## 修订模板

修订既有结论时，新文件必须保留：

1. 原结论
2. 修订原因
3. 裁决来源（工单评论或 `_audit/*-decision.md`）

旧文件置 `status: superseded`，不删除历史。

## 索引约定

- 每个目录的 `index.md` 列出：条目、文件、更新时间、来源、状态、置信度。
- 新增/改名/删除文件必须在同一次提交内更新所在目录 `index.md` 与根 `index.md`。
