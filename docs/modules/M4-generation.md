# M4 JD 分析与简历生成 — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：解析 JD；读**全量 wiki**（[M3](M3-knowledge-base.md)）选材、定详略、润色；输出**结构化简历数据**（对齐 JSON Resume，含来源 id）交 [M5](M5-rendering.md) 渲染；产出缺口分析；管理草稿。

---

## 1. 职责与边界
- **做**：JD 关键词抽取；Query（全量上下文选材+润色）；长度/语言控制；结构化数据输出；缺口分析；草稿/迭代。
- **不做**：PDF 排版（M5）、真实性核查的执行（M6，但 M4 须配合输出来源 id）。LLM 调用经 [M7](M7-llm-orchestration.md)。

## 2. 功能需求

### 2.1 JD 分析
| 编号 | 需求 | 优先级 |
|---|---|---|
| M4-FR-01 | 输入 JD 全文文本。 | MVP |
| M4-FR-02 | 抽取关键词（技能/资深级别/行业/硬性要求/加分项/JD 语言）。 | MVP（仅关键词）/后续（完整） |
| M4-FR-03 | 计算 wiki 对 JD 的匹配度/覆盖率。 | 后续 |

### 2.2 生成（Query，全量上下文）
| 编号 | 需求 | 优先级 |
|---|---|---|
| M4-FR-04 | 读全量 wiki 选材（相关性 + 标签过滤）。 | MVP |
| M4-FR-05 | 按相关性定详略，受所选长度约束（一页/两页/精简）。 | MVP |
| M4-FR-06 | 润色改写贴合 JD 用词，内容必须基于真实素材（GR-1）。 | MVP |
| M4-FR-07 | 按 JD 优先级重排技能区。 | MVP |
| M4-FR-08 | 按 JD 语言输出中文/英文/双语（wiki 已存双语）。 | MVP |
| M4-FR-09 | 输出**结构化简历数据**（对齐 JSON Resume），每条 bullet 附来源 id（`_source`）与语言（`_lang`）。 | MVP |
| M4-FR-10 | 缺口分析：列出 JD 要求但 wiki 薄弱/缺失的点（生成时同时输出未覆盖需求）。 | MVP（轻量） |
| M4-FR-11 | 按 JD 重写 Summary/Objective。 | 后续 |

### 2.3 草稿与迭代
| 编号 | 需求 | 优先级 |
|---|---|---|
| M4-FR-12 | 生成先以草稿呈现、可预览。 | MVP |
| M4-FR-13 | 草稿确认转定稿。 | MVP |
| M4-FR-14 | 草稿局部重生成（"这段再改"/"换条经历"）。 | 后续 |
| M4-FR-15 | 同一 JD 多次生成保留版本历史、可对比/回滚。 | 后续 |

## 3. 输出数据契约（→ M5）
对齐 **JSON Resume** schema，核心字段：`basics`（双语）、`summary`、`work[]/projects[]`（`highlights[]`=成就 bullet）、`skills[]`（文字分级）、`education[]`、`certificates[]/awards[]/languages[]`。
**扩展字段**：每个条目/highlight 带 `_source`（来源 page/raw id）与 `_lang`；顶层带 `_gaps`（缺口列表）、`_params`（length/language/template）。

## 4. 关键流程
```
JD 文本 → 抽关键词
     → Query(CLI, CWD=用户wiki)：读全量 wiki + JD + 长度/语言
        → 选材 + 润色 → 结构化数据(JSON Resume + _source/_lang + _gaps)
     → 草稿预览 → (M6 后置校验) → 定稿 → (M5 渲染)
```

## 5. 依赖
- M3（读全量 wiki + 状态摘要）、M7（Query 的 CLI 调用）、M6（生成后校验）、M8（JD/生成记录）。

## 6. 验收标准
- ✅ 输入真实 JD，可选长度/语言，得到结构化简历数据。
- ✅ 选材润色未引入素材外事实（配合 M6 溯源核查通过）。
- ✅ 每条 bullet 带来源 id；输出含轻量缺口列表。
- ✅ 草稿可预览、可定稿。
