# 技术栈与工程约定（Tech Stack）

> 本文件是**实现层的强制约定**，给 AI agent 全自动开发用。需求文档（`REQUIREMENTS.md` + M1~M8）定"做什么"，本文件定"用什么、怎么组织"。
> 若需求文档与本文件冲突，以需求文档为准；本文件负责填补需求文档未涉及的技术细节。

---

## 1. 后端

| 维度 | 选型 | 说明 |
|---|---|---|
| 语言 | **Python 3.11+** | |
| Web 框架 | **FastAPI** | 异步、自动 OpenAPI 文档；适合 LLM 编排的长任务 |
| ASGI 服务器 | **uvicorn**（开发）；生产用 **uvicorn 单 worker**（或 gunicorn + uvicorn worker，`--workers 1`） | **MVP 强制 1 worker**，见 §1.1 |
| 模板 | **Jinja2** | SSR 页面（登录/问答/表单/草稿预览） |
| ORM | **SQLAlchemy 2.x（async）** | |
| 迁移 | **Alembic** | 每次 schema 变更生成迁移 |
| 数据库驱动 | **aiomysql**（async MySQL） | 连"独立 MySQL 服务" |
| 数据校验 | **Pydantic v2** | FastAPI 自带；也用于内部 DTO |
| 配置 | **pydantic-settings** | 读环境变量，密钥/DB/路径等 |
| Git 操作 | **GitPython**（或直接 subprocess 调 git） | 应用层原子提交 |
| Shell/CLI 调用 | **`subprocess`**（async via `anyio`/`asyncio`） | 调 `claude` CLI，stdin 传用户内容 |
| 任务/队列 | **单进程内串行队列**（`asyncio.Lock`/单 worker） | M7 要求同一时刻仅 1 个 CLI 任务；MVP 不引 Celery。**进程内锁只对单进程有效**，故必须配合 §1.1 的单 worker 约束 |
| 日志 | 标准 **logging** | |

### 1.1 worker 数量：MVP 锁定 1（重要）

M7-FR-04 要求"同一时刻仅 1 个 CLI 任务"，且单用户无需并发。串行队列用**进程内 asyncio 锁**实现，**只对单进程有效**。一旦起 2 个 uvicorn worker（= 2 个进程），进程内锁管不住跨进程，会出现 2 个 CLI 并发跑，违反 M7-FR-04，2核4G 上还可能 OOM。

**结论：MVP 强制单实例运行**（`--workers 1`）。但应用进程**从内部无法可靠得知总 worker 数**（每个 worker 是独立进程，互相不知彼此），故**不用"数 worker"自检**，而是用**启动抢独占文件锁**实现：

- 应用启动时抢一把**独占文件锁**（`portalocker`，跨平台；锁文件如 `DATA_DIR/app.lock`）。
- **第 2 个进程抢不到锁 → 立即报错退出**。这样既检测了多 worker / 多实例，又顺手提供了**跨进程串行保护**（T-04 的串行队列可据此扩展为跨进程，MVP 先用进程内锁 + 这把实例锁组合即可）。
- 比从进程内"探测 worker 数"稳健得多，且实现成本极低。

后续若要多 worker，须先把串行锁升级为跨进程锁（DB 行锁等）并重新评估——属于"后续"工作，不在 MVP。

### 1.2 运行时关键依赖（除选型表外，务必写入 `pyproject.toml`）

| 依赖 | 用途 | 关联任务 |
|---|---|---|
| **python-multipart** | FastAPI multipart 表单/文件上传（登录表单、导入简历、证件照） | T-03 / T-07 |
| **bcrypt**（或 `passlib[bcrypt]`） | 密码 bcrypt 哈希 | T-03 |
| **itsdangerous** | httpOnly 签名 cookie 的签名/校验 | T-03 |
| **pypdf** + **pdfplumber** | 导入：PDF 简历抽文本 | T-07 |
| **python-docx** | 导入：Word 简历抽文本 | T-07 |
| **httpx** | 测试客户端（FastAPI TestClient/异步） | T-00 起全部 |
| **anyio** | 异步 subprocess（调 claude/typst） | T-04 / T-09 |
| **portalocker** | 启动独占文件锁（防多 worker/多实例，兼跨进程串行保护基础）；CLI 运行期文件写锁 | T-00 / T-04 |

## 2. 前端

| 维度 | 选型 |
|---|---|
| 形态 | **最小 SSR（Jinja2 模板）+ 少量原生 JS**；单用户工具，**不引入 SPA 框架** |
| 交互 | 表单提交走普通 HTTP；问答/草稿预览的动态部分用 `fetch` + 少量 JS 局部刷新 |
| 样式 | 简洁 CSS（可引一份轻量 reset），不引重型 UI 库 |
| 文件上传 | multipart/form-data（导入简历、证件照） |
| 进度/排队 | 轮询或 SSE；MVP 可用简单轮询 |
| 会话令牌 | **httpOnly 签名 cookie**（itsdangerous），契合 SSR+Jinja2。**钉死这一种**，不用 bearer（T-03 依此实现） |

> 原则：前端只为跑通流程，不在 UI 上过度投入。

## 3. 渲染

| 维度 | 选型 | 说明 |
|---|---|---|
| PDF 引擎 | **Typst**（独立二进制） | 单栏 ATS 模板 `templates/standard.typ` |
| 数据传入 | **走临时文件**：后端把结构化数据写 `outputs/<id>/data.json`，模板用 `#let data = json("data.json")` 读入 | `typst --input key=val` **只能传字符串**，大 JSON 必须走文件；token 类小参数可用 `--input` |
| 调用方式 | 后端 subprocess 调 `typst compile <main.typ>`，输出 PDF | 确定性渲染，LLM 不参与 |
| 字体 | Inter（拉丁）/ 思源黑体（CJK），**必须嵌入 PDF** | 字体文件须**预先放入项目字体目录**（如 `typst/fonts/`），模板用 `#set text(font: ...)` 声明；**不依赖系统字体**（CI/容器内常无字体）。属 T-09 环境准备，并验证产物 PDF 字体已嵌入 |
| 双语布局 | **每个区块内"zh 段 + en 段顺排"**（如一条经历下方先排中文 bullet，再排英文 bullet；Summary 先 zh 后 en），单一 PDF 内并列呈现 | `language=bilingual` 时渲染一份双语 PDF（非两份）。各区块用清晰分隔（小标题或留白）区分语种 |

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
- ⚠️ **字体文件需自行准备**放入 `typst/fonts/`（Inter + 思源黑体），不依赖系统字体

## 6. 项目结构（建议骨架）

```
resume/
  app/
    main.py                  # FastAPI 入口、路由挂载、启动自检（worker 数、CLI/Typst 版本）
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
  prompts/                   # 固化进各用户 wiki/schema.md 的契约源，在此维护
  typst/
    templates/standard.typ
    fonts/                   # Inter + 思源黑体（随仓库分发）
  alembic/                   # 迁移
  tests/
  pyproject.toml
  .env.example
```

## 7. 工程约定

- **包管理**：`uv`（或 `pip` + `pyproject.toml`）；依赖锁版本。
- **测试**：`pytest` + `pytest-asyncio`；外部 CLI（claude/typst）以接口隔离，可注入 fake/mock；涉及真实 CLI 的用 `@pytest.mark.e2e` 单独标记。
- **配置/密钥**：全部走环境变量；`.env.example` 提交，`.env` 不入库。
- **代码风格**：`ruff`（lint+format）。
- **路径安全**：所有用户内容生成的文件名经 slug 清洗（M3 §3.1），禁止拼 `..`/`/`。
- **隔离**：每个请求带 user_id，文件/DB 操作强制带 user_id 过滤。
- **幂等/可重跑**：`alembic upgrade`、`git init`、用户目录创建等初始化操作必须**幂等**（已存在则跳过，不报错），以便任务可安全重跑。
- **提交粒度**：一个任务一个 PR；PR 标题含任务 ID（如 `[T-01] ...`）。

## 8. 数据库连接（约定）

- 从环境变量 `DATABASE_URL`（如 `mysql+aiomysql://user:pass@host:3306/resume`）读取。
- 表名用复数（`users/sessions/...`），与 M8 §4 一致。
- 字段名 snake_case；时间戳统一 `created_at`/`updated_at`（UTC 存储）。

## 9. 版本与依赖锁定（外部二进制）

- 记录并锁定 `claude` CLI 版本与 Typst 版本（写进启动自检），纳入兼容性检查（M7-FR-09）。
