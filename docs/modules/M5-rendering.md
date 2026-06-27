# M5 渲染与输出 — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：把 [M4](M4-generation.md) 的结构化简历数据，经**确定性渲染**（Typst）输出为 ATS 友好的 PDF（后续 DOCX）。本模块定义**设计规格、设计令牌、Typst 模板**——内容层不碰排版。

---

## 1. 职责与边界
- **做**：设计规格与令牌、Typst 模板、JSON Resume→PDF 渲染、ATS 规则强制、照片/纸张/语言处理、文件落盘元数据。
- **不做**：内容选材/润色（M4）。**LLM 不参与排版**。

## 2. 设计原则
简历须同时被 **ATS 解析**与**人审阅**；冲突时**默认偏 ATS**（ATS 安全为底线，再求美观）。强设计感模板（多栏/图形）为"人审版"，与 ATS 版分开。内容层产数据、渲染层产版式，二者分离。

## 3. 功能需求
| 编号 | 需求 | 优先级 |
|---|---|---|
| M5-FR-01 | 结构化数据 → PDF（Typst 渲染）。 | MVP |
| M5-FR-02 | 至少一套 ATS 友好单栏模板（文字可解析、关键词达标、照片默认关）。 | MVP |
| M5-FR-03 | 按输出语言出中/英/双语；**纸张由目标地区决定**（国内 A4，北美 Letter，缺省 A4，见 §5b），可覆盖。**注：纸张随地区而非语言**。 | MVP |
| M5-FR-04 | **地区规则驱动 basics**：按目标地区决定渲染哪些敏感字段（照片/年龄/性别/婚姻/政治面貌）——投北美等地一律不渲染（GR-10），国内可渲染。照片开关由地区规则驱动，非裸开关。纯黑白开关。 | MVP |
| M5-FR-05 | 渲染完整 basics（姓名/电话/邮箱/城市/headline + 按地区的 links），不只 name。 | MVP |
| M5-FR-06 | 设计令牌参数化（§6），换风格=改令牌/换布局。 | MVP |
| M5-FR-07 | 长度控制闭环：渲染后检测页数溢出，**回传溢出量给 M4**缩减重渲染（限最大重试次数，如 2 次）。 | MVP |
| M5-FR-08 | 降级（GR-6，**最小降级**避免重造选材器）：LLM 不可用时，优先渲染**最近一次成功的定稿**；若无，则渲染 **basics + 全部 bullet 原样平铺**的保守版。完整规则版选材为后续。 | MVP |
| M5-FR-09 | 多套模板（技术/设计/学术/商务）。 | 后续 |
| M5-FR-10 | 导出 DOCX。 | 后续 |
| M5-FR-11 | 求职信（Cover Letter）。 | 后续 |

## 4. 版式与排版规格
- **单栏**（MVP）。区块顺序：Header → Summary → Experience → Projects → Skills → Education → 证书/奖项/语言（应届生 Education 提前）。标准区块标题。
- 页边距 1.8cm；每段经历 3–6 条 bullet，逆时序。
- 字体：拉丁 Inter（衬线模式 EB Garamond/思源宋体），CJK 思源黑体；**字体必须嵌入**；中英混排靠字体回退。
- 字号：姓名 20pt / 区块标题 13pt（加粗+细分隔线）/ 职位 11pt 加粗 / 正文 10.5pt / 辅助 9.5pt 灰。行距 1.25。
- 配色：正文 #1a1a1a；单一强调色 #1f3a5f（仅姓名/标题/分隔线）；禁止文字下铺色块。
- 图片：照片默认关、开启时右上角统一裁剪；联系方式纯文本（图标仅装饰、可缺省）；无 logo/进度条/星级，技能用文字分级。

## 5. ATS 硬规则与细节（渲染层强制，GR-4）
- 单栏；真文本 PDF（不得把文字渲染成图片）；嵌入标准字体；标准区块标题；关键信息放正文区不放页眉页脚；不用表格/文本框承载关键内容。
- **日期**：统一 `YYYY.MM – YYYY.MM`；`end=present` 渲染为 `至今 / Present`。
- **应届生判定**（Education 提前）：依据无全职 work 经历或最近毕业时间在阈值内；判定来源为 basics/experiences 字段，不臆测。
- **文件名** `姓名_岗位_简历.pdf` 中"岗位"取 `_meta.params.headline` 或 JD 解析岗位，缺失时回退 headline。

## 5b. 地区内容规则表（GR-10，与 M4 共用）
| 地区(枚举) | 照片/年龄/性别/婚姻/政治面貌 | 纸张 | 长度倾向 | links |
|---|---|---|---|---|
| `na` 北美(US/CA) | **禁用全部敏感字段**（反歧视） | Letter | 偏 1 页 | LinkedIn/GitHub |
| `cn` 国内 | 常用照片/年龄/性别，部分场景政治面貌 | A4 | 1–2 页 | 可选 |
| `eu` 欧洲 | 多数禁照片/年龄（含 GDPR 倾向），按国细化 | A4 | 1–2 页 | LinkedIn |
| `other` 其他 | 按当地规范（默认从严） | A4 | — | — |

## 6. 设计令牌
```yaml
paper: A4            # A4 | Letter
margin: 1.8cm
density: standard    # compact | standard
font: { latin: "Inter", cjk: "Source Han Sans SC", serif_mode: false }
accent_color: "#1f3a5f"
photo: false
icons: minimal       # none | minimal
section_order: [summary, experience, projects, skills, education, extras]
section_titles_locale: zh | en | bilingual
```

## 7. 数据契约（输入）
对齐 JSON Resume（见 [M4](M4-generation.md) §3）。映射：`basics`→Header、`summary`→Summary、`work[]/projects[]`→经历（`highlights`=bullet）、`skills[]`→技能（文字分级）、`education[]`→教育、`certificates[]/awards[]/languages[]`→附加区。**只渲染主数据；并行 `_meta`（`source/lang/gaps`，统一无下划线前缀）不渲染入正文**，供 M6/前端用。
- **basics 敏感字段职责分工（P2-7）**：M4 生成时按地区**主责不输出**越界字段；**M5 渲染层强制过滤为兜底**；M6 仅做合规校验报告。三层为防御性重复，主责在 M4。

## 8. 渲染器与 MVP 模板
- 渲染器 **Typst**（单二进制、低内存、真文本 PDF、支持图片，契合 2核4G）；可参考 **RenderCV**。
- **MVP 模板（标准/商务 ATS）`templates/standard.typ`** 规格骨架：
```typst
#let resume(data, t) = {
  set document(title: data.basics.name + "_简历")
  set page(paper: t.paper, margin: t.margin)
  set text(font: (t.font.latin, t.font.cjk), size: 10.5pt, fill: rgb("#1a1a1a"))
  // Header
  text(size: 20pt, weight: "bold", data.basics.name)
  // 联系方式：纯文本一行
  // 区块：按 t.section_order 渲染；标题 13pt 加粗 + 0.5pt 分隔线，强调色 accent
  // work[]：org/title/date(右对齐灰) + highlights 列表(10.5pt)
  // skills[]：分组 + 文字分级
  // 照片：t.photo 为 true 时右上角插入
}
```
- 渲染流程：`结构化数据(json) + 令牌(yaml) → Typst → PDF`（确定性、可复现）。
- **落地前复核**：思源/Inter 字体在目标 Typst 版本的嵌入与 fallback、RenderCV 可定制度与许可证。

## 9. 依赖
- 输入 M4 结构化数据；输出文件落 M8（本机），MySQL 存指针。

## 10. 验收标准
- ✅ 结构化数据渲染出 ATS 友好中/英/双语 PDF，文字可选中、排版整洁。
- ✅ 照片默认关、可开；纸张**随地区**（na→Letter，cn/eu/other→A4）。
- ✅ 令牌可改变风格；LLM 不可用时规则降级仍能出 PDF。
