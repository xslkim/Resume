# 智能简历生成服务 — 框架与模块地图

> **本文件是框架/指导性文档**：描述整体定位、架构、模块划分与横切约束。
> 各模块的**详细软件需求**见 `docs/modules/` 下的独立文档（见 §8 文档索引）。

---

## 1. 项目概述

### 1.1 定位
**个人素材沉淀库（LLM Wiki）+ 问答驱动录入 + 工作描述驱动的选材综合 + PDF 排版输出**。
一句话：**用问答帮你把真实经历沉淀进一个由 LLM 维护的个人 wiki，告诉我目标岗位，我给你一份针对性简历。**

### 1.2 核心价值
- **低门槛录入**：以"系统提问、你作答"为主，不用对着空白表格发愁。
- **针对性**：同一批素材，针对不同岗位组合出不同侧重与详略。
- **真实性**：所有内容可追溯到真实素材，不凭空编造（最高红线，见 §5 与 M6）。
- **复利积累**：知识库越用越丰富、越用越准。
- **ATS 友好**：输出可被招聘系统解析、关键词达标。

### 1.3 使用范围
小范围分享，**第一阶段仅本人单用户**；非公网规模化服务。

---

## 2. 术语

| 术语 | 定义 |
|---|---|
| Raw sources（原始素材） | 用户录入的不可变副本，系统**唯一事实源**，LLM 只读不写。 |
| Wiki（个人职业 wiki） | LLM 基于 raw sources 增量编译并维护的、结构化、互链的 markdown 页面集合。 |
| Schema（规则文件） | 规定 wiki 如何组织、各操作遵循什么流程的配置（人写的契约，见 M3）。 |
| 成就颗粒（Bullet） | 最小可复用单元：一条可独立挑选、量化的成就。 |
| 标签（Tag） | 附加在页面/颗粒上的分类（技能/行业/资深度/软技能）。 |
| 问答会话（Interview） | 系统经 Claude CLI 提问、用户作答，驱动素材录入的过程（M2）。 |
| Ingest（编译） | 把新素材整合进 wiki：更新页面、维护交叉引用、双语翻译、标记矛盾（M3）。 |
| Query（生成） | 输入 JD 后，LLM 读全量 wiki 选材、综合出简历草稿（M4）。 |
| Lint（健康检查） | LLM 自检 wiki：矛盾、过时、孤儿页、缺口（M3）。 |
| JD | 目标岗位描述文本，生成的触发与选材依据。 |
| 画像（Profile） | 预设的某一求职方向及默认选材倾向（后续）。 |
| 草稿 / 定稿 | 草稿可反复局部重生成；定稿为可导出 PDF 的稳定版本。 |
| 护栏（Guardrail，GR） | 强制约束生成不越界的规则（见 §5 / M6），核心是"不造假"。 |

---

## 3. 总体架构

### 3.1 分层与数据流
```
用户
 │  问答/录入                      JD
 ▼                                 ▼
[M2 素材录入] ──raw──> [M3 知识库引擎(LLM Wiki)] <──全量wiki── [M4 JD分析与生成]
                              │  文件+git                          │ 结构化简历数据
                              ▼                                    ▼
                        [M8 存储与数据模型]                  [M5 渲染与输出] ──> PDF/DOCX
                                                                   ▲
[M6 校验与护栏] 贯穿 M2(确认)/M3(矛盾)/M4(生成后)/M5(ATS)
[M7 LLM编排] 所有 LLM 调用统一经 Claude CLI（Interview/Ingest/Query/Lint）
[M1 用户与账号] 数据隔离
```

### 3.2 知识库形态（LLM Wiki，自建轻量）
知识库 = **一套 markdown 目录约定 + schema + 各操作 prompt 模板 + 调 Claude CLI 的薄编排**，用 **git** 做版本历史。不引入向量库/图库/RAG 框架。详见 **M3**。

### 3.3 部署
**应用服务器 2核4G**（编排 + Claude CLI + Typst + 本地文件/git）**+ 独立 MySQL 服务（不在本机）**。重活（LLM 推理）在远端经 CLI 调用；本机只做编排、文件 IO、PDF 渲染。详见 **M7 / M8**。

### 3.4 LLM 调用
后端不直接对接 LLM SDK，而是 **shell out 调用 `claude` CLI**（非交互模式），工作目录设为用户 wiki 目录。**模型用 CLI 默认模型；密钥/模型由用户手动配置环境变量，代码不持久化。** 详见 **M7**。

---

## 4. 功能模块地图

| 模块 | 职责（一句话） | 关键依赖 | MVP | 文档 |
|---|---|---|---|---|
| M1 用户与账号 | 登录、数据隔离 | — | ✓ | [M1](modules/M1-user-auth.md) |
| M2 素材录入 | 问答驱动（核心）+ 手动录入/编辑 | M3,M6,M7 | ✓ | [M2](modules/M2-intake.md) |
| M3 知识库引擎 | LLM Wiki：schema/ingest/lint、文件+git | M7,M8 | ✓ | [M3](modules/M3-knowledge-base.md) |
| M4 JD 分析与生成 | JD 抽取、Query 选材润色、结构化数据、缺口 | M3,M6,M7 | ✓ | [M4](modules/M4-generation.md) |
| M5 渲染与输出 | 设计令牌、Typst 模板、PDF/DOCX | M8 | ✓ | [M5](modules/M5-rendering.md) |
| M6 校验与护栏 | 后置校验、一致性、溯源、GR 执行 | M3,M7 | ✓ | [M6](modules/M6-guardrails.md) |
| M7 LLM 编排与平台 | Claude CLI 调用、env、串行、重试、审计 | M8 | ✓ | [M7](modules/M7-llm-orchestration.md) |
| M8 存储与数据模型 | 文件/git 布局 + MySQL 表 | — | ✓ | [M8](modules/M8-storage-data.md) |

**契约由人定、内容由 LLM 填**：M3 的 wiki schema、M5 的设计规格与数据契约（JSON Resume）、M7 的操作 prompt 均为**人写的固定契约**；LLM 只在契约内产出内容（不得自行改变结构，否则违反 GR-7）。

---

## 5. 横切关注点（贯穿所有模块）

### 5.1 LLM 经 Claude CLI（M7）
四个操作 Interview / Ingest / Query / Lint 全部是对用户 wiki 目录运行的 Claude CLI 任务；prompt 模板固化于 `schema.md`；调用串行；每次调用写审计（模型/成本/溯源）。

### 5.2 护栏（M6）
| 编号 | 护栏 |
|---|---|
| GR-1 | 不造假 / Grounded：内容可追溯到 raw/wiki；每条 bullet 挂来源 id；生成后一次性核查。 |
| GR-2 | 一致性：时间不冲突、同机构信息一致、头衔与年限自洽；冲突报警不篡改。 |
| GR-3 | 量化优先：问答主动逼出数字；纯定性提示量化。 |
| GR-4 | 关键词覆盖：JD 关键技能词合理出现，不堆词、不失实。 |
| GR-5 | 可解释：每段内容可回溯来源页面。 |
| GR-6 | 降级：素材不足或 LLM/CLI 失败时，宁可输出不完整但真实的版本；必要时回退规则填充 ATS 模板。 |
| GR-7 | wiki 与 raw 一致：wiki 是 raw 的忠实编译；Lint 修复漂移。 |
| GR-8 | 诱导防护：问答/提示式抽取必须回显确认才入库；提问"澄清而非诱导"。 |

### 5.3 存储（M8）
raw + wiki + git 在**本机文件系统**（事实源）；结构化数据在**独立 MySQL**；二进制（PDF/图片）本机、MySQL 存指针；备份可选（私有 GitHub remote）。

### 5.4 双语
用户单语录入；Ingest 阶段由 LLM 自动翻译成中英双语存入 wiki；Query 按 JD 语言输出中/英/双语。

### 5.5 降级
任一 LLM/CLI 失败不丢已确认数据（git 提交/会话可续）；生成失败回退 GR-6 真实但不完整的输出。

---

## 6. MVP 范围（按模块）

- **M1**：单用户登录、数据隔离。
- **M2**：问答驱动录入（提问→作答→抽取双语→回显确认→入库）+ 手动录入/编辑。
- **M3**：wiki schema 固化；Ingest 编译 + 双语 + index/log；矛盾标记。
- **M4**：JD 关键词抽取；全量上下文 Query 选材润色；可选长度/语言；结构化数据（含来源 id）；轻量缺口。
- **M5**：一套 ATS 友好单栏 Typst 模板，出中/英/双语 PDF（照片默认关）。
- **M6**：生成后一次性后置校验（至少 GR-1 溯源）。
- **M7**：Claude CLI 串行调用四操作；env 手动配置；审计。
- **M8**：文件/git 布局；MySQL 核心表。

> 成功标准：经问答录入若干真实素材并编译后，输入一个真实 JD，数分钟内得到**内容真实、针对该岗位、可投递**的 ATS PDF。

---

## 7. 总体验收（端到端）

1. ✅ 单用户登录，raw/wiki 隔离（M1）。
2. ✅ 系统提问→用户自然语言作答→抽取回显确认→入库（M2）。
3. ✅ Ingest 编译进 wiki 并自动双语，生成 index（M3）。
4. ✅ 输入真实 JD，可选简历长度（M4）。
5. ✅ Query 全量上下文选材润色，未引入素材外事实（M4+M6）。
6. ✅ Typst 渲染出 ATS 友好中/英/双语 PDF（M5）。
7. ✅ 每条经历可溯源（bullet 带来源 id）（M6）。
8. ✅ 全流程 LLM 经 Claude CLI，跑在 2核4G + 独立 MySQL（M7+M8）。

---

## 8. 文档索引

- 框架（本文件）：`REQUIREMENTS.md`
- [M1 用户与账号](modules/M1-user-auth.md)
- [M2 素材录入](modules/M2-intake.md)
- [M3 知识库引擎](modules/M3-knowledge-base.md)
- [M4 JD 分析与生成](modules/M4-generation.md)
- [M5 渲染与输出](modules/M5-rendering.md)
- [M6 校验与护栏](modules/M6-guardrails.md)
- [M7 LLM 编排与平台](modules/M7-llm-orchestration.md)
- [M8 存储与数据模型](modules/M8-storage-data.md)
