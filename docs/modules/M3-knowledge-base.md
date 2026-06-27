# M3 知识库引擎（LLM Wiki） — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：维护个人职业 wiki。把 raw 素材**编译（Ingest）**进结构化、互链的 markdown 页面，自动双语，维护 index/log，标记矛盾；定期 **Lint** 自检。**本模块定义 wiki 的内容 schema（人写的契约），LLM 只在契约内填内容。**

---

## 1. 职责与边界
- **做**：定义并执行 wiki schema；Ingest；Lint；维护 raw（只读不可变）、wiki、index、log；git 版本化。
- **不做**：提问/抽取（M2）、生成简历（M4）。LLM 调用经 [M7](M7-llm-orchestration.md)，文件落盘经 [M8](M8-storage-data.md)。

## 2. 目录与文件布局（每用户，唯一权威布局）
```
users/<user_id>/             # git 仓库根（应用代码管理提交）
  raw/                       # 不可变原始素材；应用代码写，LLM 不读不写（在 wiki/ 外）
    <yyyy-mm-dd>-<type>-<slug>.md
  outputs/<generation_id>/   # 生成的 PDF/DOCX（见 M8）
  uploads/                   # 上传图片/证件照（见 M8）
  wiki/                      # CLI 的 CWD；LLM 在此读写
    experiences/<slug>.md    # 经历页（教育/工作/项目）
    skills/<slug>.md         # 技能/证书/奖项页
    topics/<slug>.md         # 主题摘要/综合页（后续）
    profile/basics.md        # 个人信息页（§3.6）
    index.md                 # 内容目录（§3.4）
    schema.md                # 契约：页面规则 + 操作 prompt（M7-FR-02）
    issues.md                # 矛盾记录（§3.7）
    log.md                   # 时间线（§3.5）
```
- **`raw/` 在 `wiki/` 之外**：CLI 的 CWD=`wiki/`，物理上接触不到 `raw/`（机制隔离 GR-1，配合 M7）。
- **wiki/ 内所有契约文件就地可读**（CWD=`wiki/` 时无需跨级）。
- git 仓库根在 `users/<user_id>/`；**一次"确认→入库"由应用层做原子提交**（raw+wiki 一起，见 M3-FR-08），可回滚到一致快照。

## 3. wiki 内容 schema（契约，必须固化）

### 3.1 通用规则
- 每个页面 = **YAML frontmatter（结构化元数据）+ markdown 正文（叙述）**。
- **双语**：需双语的文本字段统一用 `{ zh: "...", en: "..." }`；正文双语段并排（中文段 + 英文段）。
- **来源**：每条事实/颗粒带 `source`（指向 raw 的 id），承载 GR-1/GR-5。
- **命名**：文件名 kebab-case slug；页面间用 `[[slug]]` 互链。
- LLM **不得新增 schema 之外的字段或页面类型**（违反则 Lint 报警，GR-7）。
- **raw 写入机制（红线）**：`raw/` 由**应用代码在用户确认后写入**，LLM 只读不写；Ingest/Query 的 CLI **工作目录设为 `wiki/`**（而非用户根目录），从机制上隔离 LLM 对 raw 的写权限（配合 M7）。
- **内容时效**：含时间属性的内容默认带"最后相关时间"意识；远期经历/过时技能在 Query 选材中**默认降权**（见 M4），陈旧内容不污染生成。
- **raw 内部结构与锚点（溯源最后一跳）**：raw 文件为 markdown + YAML frontmatter（`id/type/date/source_kind`）；正文按"原子事实块"切分，每块带块锚点 `^a/^b/...`（应用代码在写入时分配）。bullet 的 `source: <raw文件>#a` 即指向该块，`verify_against_raw`（M6）据此做结构化匹配。
- **slug 生成与路径安全**：slug 由内容标题转写（中文取拼音，附短哈希去重），仅允许 `[a-z0-9-]`，长度上限（如 50）；清洗 `..`/`/` 防路径遍历。
- **Ingest 的输入来自应用经 stdin 传入的"已确认条目"**，Ingest（CWD=wiki/）不读 raw；raw 与 wiki 由同一份确认条目分别落盘 → 二者同源一致（GR-7 的 MVP 保证方式）。

### 3.2 经历页 `experiences/<slug>.md`
frontmatter 与成就颗粒均为**结构化对象**（修正旧示例的行内 flow / block 混排）：
```yaml
---
id: exp-acme-backend
type: experience
category: work                       # education | work | project
employment_type: full_time          # full_time | intern | part_time | contract | freelance
org: { zh: "Acme公司", en: "Acme Inc." }
title: { zh: "后端工程师", en: "Backend Engineer" }
start: 2022-03
end: 2024-06                         # 日期或 present
location: { zh: "上海", en: "Shanghai" }
team_size: 8                         # 可选，影响成就份量
reports_to: { zh: "技术总监", en: "Engineering Director" }  # 可选
tags: [backend, fintech, senior]
sources: [raw/2026-06-27-work-acme.md]
bullets:                             # 成就颗粒（结构化对象数组）
  - id: b1
    source: raw/2026-06-27-work-acme.md#a
    skills: [go, mysql]
    star:                            # 可选 STAR/CAR 结构，action/result 分离
      action: { zh: "用 Go 重构支付网关", en: "Rebuilt the payment gateway in Go" }
      result: { zh: "P99 延迟 800ms→120ms", en: "cut P99 latency 800ms→120ms" }
    metrics: [{ name: p99_latency, from: 800, to: 120, unit: ms }]   # 可选
    text: { zh: "用 Go 重构支付网关，将 P99 延迟从 800ms 降到 120ms。",
            en: "Rebuilt the payment gateway in Go, cutting P99 latency from 800ms to 120ms." }
---
## 概述 / Overview
（中文叙述）
(English narrative)
```
- 成就颗粒 = `{id, source, skills, star?, metrics?, text:{zh,en}}`。**允许纯定性 bullet**（无 metrics）；`skills` 即"技能↔成就印证"标签（替代建图）。
- bullet 质量规则（GR-3，详见 M4 §7 内容质量规范）：优先真实可量化结果；禁止空泛动词堆砌（"负责/参与了…"而无结果）；指标按价值梯度（营收/成本 > 用户/规模 > 速度/效率）。**"代码行数"等是公认坏指标，禁止作为成就量化**。

### 3.3 技能/证书页 `skills/<slug>.md`
```yaml
---
id: skill-go
type: skill                 # skill | certificate | award | language（type=内容类型）
name: { zh: "Go", en: "Go" }
level: { zh: "精通", en: "Expert" }   # 文字分级，不用图形
years: 4                     # 经验年限（ATS 常抓）
last_used: 2024-06           # 最后使用时间（时效/降权用）
tags: [backend]             # tag=横切分类（技能/行业/资深度/软技能），与 type 不同维度
evidenced_by: [exp-acme-backend#b1]   # 被哪些成就印证
sources: [raw/...]
---
（可选补充说明，双语）
```
- **level × 印证一致性**：自评 `level` 高（如"精通"）但 `evidenced_by` 为空 → Ingest 标记提示（GR-2），避免"印证机制形同虚设"。
- **保守自评纪律**：`level` 按"**能通过目标岗位面试追问**"的标准，宁可下调一档——写"精通"却面试露馅比不写更伤（见 M4 §7）。`years` 不可超过经历时间跨度。
- **软技能不单列空泛词**：用具体成就体现（与 `evidenced_by` 呼应），不写"优秀的沟通能力"这类空话。
- **type vs tag**：`type` 是内容类型（skill/certificate/award/language），`tag` 是横切分类，二者正交。

### 3.4 index.md
逐页一行：`- [[slug]] (type/category) — 一句话摘要 | tags | start–end`。供 LLM 与用户导航；Query/Interview 读它了解全局。

### 3.5 log.md
按时间追加：`- <ts> <op:interview|ingest|lint> <摘要：触及哪些页面/标记了什么矛盾>`。

### 3.6 个人信息页 `profile/basics.md`（必备，简历地基）
```yaml
---
id: basics
type: basics
name: { zh: "张三", en: "San Zhang" }
email: zhang@example.com          # 必选
phone: "+86 138..."               # 必选
location: { zh: "上海", en: "Shanghai" }   # 城市即可，必选
headline: { zh: "后端工程师", en: "Backend Engineer" }  # 求职意向/目标岗位
links:                            # 按地区/行业可选
  linkedin: ...
  github: ...
  website: ...
  scholar: ...                    # 研究岗
# —— 敏感字段：仅在"目标地区"允许时填用（GR-10，见 M4/M5）——
photo: null                       # 证件照路径，默认空
birth: null                       # 年龄/出生
gender: null
marital: null
political: null                   # 政治面貌（部分国内场景）
sources: [raw/...]
---
```
- **必选**：name/email/phone/location/headline。**按地区**：photo/birth/gender/marital/political（投北美等地**禁用**，国内常用）、linkedin/github（海外/技术岗常用）。
- M2 录入须覆盖必选 basics；M4 选用、M5 渲染均受"目标地区规则"约束。

### 3.7 矛盾标记的生命周期
- 矛盾存于独立 `wiki/issues.md`（每条：`{id, 类型, 涉及页面, 描述, 状态:open|resolved, 时间}`）。
- Ingest 发现即记 `open`；用户处理后置 `resolved`；Query 对 `open` 矛盾给出提示，不静默采用冲突数据。

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
| M3-FR-08 | **一次"确认→入库"由应用层做一次原子 git 提交**（`git add raw/ wiki/` 后 commit，单次、可回滚到一致快照）；提交方是应用，不是 CLI。 | MVP |
| M3-FR-09 | Lint：自检矛盾/过时/孤儿页（无入链）/缺失交叉引用/缺口。**GR-7 在 MVP 靠"raw 与 wiki 同源于同一份确认条目"保证一致，系统级漂移检测随本项在后续引入。** | 后续 |
| M3-FR-10 | 回填：Query 的好结果沉淀为新 topic 页供复用。 | 后续 |
| M3-FR-11 | 提供"wiki 状态摘要"接口（index 概览 + 缺口 + **字符数/段数估算**，不依赖 token 分词）供 M2 提问、M4 生成读取与规模判断。 | MVP |
| M3-FR-12 | 维护 basics 页（§3.6），区分必选/按地区敏感字段。 | MVP |
| M3-FR-13 | 经历页支持 employment_type/team_size/reports_to；技能页支持 years/last_used/level×印证一致性提示。 | MVP |
| M3-FR-14 | 成就颗粒按 §3.2 结构（star?/metrics?/text），允许纯定性；执行 bullet 质量规则。 | MVP |
| M3-FR-15 | raw 由应用代码在用户确认后写入，LLM 不写 raw；Ingest CWD=`wiki/`、输入为应用经 stdin 传入的已确认条目。 | MVP |
| M3-FR-16 | 矛盾以 `wiki/issues.md` 记录并维护 open/resolved 生命周期（§3.7）。 | MVP |
| M3-FR-17 | 规定 raw 文件内部结构与块锚点（§3.1），供 `verify_against_raw` 结构化匹配。 | MVP |
| M3-FR-18 | **软删除跳过机制**：被软删的 raw 条目带 `deleted: true`（或 tombstone），Ingest 须读取并**跳过**，避免重编译把已删内容编回 wiki。 | MVP |
| M3-FR-19 | slug 生成与路径清洗规则（§3.1）。 | MVP |

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
