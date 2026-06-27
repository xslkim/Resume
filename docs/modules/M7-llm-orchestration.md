# M7 LLM 编排与平台 — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：统一所有 LLM 调用——经 **Claude CLI** 执行四个操作（Interview / Ingest / Query / Lint）。负责调用约定、prompt 模板管理、环境配置、串行/重试/超时、成本与溯源审计。是 M2/M3/M4/M6 共用的平台层。

---

## 1. 职责与边界
- **做**：封装 `claude` CLI 调用；管理固化于 `schema.md` 的 prompt 模板；注入运行时变量；设置 CWD=用户 wiki 目录；串行调度；超时/重试；写审计。
- **不做**：业务语义（属各业务模块）。**不直接对接 LLM SDK/API**，不持久化密钥。

## 2. 调用方式
- **shell out 调用 `claude` CLI 非交互模式**（如 `claude -p "<prompt>"` 或管道输入），**工作目录设为 `users/<user_id>/`**，让代理直接读写其中 markdown。
- **模型**：用 CLI 默认模型（不在代码指定）。
- **环境变量**：API Key / 模型等由**用户手动配置**（如 `ANTHROPIC_*`、CLI 自身配置）；代码只读取环境、不写入、不落库。
- **并发**：第一阶段单用户**串行**（同一时刻 1 个 CLI 任务），避免 2核4G 过载。

## 3. 四个操作
| 操作 | 调用方 | CWD | 输入 | 输出 |
|---|---|---|---|---|
| Interview | M2 | 用户wiki | index概览/covered/skipped/target | focus+question 或 done |
| Ingest | M2/M3 | 用户wiki | 确认条目/raw | 更新 wiki/index/log + 矛盾标记 |
| Query | M4 | 用户wiki | 全量wiki+JD+长度/语言 | 结构化简历数据(JSON Resume) |
| Lint（后续） | M3 | 用户wiki | 全量wiki | 问题清单 |

## 4. 功能需求
| 编号 | 需求 | 优先级 |
|---|---|---|
| M7-FR-01 | 提供统一调用封装：拼装 prompt（模板+注入变量）、设 CWD、执行 CLI、收集输出。 | MVP |
| M7-FR-02 | prompt 模板集中管理并固化于 `schema.md`（Interview a–d / Ingest / Query / Lint）。 | MVP |
| M7-FR-03 | 从环境变量读取 LLM 配置；缺失则明确报错指引用户配置，不内置密钥。 | MVP |
| M7-FR-04 | 调用串行化（队列），单用户同一时刻仅 1 个 CLI 任务。 | MVP |
| M7-FR-05 | 超时与重试（有限次）；失败返回结构化错误，触发上层 GR-6 降级，不丢已有数据。 | MVP |
| M7-FR-06 | 审计：每次调用记录 操作类型/所用模型/耗时/（可得的）成本/溯源/状态 → MySQL `audit_log`。 | MVP |
| M7-FR-07 | 解析 CLI 输出（约定 JSON 输出的操作做容错解析）。 | MVP |
| M7-FR-08 | 多用户并发调度与限流。 | 后续 |

## 5. 接口与数据
- 输入：`{operation, user_id, vars}`。输出：`{ok, data|error, audit}`。
- 数据：`audit_log`（→ [M8](M8-storage-data.md)）。

## 6. 依赖
- 依赖宿主机已安装并配置 `claude` CLI 与 Typst；M8（审计）。被 M2/M3/M4/M6 调用。

## 7. 验收标准
- ✅ 四操作均能经 CLI 在用户 wiki 目录上执行并返回预期输出。
- ✅ 密钥仅来自环境变量、代码无硬编码。
- ✅ 串行生效；失败有重试与结构化错误；每次调用有审计记录。
