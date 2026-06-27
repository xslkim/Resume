# 技术栈与工程约定（Tech Stack）

> 本文件是**实现层的强制约定**，给 ag agent 全自动开发用。需求文档（`REQUIREMENTS.md` + M1~M8）定"做什么"，本文件定"用什么、怎么组织"。
> 若需求文档与本文件冲突，以需求文档为准；本文件负责填补需求文档未涉及的技术细节。

---

## 1. 后端

| 维度 | 选型 | 说明 |
|---|---|---|
| 语言 | **Python 3.11+** | |
| Web 框架 | **FastAPI** | 异步、自动 OpenAPI 文档；适合 LLM 编排的长任务 |
| ASGI 服务器 | **uvicorn**（开发）；生产用 **uvicorn workers 或 gunicorn+uvicorn worker** | 2核4G 用 1~2 worker |
| 模板 | **Jinja2** | SSR 页面（登录/问答/表单/草稿预览） |
| ORM | **SQLAlchemy 2.x（async）** | |
| 迁移 | **Alembic** | 每次 schema 变更生成迁移 |
| 数据库驱动 | **aiomysql**（async MySQL） | 连"独立 MySQL 服务" |
| 数据校验 | **Pydantic v2** | FastAPI 自带；也用于内部 DTO |
| 配置 | **pydantic-settings** | 读环境变量，密钥/DB/路径等 |
| Git 操作 | **GitPython**（或直接 subprocess 调 git） | 应用层原子提交 |
| Shell/CLI 调用 | **`subprocess`**（async via `anyio`/`asyncio`） | 调 `claude` CLI，stdin 传用户内容 |
| 任务/队列 | **单进程内串行队列**（`asyncio.Lock`/单 worker） | M7 要求同一时刻仅 1 个 CLI 任务；MVP 不引 Celery |
| 日志 | 标准 **logging** | |

## 2. 前端

| 维度 | 选型 |
|---|---|
| 形态 | **最小 SSR（Jinja2 模板）+ 少量原生 JS**；单用户工具，**不引入 SPA 框架** |
| 交互 | 表单提交走普通 HTTP；问答/草稿预览的动态部分用 `fetch` + 少量 JS 局部刷新 |
| 样式 | 简洁 CSS（可引一份轻量 reset），不引重型 UI 库 |
| 文件上传 | multipart/form-data（导入简历、证件照） |
| 进度/排队 | 轮询或 SSE；MVP 可用简单轮询 |

> 原则：前端只为跑通流程，不在 UI 上过度投入。

## 3. 渲染

| 维度 | 选型 |
|---|---|
| PDF 引擎 | **Typst**（独立二进制） | 单栏 ATS 模板 `templates/standard.typ` |
| 调用方式 | 后端 subprocess 调 `typst compile`，输入 JSON+令牌，输出 PDF | 确定性渲染，LLM 不参与 |
| 字体 | Inter（拉丁）/ 思源黑体（CJK），**必须嵌入**；落地前确认目标 Typst 版本可用 |

## 4. LLM

| 维度 | 选型 |
|---|---|
| 调用方式 | **shell out 调 `claude` CLI（非交互）**；用户内容经 stdin；不接 SDK |
| 模型 | CLI 默认模型（代码不指定） |
| 密钥 | 用户手动配置环境变量（如 `ANTHROPIC_API_KEY` 等），代码只读不落库 |
| CWD | 各操作 CWD = 用户 `wiki/` 目录 |

## 5. 外部依赖（开发/验证环境已具备）

- ✅ `claude` CLI 已安装，且有可用 **API Key**
- ✅ **Typst** 已安装
- ✅ MySQL（独立服务，开发环境可本地起一个）

## 6. 项目结构（建议骨架）

```
resume/
  app/
    main.py                  # FastAPI 入口、路由挂载
    config.py                # pydantic-settings，环境变量
    deps.py                  # 依赖注入（db session、current_user）
    db/                      # engine、session、base
      base.py, session.py
    models/                  # SQLAlchemy ORM（对应 M8 表）
      user.py session.py interview.py jd.py generation.py output.py audit.py
    schemas/                 # Pydantic DTO（API 入参/出参、JSON Resume 契约）
    modules/
      m1_user_auth/
      m2_intake/
      m3_knowledge/
      m4_generation/
      m5_rendering/
      m6_guardrails/
      m7_llm_orch/           # CLI 封装、prompt 模板加载、串行队列、审计
      m8_storage/            # 文件落盘、git、指针
    templates/               # Jinja2 SSR 页面
    static/                  # css/js
  prompts/                   # 将固化进各用户 wiki/schema.md，源在此维护
  typst/templates/standard.typ
  alembic/                   # 迁移
  tests/
  pyproject.toml
  .env.example
```

## 7. 工程约定

- **包管理**：`uv`（或 `pip` + `pyproject.toml`）；依赖锁版本。
- **测试**：`pytest` + `pytest-asyncio`；外部 CLI（claude/typst）在测试中以接口隔离，可注入 fake/mock，端到端测试单独标记（需真实 CLI）。
- **配置/密钥**：全部走环境变量；`.env.example` 提交，`.env` 不入库。
- **代码风格**：`ruff`（lint+format）。
- **路径安全**：所有用户内容生成的文件名经 slug 清洗（M3 §3.1），禁止拼 `..`/`/`。
- **隔离**：每个请求带 user_id，文件/DB 操作强制带 user_id 过滤。
- **提交粒度**：一个任务一个 PR；PR 标注对应任务 ID（T-xx）。

## 8. 数据库连接（约定）

- 从环境变量 `DATABASE_URL`（如 `mysql+aiomysql://user:pass@host:3306/resume`）读取。
- 表名用复数（`users/sessions/...`），与 M8 §4 一致。
- 字段名 snake_case；时间戳统一 `created_at`/`updated_at`（UTC 存储）。

## 9. 版本与依赖锁定（外部二进制）

- 记录并锁定 `claude` CLI 版本与 Typst 版本（写进 `config`/启动自检），纳入兼容性检查（M7-FR-09）。
