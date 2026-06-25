# 方案目录（供横向比较）

> 用途：列出"个人知识库 → 按 JD 生成简历"的**各条独立技术路线**，每条一份独立文档。
> 本文件只做**中性目录与一句话本质**，不做优劣评判——比较由你最后来做。
> 原始需求文档 `REQUIREMENTS.md`（= 方案 A）保持不动，作为基线。

---

## 共同目标（所有方案都要满足）

无论走哪条路线，都服务同一组需求与护栏（详见 `REQUIREMENTS.md`）：

- 个人真实素材沉淀 → 输入目标岗位 JD → 产出针对性、可投递的 PDF 简历。
- 双语（中/英）、简历长度可选（一页/两页）、ATS 友好。
- 护栏 **GR-1 不造假**（内容可溯源到真实素材）为最高红线，GR-2~GR-7 同样适用。
- 小范围分享、数据隔离、走线上 LLM。

各方案的差异**不在目标，而在"用什么载体存知识 + 用什么机制选材生成"**。

---

## 方案清单

| 方案 | 文档 | 一句话本质 | 主要差异轴 |
|---|---|---|---|
| **A** | [`REQUIREMENTS.md`](REQUIREMENTS.md) | LLM Wiki：把素材编译成互链叙事 wiki，查询时全量塞上下文由 LLM 选材 | 叙事知识 / 全量上下文 |
| **B** | [`SOLUTION-B-atom-retrieval.md`](SOLUTION-B-atom-retrieval.md) | 结构化成就原子库 + 检索-打分-预算流水线，LLM 只抽取与最终润色 | 结构化数据 / 检索打分 |
| **C** | [`SOLUTION-C-agentic.md`](SOLUTION-C-agentic.md) | 多智能体流水线，带 Critic 自我校验循环直到通过护栏 | 编排 / 自我校验 |
| **D** | [`SOLUTION-D-knowledge-graph.md`](SOLUTION-D-knowledge-graph.md) | 职业知识图谱：经历/技能/成就建模为图，靠子图匹配选材 | 关系建模 / 图谱 |
| **E** | [`SOLUTION-E-slot-filling.md`](SOLUTION-E-slot-filling.md) | 确定性槽位填充：简历=固定 schema 槽位，规则+标签选材，LLM 仅可选润色 | 确定性 / 最少 AI |
| **F** | [`SOLUTION-F-interview.md`](SOLUTION-F-interview.md) | 访谈驱动：LLM 像教练一样追问，把回答挖成结构化素材 | 获取方式 / 素材质量 |
| **G** | [`SOLUTION-G-master-resume.md`](SOLUTION-G-master-resume.md) | 大师简历 + 裁剪：维护一份全集长简历，每次投递裁剪出针对性子集 | 制品同构 / 直觉心智 |

---

## 建议的比较维度（你最后比较时可参考）

- 真实性保证强度（GR-1 是结构性成立还是尽力而为）
- 选材的可解释性 / 可复现性
- 规模上限（素材长大后是否撞上下文窗口）
- 跨经历综合能力（能否"连点成线"）
- 工程量 / 起步成本
- LLM 调用成本与延迟
- 录入门槛 / 用户心智负担
- 退化与降级行为（素材不足时）
