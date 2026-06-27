# 开发任务清单（面向 AI agent 自动开发）

> **目的**：本文档是给 AI agent **全自动完成开发**用的任务说明书。需求见 `REQUIREMENTS.md` + `modules/M1~M8`，技术约定见 `TECH-STACK.md`。本文件只定"做什么任务、怎么执行、交付什么、怎么验证、依赖谁"。
>
> **执行规则**：
> 1. 严格按依赖顺序执行；一个任务一个 PR，PR 标题含任务 ID（如 `[T-01] ...`）。
> 2. 每个任务的**交付物**是硬性完成标准；**验证**必须实际跑通（命令见各任务），不能只靠肉眼检查。
> 3. 状态用表格跟踪：`未开始 / 进行中 / 已完成 / 阻塞`。开始前先把目标任务置 `进行中`，完成并验证通过后置 `已完成`。
> 4. 任务粒度按模块拆，但**横切基础（T-00~T-03）先行**，否则后续无地基。
> 5. 遇到需求文档与本文冲突，以需求文档为准，并在 PR 里说明。
> 6. 外部依赖（claude CLI / Typst / MySQL）开发环境已具备，可真实跑端到端（见 TECH-STACK §5）。
> 7. **初始化操作必须幂等**：`alembic upgrade`、`git init`、用户目录创建等"已存在则跳过，不报错"，便于任务安全重跑。
> 8. **验证分两类，不要混淆**（见 §附 A）：
>    - **可自动验证**：必须落成断言/脚本（如动词黑名单正则扫描、`_meta.source` 覆盖率=100%、长度档位达标、技能数量上限、页数、ATS 文本提取）。
>    - **人工确认闸门**：内容质量、措辞好坏、真实性等**主观项无法自动断言**，明确标注 `【人工闸门】`，由人最终确认，**不伪装成自动验证**。

---

## 0. 任务总览与依赖图

```
T-00 项目脚手架与工程基线（含字体/CLI 版本自检）
  └─> T-01 数据库与 ORM 模型(M8)
        └─> T-02 文件/git 存储层(M8)
              ├─> T-03 用户与鉴权(M1)
              │     └─> T-04 LLM CLI 编排平台(M7)  ← 技术spike合并于此
              │           ├─> T-05 wiki schema 与目录初始化(M3)
              │           │     └─> T-06 Ingest 编译引擎(M3)
              │           │           ├─> T-07 素材录入:导入/手动/问答(M2)
              │           │           └─> T-10 后置校验(溯源/一致性部分)
              │           └─> T-08 JD分析与简历生成(M4)  ← 依赖 T-06 的编译产物
              │                 └─> T-09 Typst 渲染输出(M5)
              │                       └─> T-10 后置校验(长度/ATS 部分)
              └─> (所有功能模块依赖 T-01/T-02)
                                              └─> T-11 端到端串联与验收(集成)
```

**关键路径**：T-00 → T-01 → T-02 → T-03 → T-04 → T-05 → T-06 → (T-07 ‖ T-08 → T-09) → T-10 → T-11

**运行时主流程顺序（重要，T-10/T-11 依此）**：
**生成草稿(M4) → 预览渲染(M5) → 后置校验(M6，含长度/ATS，故在渲染之后) → 长度回环修正(M4+M5，限2次) → 定稿（人工闸门确认）→ 终稿渲染 PDF(M5)**。
> 即：**校验一定在渲染之后**（量页数、查 ATS 文本都需要 PDF 产物）。T-10 同时依赖 T-06（溯源 raw）和 T-09（长度/ATS 校验依赖渲染产物）。

---

## 1. 状态总表

| ID | 任务 | 模块 | 依赖 | 状态 |
|---|---|---|---|---|
| T-00 | 项目脚手架与工程基线（含字体/CLI 版本自检） | — | — | 未开始 |
| T-01 | 数据库与 ORM 模型 | M8 | T-00 | 未开始 |
| T-02 | 文件/git 存储层 | M8 | T-01 | 未开始 |
| T-03 | 用户与鉴权 | M1 | T-02 | 未开始 |
| T-04 | LLM CLI 编排平台（含技术 spike） | M7 | T-03 | 未开始 |
| T-05 | wiki schema 与目录初始化 | M3 | T-02,T-04 | 未开始 |
| T-06 | Ingest 编译引擎 | M3 | T-05 | 未开始 |
| T-07 | 素材录入（导入/手动/问答） | M2 | T-06 | 未开始 |
| T-08 | JD 分析与简历生成 | M4 | T-04,T-06 | 未开始 |
| T-09 | Typst 渲染输出 | M5 | T-08 | 未开始 |
| T-10 | 后置校验与护栏 | M6 | T-06,T-09 | 未开始 |
| T-11 | 端到端串联与验收 | 集成 | T-07,T-08,T-09,T-10 | 未开始 |

---

## T-00 项目脚手架与工程基线

**目标**：搭起可运行的空项目骨架，确立依赖管理、配置、日志、lint、测试基线；锁死单 worker；字体/CLI 版本自检。

**执行方式**：
1. 按 `TECH-STACK §6` 创建目录骨架（含 `typst/fonts/` 占位）。
2. `pyproject.toml` 锁定核心依赖（**务必含 §1.2 的运行时关键依赖**：python-multipart、bcrypt、itsdangerous、pypdf、pdfplumber、python-docx、httpx、anyio）+ 开发依赖（ruff、pytest、pytest-asyncio）。
3. `app/config.py`：pydantic-settings 读 `DATABASE_URL`、`ANTHROPIC_API_KEY`、`DATA_DIR`、CLI 超时、cookie 签名密钥等；缺失明确报错。
4. `app/main.py`：最小 FastAPI app + 健康检查 `GET /healthz` 返回 `{"ok":true}`。
5. **启动独占锁**（替代"数 worker"自检——进程内无法可靠得知总 worker 数）：用 `portalocker` 在启动时抢 `DATA_DIR/app.lock`，**第 2 个进程抢不到锁 → 立即报错退出**。一举检测多 worker / 多实例，并作为跨进程串行保护基础（见 TECH-STACK §1.1）。
6. `.env.example`（全量占位）、`.gitignore`（忽略 `.env`、`__pycache__`、`data/`、`*.pdf`、`typst/fonts/*.ttf` 等二进制字体按需）。
7. `ruff` 配置；`tests/` 放一个 smoke 测试（打 `/healthz`）。
8. `README.md`：如何 `uv sync`、如何 `--workers 1` 跑、如何跑测试。

**交付物**：可 `uvicorn app.main:app --workers 1 --reload` 启动并访问 `/healthz` 的项目；`ruff check` 通过；`pytest` 通过（含 smoke）；独占锁生效。

**验证**：
- `ruff check .` 无错误
- `uvicorn app.main:app --workers 1` 启动后 `curl localhost:8000/healthz` 返回 `{"ok":true}`
- 启动第 2 个实例（或 `--workers 2`，第二个进程）→ 因抢不到独占锁而报错退出
- `pytest -q` 全绿

**依赖**：无。

---

## T-01 数据库与 ORM 模型（M8 §4）

**目标**：建好 M8 全部核心表的 ORM 模型与迁移，按 user_id 隔离。

**执行方式**：
1. `app/db/`：async engine + sessionmaker（从 `DATABASE_URL`）。
2. `app/models/`：逐表建 SQLAlchemy 模型，共 **7 张表**：`users / sessions / interview_sessions / jd / generations / output_files / audit_log`。**字段名/表名严格对齐 M8 §4 的 SQL**（含索引 `INDEX(user_id)` 等）。时间戳用 `DateTime(timezone=True)` 存 UTC。**只建 M8 §4 列出的这 7 张，不要凭空加第 8 张表**。
3. `alembic` 初始化并生成首个迁移（autogenerate）。**幂等**：`alembic upgrade head` 在已迁移库上重跑不报错。
4. 写一个 user_id 隔离的单元测试：插两条不同 user_id 数据，验证查询只返回本用户。

**交付物**：7 张表的 ORM 模型 + Alembic 迁移脚本 + 隔离测试。

**验证**：
- `alembic upgrade head` 在干净库上成功建出全部表；在已建库上重跑不报错
- 表数量核对：`SHOW TABLES` 恰好为 M8 §4 的 **7 张**（不多不少）
- `pytest tests/test_models_isolation.py` 验证 user_id 隔离
- 表结构 `DESC` 与 M8 §4 一一对应（自查表）

**依赖**：T-00。

---

## T-02 文件/git 存储层（M8 §2/§3 + M3 §2 布局）

**目标**：实现文件落盘、按用户隔离的目录创建、git 初始化与原子提交。

**执行方式**：
1. 实现 `app/modules/m8_storage/`：
   - `ensure_user_dirs(user_id)`：按 M3 §2 唯一权威布局创建 `users/<user_id>/{raw,wiki/{experiences,skills,topics,profile},outputs,uploads}`。**幂等**。
   - 文件读写接口（raw/wiki/outputs/uploads），所有路径强制带 user_id 根，**slug 清洗函数**（M3 §3.1：仅 `[a-z0-9-]`，过滤 `..`/`/`，拼音转写+短哈希去重，长度≤50）。
   - git：`init_repo(user_id)`（幂等）；`commit_atomically(user_id, message)` = `git add raw/ wiki/ && commit`（M3-FR-08，应用层单次原子提交）。**幂等处理空 diff**（无变更时跳过提交不报错）。
2. 二进制文件落 outputs/uploads，MySQL 存指针（`output_files.path`）。
3. 单元测试：创建用户目录、写入 raw 文件、原子提交、`git log` 可见、回滚可恢复。

**交付物**：存储层模块 + slug 清洗 + git 原子提交 + 测试。

**验证**：
- `pytest tests/test_storage.py`：建目录→写 raw→commit→`git log` 有记录→`git revert` 后文件回到提交前
- **幂等**：重复 `ensure_user_dirs` / 重复 `commit_atomically`（空 diff）不报错
- slug 清洗测试：`"Acmé 公司/项目?!"` → 仅含 `[a-z0-9-]`；含 `../` 的输入被清洗

**依赖**：T-01。

---

## T-03 用户与鉴权（M1）

**目标**：单用户登录、服务端 sessions 表、登出/改密吊销、按 user_id 隔离边界。

**执行方式**：
1. 预置账号：启动时若 users 表空，从环境变量读初始 email/口令创建（M1-FR-01），密码 **bcrypt** 哈希。
2. 登录 `POST /login`（multipart 表单）→ 校验 → 写 `sessions` 表（`token_hash`、`expires_at`）→ 签发 **httpOnly 签名 cookie**（itsdangerous）。**钉死 cookie 这一种**（见 TECH-STACK §2），不用 bearer。
3. `deps.py`：`get_current_user` 从 cookie 令牌查 sessions（未吊销、未过期），注入 user_id；失败 401。
4. 登出 → 置 `revoked_at`；改密 → 该用户全部 session 吊销（M1-FR-04/05）。
5. 中间件：所有需登录路由强制带 user_id，越权访问拒绝（403）。
6. SSR 登录页（Jinja2）。

**交付物**：登录/登出/改密 API（cookie 会话）+ SSR 页 + 会话中间件 + 隔离测试。

**验证**：
- 登录拿 cookie→带 cookie 访问受保护路由 200；不带 401。
- 登出后旧 cookie 401（不可复用，M1-FR-04）。
- 改密后所有旧会话 401（M1-FR-05）。
- 越权 user_id 访问 403（M1 §5）。

**依赖**：T-02。

---

## T-04 LLM CLI 编排平台（M7，含技术 spike）

**目标**：封装 `claude` CLI 调用，跑通四个操作的最小调用，验证关键假设；提供串行队列/超时/重试/审计/JSON 解析；处理 CLI 运行期文件竞态。

**执行方式（分两段）**：
1. **技术 spike（先做，验证 M7 §8 的关键假设）**：对 Interview/Ingest/Query/Lint 各写一个最小 CLI 调用脚本，验证：非交互 stdin 输入可用、稳定 JSON 输出、CWD 隔离生效、默认模型/版本、成本/token 是否回传。**假设不成立则停下报告，调整方案后再继续**——这是本任务的硬门。
2. **正式封装**：
   - `app/modules/m7_llm_orch/`：`LLMClient.invoke(operation, user_id, vars)`。
   - prompt 模板从源 `prompts/` 加载（运行时写入各用户 `wiki/schema.md`，见 T-05）；运行时变量 `{{...}}` 注入。
   - CWD = `users/<user_id>/wiki/`；用户内容（作答/JD）经 stdin，**禁止拼命令行**（GR-9）。
   - **串行**：单用户进程内 `asyncio` 单 worker 队列，同一时刻仅 1 个 CLI 任务（M7-FR-04）。**仅对单进程有效**，故依赖 TECH-STACK §1.1 的单 worker 约束。
   - 超时（Query 宽松）+ 有限次重试；失败返回结构化错误供 GR-6 降级。
   - JSON 输出协议：提取首个 `{` 到末个 `}`、去围栏、解析失败重试再降级（M7-FR-07）。
   - 审计：每次调用写 `audit_log`（操作/模型/耗时/成本/状态/溯源），成本无回传则估算（M7-FR-06）。
   - 记录/校验 `claude` CLI 版本（M7-FR-09）。
   - **CLI 运行期文件竞态**（M7 §8 Q-3，已知风险）：CLI 运行期间对用户 wiki 文件加**文件级写锁**（如 `portalocker` 或简单 `.lock` 标记），并发手动编辑（M2）同文件时排队/拒绝并提示；单用户降低概率但非零。

**交付物**：spike 报告（结论：假设是否成立）+ LLMClient 封装 + 串行队列 + 审计写入 + 文件写锁 + 四操作最小集成测试（用真实 CLI）。

**验证**：
- spike 四操作各返回预期 JSON 结构（真实 CLI 跑通）。
- 并发触发 2 个调用→实测串行（第二个排队）。
- 超时/故意输入破坏 JSON→触发重试→最终降级返回结构化错误，不抛未处理异常。
- `audit_log` 每次调用有记录。
- CLI 运行期对某文件写锁生效：此时另一路手动写该文件被挡下/排队。
- 代码 grep 无用户内容拼进命令行参数（GR-9）。

**依赖**：T-03。

---

## T-05 wiki schema 与目录初始化（M3 §3）

**目标**：把"人写的契约" `schema.md` 落到每个用户 wiki 目录；定义并固化 wiki 内容 schema。

**执行方式**：
1. 在 `prompts/schema.md` 维护契约源（页面规则 §3.1~§3.7 + 操作 prompt 模板 a–d/Ingest/Query/Lint）。用户首次初始化时复制到 `wiki/schema.md`（M7-FR-02）。
2. 实现 wiki 文件的**结构校验器**：读 markdown frontmatter，校验字段是否符合 §3（经历页字段、bullet 结构 `{id,source,skills,star?,metrics?,text}`、技能页 `level×evidenced_by` 一致性提示 GR-2、basics 必选字段）。
3. 提供"wiki 状态摘要"接口（M3-FR-11）：index 概览 + 缺口 + **字符数/段数估算**（不依赖 token 分词），供 M2/M4 读。
4. 矛盾记录 `wiki/issues.md` 的读写（M3-FR-16）。
5. **schema 副本陈旧说明**（单用户影响小）：记录 `prompts/schema.md` 的内容哈希到用户 `wiki/schema.md` frontmatter；启动时若源哈希更新，在 issues.md 提示"schema 源已更新，建议同步"（不自动覆盖，避免覆盖用户可能调整的 prompt）。

**交付物**：`prompts/schema.md` 契约源 + 初始化逻辑 + 结构校验器 + 状态摘要接口 + 测试。

**验证**：
- 给定一个合法经历页 markdown → 校验器通过；故意缺 `source`/加 schema 外字段 → 校验失败并指明（GR-7）。
- 状态摘要能输出段数/字符数。
- issues.md 读写正确。

**依赖**：T-02, T-04。

---

## T-06 Ingest 编译引擎（M3 §4）

**目标**：把"已确认条目"（应用经 stdin 传给 CLI）编译进 wiki，自动双语、维护交叉引用/index/log、标记矛盾、跳过软删。

**执行方式**：
1. 实现 Ingest 操作（经 T-04 LLMClient，CWD=wiki/，输入=已确认条目经 stdin）：
   - 按 schema 生成/更新经历页/技能页，维护 `[[]]` 互链与 tags（M3-FR-03/07）。
   - 自动中英双语（M3-FR-04）。
   - 维护 index.md / log.md（M3-FR-05）。
   - 矛盾标记写入 issues.md，不静默覆盖（M3-FR-06/16，GR-2）。
2. **软删除跳过**：读 raw frontmatter 的 `deleted:true`，跳过不编回 wiki（M3-FR-18）。
3. **应用层编排入库闭环**：用户确认→应用写 raw（带块锚点 `^a/^b`，M3-FR-17）→调 Ingest（写 wiki）→应用 `commit_atomically`（raw+wiki 一次提交，M3-FR-08）。
4. Ingest 输入明确不含 raw 读取（CWD=wiki/ 接触不到 raw），raw/wiki 同源于同一份确认条目（GR-7 保证方式）。

**交付物**：Ingest 操作集成 + 入库闭环编排 + 软删除跳过 + raw 锚点写入 + 测试（真实 CLI）。

**验证**：
- 给定确认条目→Ingest 后 wiki 出现合法经历/技能页，双语齐全，bullet 带 id/source/skills。
- index/log 更新；故意制造时间冲突→issues.md 出现 open 矛盾，未被覆盖。
- 软删一条→重 Ingest→该条不进 wiki。
- 入库后 `git log` 一次原子提交（raw+wiki 都在）。
- `verify_against_raw` 用到的锚点 `#a` 能在 raw 中定位（为 T-10 铺路）。

**依赖**：T-05。

---

## T-07 素材录入：导入/手动/问答（M2）

**目标**：三条冷启动入口（导入已有简历 / 手动录入 / 问答驱动），basics 采集，回显确认，缺口闭环。

**执行方式**：
1. **导入（M2-FR-12，最高频）**：上传 **PDF/Word** → 先**抽文本**（PDF 用 pypdf/pdfplumber；Word 用 python-docx）→ 喂 LLM 解析为 raw 候选 → **回显确认** → 入库编译。
   - **LinkedIn 仅支持"导出为 PDF"那种简历 PDF**（即"Save as PDF"的简历页），走上面同一 PDF 抽取路径。
   - **LinkedIn "Data Export"（账号下载 ZIP，含 CSV/JSON）不在 MVP 范围**（结构差异大、非简历形态），导入入口明确提示"请上传 PDF/Word 简历"。
   - 抽文本为空或乱码 → 报错回前端，不进 LLM。
2. **手动录入（M2-FR-13/15/16/17）**：表单录经历/成就/技能/证书；编辑、软删除（`deleted:true`）；时间戳；预置标签分类。
3. **basics 采集（M2-FR-14）**：首次/缺口驱动采集 name/email/phone/location/headline；缺 basics 时状态机 INIT 引导补全。**含证件照上传**：在**允许照片的地区**（`region=cn`，照片开关默认关；`na/eu/other` 不提供照片字段）提供照片上传入口（落 `uploads/`，存指针到 basics），与 M5 §5b 照片规则一致——`na/eu/other` 即使上传也不渲染。
4. **问答（M2-FR-01~09/18）**：状态机（INIT→SKELETON→DEPTH→CONFIRM→INGEST→DONE，§4）；提问策略（§3 漏斗，澄清不诱导）；抽取回显确认；会话可中断续答（存 `interview_sessions`）；提问预算（M2-FR-08）；**指标回忆引导**（M2-FR-18）；缺口→补真实素材闭环（M2-FR-09，文案"补你确实做过但没录的"，非"凑齐缺口"）。
5. SSR 页面：导入上传、手动表单、问答界面（进度显示、确认回显）。

**交付物**：三入口的 API + SSR 页 + 会话状态机持久化 + 缺口闭环入口 + 抽文本工具 + 测试。

**验证**：
- 导入一份真实旧简历（PDF）→抽文本非空→解析候选→确认→wiki 出现条目（真实 CLI）。
- 抽文本：构造一个非文本 PDF（扫描件）→报错拦截，不进 LLM。
- 手动录一段经历+软删→行为正确。
- 问答：空白起步→提问→作答→回显确认→入库；中断后续答回到原状态。
- basics 缺失时被 INIT 引导补全。
- 证件照：`region=cn`+开照片→可上传且存指针；`region=na`→不提供照片字段。
- grep 提问 prompt 文案，确认"补真实素材"措辞而非"凑齐缺口"。

**依赖**：T-06。

---

## T-08 JD 分析与简历生成（M4）

**目标**：JD 抽取、Query 全量选材润色、详略取舍、Summary（成长主线）、结构化数据（JSON Resume + _meta sidecar）、缺口分析、多 JD 并存、草稿。

**执行方式**：
1. **JD 分析（M4-FR-01/02）**：抽技能、资深级别、行业、**硬门槛淘汰信号**、软技能等；存 `jd` 表（keywords_json）。
2. **Query（M4-FR-04~09）**：经 T-04 CLI（CWD=wiki/，输入全量 wiki+JD+length/language/region）。
   - **依赖 T-06**：读的是已编译的 wiki（非直接读 raw），故 T-08 依赖 T-06。
   - 详略取舍（相关性×时效×成就强度，跨行转岗强调可迁移技能，M4-FR-05）。
   - 润色档位（轻/标/重，默认标准，M4-FR-06）。
   - Summary 含成长主线（M4-FR-11）。
   - 输出**结构化数据**：严格 JSON Resume 主数据 + `_meta` sidecar（`source/lang` 按路径挂、顶层 `gaps/params/positioning`，统一无前缀，M4 §3）。**附最小示例 JSON 固化契约**。
3. **参数枚举**：`region: cn|na|eu|other`、`length: 1page|2page|auto`、`language: zh|en|bilingual`。
4. **缺口分析（M4-FR-10）**：JD 要求但 wiki 薄弱/缺失 + 硬门槛是否满足；缺口项可一键转 M2 补素材。
5. **多 JD 并存（M4-FR-16）**：每个 generation 绑定 jd_id，多份不覆盖。
6. **内容质量规范执行（M4 §7）**：把禁用模式（Summary 反泛泛、动作动词黑名单、空泛软技能、技能数量上限、禁用代码行数等坏指标、好/坏 few-shot 示例）写进 Query 的系统 prompt，并**导出一份机器可读的清单**供 T-10 自动校验复用。**黑名单须中英两套**：英文万金油句式（"results-driven engineer…"等）+ 中文万金油句式（"具有 X 年经验的工程师…"等）；动作动词黑名单中英各一（如英 "Spearheaded/Orchestrated/Drove"、中"主导/赋能/助力"等的过度使用阈值）。
7. 草稿：生成先以草稿存（generations.status=draft），可预览。
8. **生成请求表单（SSR 交付）**：一个显式的生成发起页——选 JD（从已存 jd 列表）+ 选 `length`（1page/2page/auto）+ `language`（zh/en/bilingual）+ `region`（cn/na/eu/other）+ 润色档位（轻/标/重）→ POST 触发 Query。表单校验枚举值合法后才提交。

**交付物**：JD 分析 + Query 集成 + 结构化数据契约（含示例 JSON）+ 缺口分析 + 草稿持久化 + SSR 预览页 + **生成请求表单（SSR）** + **机器可读内容规范清单（中英两套）** + 测试（真实 CLI）。

**验证**（自动部分 + 人工闸门，见 §附 A）：
- 自动：输入真实 JD→得到合法 JSON Resume + _meta；每条 bullet 有 `source`；**`_meta.source` 覆盖率=100%**。
- 自动：技能数量 ≤ 规范上限；Summary 通过动词黑名单/万金油句式正则扫描（**中英两套分别扫描**，中文 Summary 用中文清单、英文 Summary 用英文清单）。
- 自动：`region=na`→basics 越界字段不在主数据（主责不输出，M5 §7 兜底）。
- 自动：同一 wiki 针对两个 JD→两个 generation 并存。
- 自动：缺口列表含硬门槛判定。
- 【人工闸门】：Summary 是否有成长主线、措辞好坏、bullet 力度——人最终确认。

**依赖**：T-04, T-06。

---

## T-09 Typst 渲染输出（M5）

**目标**：结构化数据 → ATS 友好单栏 PDF；地区规则驱动 basics/纸张；设计令牌；长度溢出回环；最小降级；字体嵌入。

**环境准备（执行第一步）**：
- 将 **Inter + 思源黑体** 字体文件放入 `typst/fonts/`，模板用 `#set text(font: ...)` 显式声明，**不依赖系统字体**（CI/容器常无字体）。此步在 T-09，不另开任务。

**执行方式**：
1. `typst/templates/standard.typ`：按 M5 §4/§8 实现 ATS 单栏模板（Header→Summary→Experience→Projects→Skills→Education→附加；应届生 Education 提前；字号/配色/页边距按 §4）。**双语布局**（`language=bilingual`）：每个区块内 zh 段+en 段顺排（先中后英），单一 PDF 内呈现，见 TECH-STACK §3。
2. **数据传入**：后端把结构化数据写 `outputs/<id>/data.json`，模板用 `#let data = json("data.json")` 读入（**不走 `--input`**，大 JSON 必须走文件，见 TECH-STACK §3）。token 类小参数可 `--input`。
3. 渲染服务：`render(generation_id)` → 后端 subprocess 调 `typst compile <main.typ>`，输出 PDF 落 outputs，写 `output_files`。
4. **地区规则（M5 §5b）**：`na`→禁全部敏感字段+Letter；`cn`→可渲染+A4；`eu`→禁照片年龄+A4；`other`→从严。**M5 渲染层强制过滤为兜底**（主责在 M4）。
5. **长度回环（M5-FR-07）**：渲染后检测页数，溢出则回传 M4 缩减低相关条目重渲染，**限 2 次**。（此回环属"长度排版回环"，与 M6"不自动回环"的 GR 校验不冲突。）
6. **最小降级（M5-FR-08）**：LLM 不可用时渲染最近成功定稿；若无则 basics+全部 bullet 平铺保守版。
7. ATS 硬规则（M5 §5）：真文本、标准区块标题、日期格式 `YYYY.MM – YYYY.MM`、文件名 `姓名_岗位_简历.pdf`。

**交付物**：standard.typ 模板 + 字体文件就位 + 渲染服务 + 地区规则表 + 长度回环 + 降级 + 测试。

**验证**：
- 自动：结构化数据→PDF；用 pdfplumber 抽取 PDF 文字可读（非图片扫描）、单栏（结构断言）。
- 自动：产物 PDF 字体已嵌入（`pdffonts` 或 Typst 日志确认 Inter/思源黑体 embedded）。
- 自动：`region=na`→PDF 无照片/年龄；`region=cn`+开启照片→有照片。
- 自动：`region=na`→纸张 Letter；`cn`→A4。
- 自动：故意塞超量内容→触发回环→≤2 次后页数达标或达上限停止。
- 自动：`language=bilingual`→一份 PDF 内含中英两套内容（区块内 zh 段+en 段顺排，非两份 PDF）；用 pdfplumber 抽取确认中英文本都在。
- 自动：LLM 不可用降级路径出 PDF。

**依赖**：T-08。

---

## T-10 后置校验与护栏（M6）

**目标**：草稿**预览渲染之后**一次性后置校验（GR-1 溯源 + GR-2 一致性 + GR-4 ATS/长度 + 翻译）；地区合规；注入防护；定稿前人工确认闸门。

**运行时位置**（见 §0 主流程）：生成草稿→**预览渲染**→**本任务校验**（量页数/ATS 需要渲染产物，故依赖 T-09）→长度回环→定稿→终稿渲染。

**执行方式**：
1. **`verify_against_raw`（M6-FR-02/08）**：应用读 raw，按块锚点结构化匹配每条 bullet 的 `_meta.source`→`has_source`（**自动断言覆盖率=100%**）；faithfulness 软判定（raw"参与"被写成"主导"等）用 LLM 兜底，输出 `faithfulness_warning`。
2. **一致性校验（M6-FR-03）**：时间区间不重叠冲突、同机构一致、头衔年限自洽（自动）。
3. **ATS/长度校验（M6-FR-04）**：单栏/真文本/标准标题 + 是否超长（**配合 T-09 的渲染产物**，自动）。
4. **地区合规（M6-FR-11，GR-10）**：按 region 拦截越界敏感字段（自动）。
5. **翻译/英文质检（M6-FR-12）**：与中文事实一致、术语正确、动词力度、"非母语高频错误"自检清单（中式英语/冠词时态/直译腔），不依赖用户英语水平（自动扫描 + 人工闸门确认）。
6. **注入防护（M6-FR-10，GR-9）**：检测输出偏离 schema / "忽略以上指令"类信号（自动）。
7. **内容规范自动扫描**：复用 T-08 的机器可读清单——动词黑名单、万金油句式正则、技能数量上限、`_meta.source` 覆盖率、长度档位达标。**能自动断言的全部自动断言**。
8. **一次性后置校验流程（M6-FR-01/§4）**：对草稿预览跑全部→全过放行定稿；有问题输出问题清单交回用户/M4（**不自动回环**；注：长度排版回环是另一回事，见 T-09）。
9. **定稿闸门（M6-FR-09）**：定稿须用户显式确认（含英文逐条过目）——**【人工闸门】**。
10. 校验报告存 `audit_log`（M6-FR-07）。

**交付物**：校验模块（各 GR 检查，自动部分落断言）+ 后置校验编排 + 定稿闸门 + 报告留存 + 测试。

**验证**：
- 自动：草稿某 bullet 去掉 source→`has_source=false` 被拦下标注；正常→`_meta.source` 覆盖率=100% 断言通过。
- 自动：故意制造时间重叠→一致性报警。
- 自动：`region=na` 但数据含年龄→合规校验拦截。
- 自动：Summary 含万金油句式/技能超上限→被自动扫描标出。
- 自动：全过→放行；有问题→问题清单（不自动回环）。
- 【人工闸门】：定稿无人工确认→拒绝转定稿。

**依赖**：T-06（raw 溯源）, T-09（长度/ATS 校验依赖渲染产物）。

---

## T-11 端到端串联与验收（集成）

**目标**：跑通 `REQUIREMENTS.md §7` 端到端验收，验证 MVP 成功标准。

**运行时主流程（依 §0）**：登录→录入（导入/问答/手动）→Ingest 编译 wiki→输入 JD →**生成草稿(M4) → 预览渲染(M5) → 后置校验(M6) → 长度回环(限2次) → 定稿(人工闸门) → 终稿渲染 PDF(M5)**。

**执行方式**：
1. 实现/打通主流程页面串联：登录→导入/问答录入→（看 wiki 编译）→输入 JD 生成草稿→预览渲染→后置校验→定稿确认→终稿 PDF。
2. 写端到端测试（真实 claude CLI + Typst + MySQL），覆盖 §7 八条验收。
3. 准备**合成但逼真的测试 fixture**（agent 无真实简历数据，不要卡在"必须真人数据"）：构造 2~3 段虚构但合理的经历（含可量化指标）、配套 basics、一个虚构但合理的 JD，完整跑一遍。fixture 入仓到 `tests/fixtures/`。
4. 修集成期暴露的跨模块问题。

**交付物**：端到端测试 + 合成 fixture（`tests/fixtures/`）跑出的完整运行记录（产物 PDF）+ §7 逐条对照表。

**验证**（`REQUIREMENTS.md §7` 逐条，自动 + 人工闸门）：
1. 自动：单用户登录，raw/wiki 隔离（M1）。
2. 自动：提问→作答→回显确认→入库（M2）。
3. 自动：Ingest 编译进 wiki 并双语、生成 index（M3）。
4. 自动：输入真实 JD，可选长度（M4）。
5. 自动：Query 全量选材润色，`_meta.source` 覆盖率=100%（M4+M6）。
6. 自动：Typst 渲染 ATS 中/英/双语 PDF（M5）。
7. 自动：每条经历可溯源（bullet 带来源 id）（M6）。
8. 自动：全流程经 claude CLI，跑在 2核4G + 独立 MySQL、`--workers 1`（M7+M8）。
- 【人工闸门】成功标准："得到**内容真实、针对岗位、可投递**的 ATS PDF"——"真实/针对/可投递"为主观判断，由人最终确认。

**依赖**：T-07, T-08, T-09, T-10。

---

## 附 A：自动验证 vs 人工闸门（重要约定）

本清单里所有"验证"分两类，**不可混淆**：

**A.1 可自动验证（必须落成断言/脚本）**
- 结构正确性：JSON Resume schema 合法、`_meta` 字段齐全。
- 溯源覆盖率：`_meta.source` 覆盖率 = 100%。
- 内容规范扫描：动词黑名单命中、万金油句式正则、技能数量上限、禁用坏指标（代码行数等）。
- 长度档位达标、页数、ATS 文本可提取、单栏结构、字体已嵌入、纸张尺寸。
- 合规：地区敏感字段拦截。
- 时序一致性、注入信号检测。

**A.2 人工闸门（明确标注 `【人工闸门】`，不伪装自动）**
- 内容质量、措辞好坏、bullet 力度、Summary 是否有成长主线、faithfulness 的灰度判断。
- "内容真实、针对岗位、可投递"的最终判定。
- 英文逐条过目、定稿确认。

> 自动验证保证"结构/合规/可溯源"硬底线；**内容质量的护城河靠人工闸门兜底**——这正是上一轮 content review 指出的薄弱点，已正视不回避。AI agent 负责把 A.1 全部自动化并跑通；A.2 的判断交还人，agent 只负责"把材料准备好供人确认"。

---

## 附：执行注意事项（给 AI agent）

1. **T-04 的 spike 是硬门**：在 T-04 完成前，T-05 及之后都可能因 CLI 假设不成立而返工。若 spike 失败，立即停下报告，不要硬写后续。
2. **单 worker 是约束的一部分**：串行队列只在单进程内有效（TECH-STACK §1.1）；任何起多 worker 的改动都必须先解决跨进程锁，否则违反 M7-FR-04。
3. **真实 CLI 测试 vs mock**：单元测试用 fake/mock 隔离 CLI；涉及 LLM 行为的验证（T-04/06/07/08）用真实 CLI，`@pytest.mark.e2e` 单独 marker，CI 可选跳过。
4. **GR-9 注入防护贯穿**：任何把用户内容（作答/JD）拼进命令行参数的代码都是 bug。code review 必查。
5. **raw 写权限红线**：除 T-02/T-06 的应用层写入外，任何 LLM 路径不得写 raw（CWD=wiki/ 是机制保证）。
6. **状态表更新**：每开始一个任务，把本文件 §1 状态总表对应行改 `进行中`；完成验证后改 `已完成`，并在 PR 里贴**自动验证**命令输出 + **人工闸门**结论。
7. **不实现"后续"项**：标 `后续` 的 FR（如多模板/DOCX/Lint/版本历史/多用户/多 worker）不在本清单，勿顺手做。
8. **一个任务一个 PR**：PR 描述含"交付物"清单与"验证"命令及结果（自动项贴输出，人工项贴结论）。
9. **CLI 运行期文件竞态**（M7 §8 Q-3）：已在 T-04 用文件写锁处理，单用户降低概率但非零，作为已知风险记录。
