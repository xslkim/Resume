# M3 知识库引擎（LLM Wiki） — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：维护个人职业 wiki。把 raw 素材**编译（Ingest）**进结构化、互链的 markdown 页面，自动双语，维护 index/log，标记矛盾；定期 **Lint** 自检。**本模块定义 wiki 的内容 schema（人写的契约），LLM 只在契约内填内容。**

---

## 1. 职责与边界
- **做**：定义并执行 wiki schema；Ingest；Lint；维护 raw（只读不可变）、wiki、index、log；git 版本化。
- **不做**：提问/抽取（M2）、生成简历（M4）。LLM 调用经 [M7](M7-llm-orchestration.md)，文件落盘经 [M8](M8-storage-data.md)。

## 2. 目录与文件布局（每用户）
```
users/<user_id>/
  raw/                       # 不可变原始素材，只读
    <yyyy-mm-dd>-<type>-<slug>.md
  wiki/
    experiences/<slug>.md    # 经历页（教育/工作/项目）
    skills/<slug>.md         # 技能/证书/奖项页
    topics/<slug>.md         # 主题摘要/综合页（后续）
  index.md                   # 内容目录
  log.md                     # 时间线
  schema.md                  # 本模块定义的契约（页面规则 + 操作流程）
```
全目录由 git 管理；每次 Ingest/Lint 为一次提交（NFR 可靠性）。

## 3. wiki 内容 schema（契约，必须固化）

### 3.1 通用规则
- 每个页面 = **YAML frontmatter（结构化元数据）+ markdown 正文（叙述）**。
- **双语**：需双语的文本字段统一用 `{ zh: "...", en: "..." }`；正文双语段并排（中文段 + 英文段）。
- **来源**：每条事实/颗粒带 `source`（指向 raw 的 id），承载 GR-1/GR-5。
- **命名**：文件名 kebab-case slug；页面间用 `[[slug]]` 互链。
- LLM **不得新增 schema 之外的字段或页面类型**（违反则 Lint 报警，GR-7）。

### 3.2 经历页 `experiences/<slug>.md`
```yaml
---
id: exp-acme-backend
type: experience            # experience
category: work              # education | work | project
org: { zh: "Acme公司", en: "Acme Inc." }
title: { zh: "后端工程师", en: "Backend Engineer" }
start: 2022-03
end: 2024-06               # 或 present
location: { zh: "上海", en: "Shanghai" }
tags: [backend, fintech, senior]
sources: [raw/2026-06-27-work-acme.md]
---
## 概述 / Overview
（中文叙述）
(English narrative)

## 成就颗粒 / Achievements
- {id: b1, source: raw/...#a, skills: [go, mysql]}
  - zh: 用 Go 重构支付网关，将 P99 延迟从 800ms 降到 120ms。
  - en: Rebuilt the payment gateway in Go, cutting P99 latency from 800ms to 120ms.
```
- 成就颗粒为列表项，**每条带 id / source / skills（关联技能=印证标签，替代建图）/ metrics（可选）**，双语并排。

### 3.3 技能/证书页 `skills/<slug>.md`
```yaml
---
id: skill-go
type: skill                # skill | certificate | award | language
name: { zh: "Go", en: "Go" }
level: { zh: "精通", en: "Expert" }   # 文字分级，不用图形
tags: [backend]
evidenced_by: [exp-acme-backend#b1]   # 被哪些成就印证（标签式关系）
sources: [raw/...]
---
（可选补充说明，双语）
```

### 3.4 index.md
逐页一行：`- [[slug]] (type/category) — 一句话摘要 | tags | start–end`。供 LLM 与用户导航；Query/Interview 读它了解全局。

### 3.5 log.md
按时间追加：`- <ts> <op:interview|ingest|lint> <摘要：触及哪些页面/标记了什么矛盾>`。

## 4. 功能需求
| 编号 | 需求 | 优先级 |
|---|---|---|
| M3-FR-01 | raw sources 不可变保存，唯一事实源（只读）。 | MVP |
| M3-FR-02 | 维护 schema.md 契约（§3）作为 LLM 纪律约束。 | MVP |
| M3-FR-03 | Ingest：将 raw（含 M2 确认条目）编译进 wiki，按 schema 生成/更新经历页与技能页，维护 `[[]]` 交叉引用与标签。 | MVP |
| M3-FR-04 | Ingest 自动把文本翻译为中英双语并存入双语字段。 | MVP |
| M3-FR-05 | 维护 index.md 与 log.md。 | MVP |
| M3-FR-06 | Ingest 遇矛盾（时间冲突/同机构信息不一致/头衔年限不符）标记，不静默覆盖（GR-2）。 | MVP |
| M3-FR-07 | "技能↔成就印证"用 `skills`/`evidenced_by` 标签表达，不建图。 | MVP |
| M3-FR-08 | 每次 Ingest/Lint 提交 git（可回滚）。 | MVP |
| M3-FR-09 | Lint：自检矛盾/过时/孤儿页（无入链）/缺失交叉引用/缺口。 | 后续 |
| M3-FR-10 | 回填：Query 的好结果沉淀为新 topic 页供复用。 | 后续 |
| M3-FR-11 | 提供"wiki 状态摘要"接口（index 概览 + 缺口）供 M2 提问、M4 生成读取。 | MVP |

## 5. 接口与数据
- 输入：M2 的确认条目 / raw。
- 输出：更新后的 wiki 文件、index、log（→ M8 落盘+git）；wiki 状态摘要（→ M2/M4）。
- LLM 调用：Ingest/Lint 经 M7（CWD=用户 wiki 目录）。

## 6. 依赖
- M7（Ingest/Lint 的 CLI 调用）、M8（文件系统+git）。被 M2/M4 读取。

## 7. 验收标准
- ✅ schema.md 固化，Ingest 产出的页面严格符合 §3 字段与格式。
- ✅ 双语字段齐全；颗粒带 id/source/skills。
- ✅ 矛盾被标记而非覆盖；每次编译有 git 提交。
- ✅ index/log 正确更新；能对外提供 wiki 状态摘要。
