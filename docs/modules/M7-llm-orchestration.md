# M7 LLM 编排与平台 — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：统一所有 LLM 调用——经 **Claude CLI** 执行四个操作（Interview / Ingest / Query / Lint）。负责调用约定、prompt 模板管理、环境配置、串行/重试/超时、成本与溯源审计。是 M2/M3/M4/M6 共用的平台层。

---

## 1. 职责与边界
- **做**：封装 `claude` CLI 调用；管理固化于 `schema.md` 的 prompt 模板；注入运行时变量；设置 CWD=用户 wiki 目录；串行调度；超时/重试；写审计。
- **不做**：业务语义（属各业务模块）。**不直接对接 LLM SDK/API**，不持久化密钥。

## 2. 调用方式
- **shell out 调用 `claude` CLI 非交互模式**；**用户内容（作答/JD）一律经 stdin/管道传入，禁止拼入命令行参数**（防命令注入，GR-9）。
- **工作目录设为 `wiki/`**（而非用户根目录），从机制上隔离 LLM 对 `raw/` 的写权限（raw 由应用代码写，见 M3-FR-15）。
- **模型**：用 CLI 默认模型（不在代码指定）。
- **环境变量**：API Key / 模型等由**用户手动配置**（如 `ANTHROPIC_*`、CLI 自身配置）；代码只读取环境、不写入、不落库。
- **并发**：第一阶段单用户**串行**（同一时刻 1 个 CLI 任务），避免 2核4G 过载；排队时前端给"排队中/可取消"提示。
- **JSON 输出协议**：需结构化输出的操作（Interview 抽取、Query）在系统 prompt 约定"仅输出单个 JSON"；解析时提取首个 `{` 到末个 `}`、容错去围栏；解析失败重试 N 次，仍失败走 GR-6。
- **CLI 版本**：锁定/记录所用 `claude` CLI 版本（外部工具升级可能改输出/默认模型），纳入兼容性检查。

## 3. 四个操作
| 操作 | 调用方 | CWD | 输入 | 输出 |
|---|---|---|---|---|
| Interview | M2 | 用户wiki | index概览/covered/skipped/target | focus+question 或 done |
| Ingest | M2/M3 | `wiki/` | 确认条目/raw | 更新 wiki/index/log + 矛盾标记 |
| Query | M4 | `wiki/` | 全量wiki+JD+长度/语言/地区 | 结构化简历数据(JSON Resume+sidecar) |
| Lint（后续） | M3 | `wiki/` | 全量wiki | 问题清单 |

> CWD 统一为 `wiki/`；Interview 行 CWD 同样为 `wiki/`。

## 4. 功能需求
| 编号 | 需求 | 优先级 |
|---|---|---|
| M7-FR-01 | 提供统一调用封装：拼装 prompt（模板+注入变量）、设 CWD、执行 CLI、收集输出。 | MVP |
| M7-FR-02 | prompt 模板集中管理并固化于 `schema.md`（Interview a–d / Ingest / Query / Lint）。 | MVP |
| M7-FR-03 | 从环境变量读取 LLM 配置；缺失则明确报错指引用户配置，不内置密钥。 | MVP |
| M7-FR-04 | 调用串行化（队列），单用户同一时刻仅 1 个 CLI 任务。 | MVP |
| M7-FR-05 | 每操作给**超时默认值**（Query 读全量 wiki 给更宽松上限）+ 有限次重试；失败返回结构化错误，触发上层 GR-6 降级，不丢已有数据。 | MVP |
| M7-FR-06 | 审计：每次调用记录 操作类型/模型/耗时/成本/溯源/状态 → `audit_log`。成本获取方式（CLI 是否回传 token/成本，否则估算）需在实现时确定。 | MVP |
| M7-FR-07 | JSON 输出协议与容错解析（见 §2）：提取 `{...}`、去围栏、失败重试再降级。 | MVP |
| M7-FR-08 | 用户内容经 stdin 传入、禁止拼命令行（GR-9）；CWD=`wiki/` 隔离 raw。 | MVP |
| M7-FR-09 | 记录/锁定 `claude` CLI 版本；排队 UX（提示/可取消）。 | MVP |
| M7-FR-10 | 多用户并发调度与限流。 | 后续 |

## 5. 接口与数据
- 输入：`{operation, user_id, vars}`。输出：`{ok, data|error, audit}`。
- 数据：`audit_log`（→ [M8](M8-storage-data.md)）。

## 6. 依赖
- 依赖宿主机已安装并配置 `claude` CLI 与 Typst；M8（审计）。被 M2/M3/M4/M6 调用。

## 7. 验收标准
- ✅ 四操作均能经 CLI 在用户 wiki 目录上执行并返回预期输出。
- ✅ 密钥仅来自环境变量、代码无硬编码。
- ✅ 串行生效；失败有重试与结构化错误；每次调用有审计记录。
