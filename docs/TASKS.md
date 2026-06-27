# 开发任务清单（面向 ag agent 自动开发）

> **目的**：本文档是给 ag agent **全自动完成开发**用的任务说明书。需求见 `REQUIREMENTS.md` + `modules/M1~M8`，技术约定见 `TECH-STACK.md`。本文件只定"做什么任务、怎么执行、交付什么、怎么验证、依赖谁"。
>
> **执行规则**：
> 1. 严格按依赖顺序执行；一个任务一个 PR，PR 标题含任务 ID（如 `[T-01] ...`）。
> 2. 每个任务的**交付物**是硬性完成标准；**验证**必须实际跑通（命令见各任务），不能只靠肉眼检查。
> 3. 状态用表格跟踪：`未开始 / 进行中 / 已完成 / 阻塞`。开始前先把目标任务置 `进行中`，完成并验证通过后置 `已完成`。
> 4. 任务粒度按模块拆，但**横切基础（T-00~T-03）先行**，否则后续无地基。
> 5. 遇到需求文档与本文冲突，以需求文档为准，并在 PR 里说明。
> 6. 外部依赖（claude CLI / Typst / MySQL）开发环境已具备，可真实跑端到端（见 TECH-STACK §5）。
>
> **MVP 范围**：本清单只覆盖 `REQUIREMENTS.md §6` 的 MVP 功能；标注"后续"的需求**不实现**。

---

## 0. 任务总览与依赖图

```
T-00 项目脚手架与工程基线
  └─> T-01 数据库与 ORM 模型(M8)
        └─> T-02 文件/git 存储层(M8)
              ├─> T-03 用户与鉴权(M1)
              │     └─> T-04 LLM CLI 编排平台(M7)  ← 技术spike合并于此
              │           ├─> T-05 wiki schema 与目录初始化(M3)
              │           │     └─> T-06 Ingest 编译引擎(M3)
              │           │           ├─> T-07 素材录入:导入/手动/问答(M2)
              │           │           └─> T-10 后置校验与护栏(M6)
              │           └─> T-08 JD分析与简历生成(M4)
              │                 └─> T-09 Typst 渲染输出(M5)
              │                       └─> T-10 后置校验(长度/ATS 校验依赖渲染)
              └─> (所有功能模块依赖 T-01/T-02)
                                              └─> T-11 端到端串联与验收(集成)
```

**关键路径**：T-00 → T-01 → T-02 → T-03 → T-04 → (T-05 → T-06 → T-07) ‖ (T-08 → T-09) → T-11

---

## 1. 状态总表

| ID | 任务 | 模块 | 依赖 | 状态 |
|---|---|---|---|---|
| T-00 | 项目脚手架与工程基线 | — | — | 未开始 |
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

**目标**：搭起可运行的空项目骨架，确立依赖管理、配置、日志、lint、测试基线。

**执行方式**：
1. 按 `TECH-STACK §6` 创建目录骨架。
2. `pyproject.toml` 锁定核心依赖：fastapi、uvicorn、sqlalchemy、alembic、aiomysql、pydantic v2、pydantic-settings、jinja2、gitpython、ruff、pytest、pytest-asyncio、httpx（测试客户端）。
3. `app/config.py`：pydantic-settings 读 `DATABASE_URL`、`ANTHROPIC_API_KEY`、存储根路径 `DATA_DIR`、CLI 超时等；缺失时明确报错。
4. `app/main.py`：最小 FastAPI app + 健康检查 `GET /healthz` 返回 `{"ok":true}`。
5. `.env.example`（含所有所需环境变量占位）、`.gitignore`（忽略 `.env`、`__pycache__`、`data/`、`*.pdf`）。
6. `ruff` 配置；`tests/` 下放一个 smoke 测试（打 `/healthz`）。
7. `README.md`：写明如何 `uv sync`（或 pip install）、如何跑、如何跑测试。

**交付物**：可 `uvicorn app.main:app --reload` 启动并访问 `/healthz` 的项目；`ruff check` 通过；`pytest` 通过（含 smoke）。

**验证**：
- `ruff check .` 无错误
- `uvicorn app.main:app` 启动后 `curl localhost:8000/healthz` 返回 `{"ok":true}`
- `pytest -q` 全绿

**依赖**：无。

---

## T-01 数据库与 ORM 模型（M8 §4）

**目标**：建好 M8 全部核心表（users/sessions/interview_sessions/jd/generations/output_files/audit_log）的 ORM 模型与迁移，按 user_id 隔离。

**执行方式**：
1. `app/db/`：async engine + sessionmaker（从 `DATABASE_URL`）。
2. `app/models/`：逐表建 SQLAlchemy 模型，**字段名/表名严格对齐 M8 §4 的 SQL**（含索引 `INDEX(user_id)` 等）。时间戳用 `DateTime(timezone=True)` 存 UTC。
3. `alembic` 初始化并生成首个迁移（autogenerate）。
4. 写一个 user_id 隔离的单元测试：插两条不同 user_id 的数据，验证查询只返回本用户。

**交付物**：8 张表的 ORM 模型 + Alembic 迁移脚本 + 隔离测试。

**验证**：
- `alembic upgrade head` 在干净库上成功建出全部表
- `pytest tests/test_models_isolation.py` 验证 user_id 隔离
- 表结构 `DESC` 与 M8 §4 一一对应（自查表）

**依赖**：T-00。

---

## T-02 文件/git 存储层（M8 §2/§3 + M3 §2 布局）

**目标**：实现文件落盘、按用户隔离的目录创建、git 初始化与原子提交。

**执行方式**：
1. 实现 `app/modules/m8_storage/`：
   - `ensure_user_dirs(user_id)`：按 M3 §2 唯一权威布局创建 `users/<user_id>/{raw,wiki/{experiences,skills,topics,profile},outputs,uploads}`。
   - 文件读写接口（raw/wiki/outputs/uploads），所有路径强制带 user_id 根，**slug 清洗函数**（M3 §3.1：仅 `[a-z0-9-]`，过滤 `..`/`/`，拼音转写+短哈希去重，长度≤50）。
   - git：`init_repo(user_id)`；`commit_atomically(user_id, message)` = `git add raw/ wiki/ && commit`（M3-FR-08，应用层单次原子提交）。
2. 二进制文件落 outputs/uploads，MySQL 存指针（`output_files.path`）。
3. 单元测试：创建用户目录、写入 raw 文件、原子提交、`git log` 可见、回滚可恢复。

**交付物**：存储层模块 + slug 清洗 + git 原子提交 + 测试。

**验证**：
- `pytest tests/test_storage.py`：建目录→写 raw→commit→`git log` 有记录→`git revert` 后文件回到提交前
- slug 清洗测试：`"Acmé 公司/项目?!"` → 仅含 `[a-z0-9-]`；含 `../` 的输入被清洗

**依赖**：T-01。

---

## T-03 用户与鉴权（M1）

**目标**：单用户登录、服务端 sessions 表、登出/改密吊销、按 user_id 隔离边界。

**执行方式**：
1. 预置账号：启动时若 users 表空，从环境变量读初始 email/口令创建（M1-FR-01），密码 bcrypt 哈希。
2. 登录 `POST /login` → 校验 → 写 `sessions` 表（`token_hash`、`expires_at`），签发令牌（cookie 或 bearer）。
3. `deps.py`：`get_current_user` 从令牌查 sessions（未吊销、未过期），注入 user_id；失败 401。
4. 登出 → 置 `revoked_at`；改密 → 该用户全部 session 吊销（M1-FR-04/05）。
5. 中间件：所有需登录路由强制带 user_id，越权访问拒绝（403）。
6. SSR 登录页（Jinja2）。

**交付物**：登录/登出/改密 API + SSR 页 + 会话中间件 + 隔离测试。

**验证**：
- 登录拿令牌→带令牌访问受保护路由 200；不带 401。
- 登出后旧令牌 401（不可复用，M1-FR-04）。
- 改密后所有旧会话 401（M1-FR-05）。
- 越权 user_id 访问 403（M1 §5）。

**依赖**：T-02。

---

## T-04 LLM CLI 编排平台（M7，含技术 spike）

**目标**：封装 `claude` CLI 调用，跑通四个操作的最小调用，验证关键假设；提供串行队列/超时/重试/审计/JSON 解析。

**执行方式（分两段）**：
1. **技术 spike（先做，验证 M7 §8 的关键假设）**：对 Interview/Ingest/Query/Lint 各写一个最小 CLI 调用脚本，验证：非交互 stdin 输入可用、稳定 JSON 输出、CWD 隔离生效、默认模型/版本、成本/token 是否回传。**假设不成立则停下报告，调整方案后再继续**——这是本任务的硬门。
2. **正式封装**：
   - `app/modules/m7_llm_orch/`：`LLMClient.invoke(operation, user_id, vars)`。
   - prompt 模板从源 `prompts/` 加载（运行时写入各用户 `wiki/schema.md`，见 T-05）；运行时变量 `{{...}}` 注入。
   - CWD = `users/<user_id>/wiki/`；用户内容（作答/JD）经 stdin，**禁止拼命令行**（GR-9）。
   - 串行：单用户进程内 `asyncio` 单 worker 队列，同一时刻仅 1 个 CLI 任务（M7-FR-04）。
   - 超时（Query 宽松）+ 有限次重试；失败返回结构化错误供 GR-6 降级。
   - JSON 输出协议：提取首个 `{` 到末个 `}`、去围栏、解析失败重试再降级（M7-FR-07）。
   - 审计：每次调用写 `audit_log`（操作/模型/耗时/成本/状态/溯源），成本无回传则估算（M7-FR-06）。
   - 记录/校验 `claude` CLI 版本（M7-FR-09）。

**交付物**：spike 报告（结论：假设是否成立）+ LLMClient 封装 + 串行队列 + 审计写入 + 四操作最小集成测试（用真实 CLI）。

**验证**：
- spike 四操作各返回预期 JSON 结构（真实 CLI 跑通）。
- 并发触发 2 个调用→实测串行（第二个排队）。
- 超时/故意输入破坏 JSON→触发重试→最终降级返回结构化错误，不抛未处理异常。
- `audit_log` 每次调用有记录。
- 代码 grep 无用户内容拼进命令行参数（GR-9）。

**依赖**：T-03。

---

## T-05 wiki schema 与目录初始化（M3 §3）

**目标**：把"人写的契约" `schema.md` 落到每个用户 wiki 目录；定义并固化 wiki 内容 schema（经历页/技能页/basics/index/log/issues 的 frontmatter 规则）。

**执行方式**：
1. 在 `prompts/schema.md` 维护契约源（页面规则 §3.1~§3.7 + 操作 prompt 模板 a–d/Ingest/Query/Lint）。用户首次初始化时复制到 `wiki/schema.md`（M7-FR-02）。
2. 实现 wiki 文件的**结构校验器**：读 markdown frontmatter，校验字段是否符合 §3（经历页字段、bullet 结构 `{id,source,skills,star?,metrics?,text}`、技能页 `level×evidenced_by` 一致性提示 GR-2、basics 必选字段）。
3. 提供"wiki 状态摘要"接口（M3-FR-11）：index 概览 + 缺口 + **字符数/段数估算**（不依赖 token 分词），供 M2/M4 读。
4. 矛盾记录 `wiki/issues.md` 的读写（M3-FR-16）。

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
1. **导入（M2-FR-12，最高频）**：上传 PDF/Word 或 LinkedIn 导出→LLM 解析为 raw 候选→**回显确认**→入库编译。
2. **手动录入（M2-FR-13/15/16/17）**：表单录经历/成就/技能/证书；编辑、软删除（`deleted:true`）；时间戳；预置标签分类。
3. **basics 采集（M2-FR-14）**：首次/缺口驱动采集 name/email/phone/location/headline；缺 basics 时状态机 INIT 引导补全。
4. **问答（M2-FR-01~09/18）**：状态机（INIT→SKELETON→DEPTH→CONFIRM→INGEST→DONE，§4）；提问策略（§3 漏斗，澄清不诱导）；抽取回显确认；会话可中断续答（存 `interview_sessions`）；提问预算（M2-FR-08）；**指标回忆引导**（M2-FR-18）；缺口→补真实素材闭环（M2-FR-09，文案"补你确实做过但没录的"，非"凑齐缺口"）。
5. SSR 页面：导入上传、手动表单、问答界面（进度显示、确认回显）。

**交付物**：三入口的 API + SSR 页 + 会话状态机持久化 + 缺口闭环入口 + 测试。

**验证**：
- 导入一份真实旧简历→解析候选→确认→wiki 出现条目（真实 CLI）。
- 手动录一段经历+软删→行为正确。
- 问答：空白起步→提问→作答→回显确认→入库；中断后续答回到原状态。
- basics 缺失时被 INIT 引导补全。
- grep 提问 prompt 文案，确认"补真实素材"措辞而非"凑齐缺口"。

**依赖**：T-06。

---

## T-08 JD 分析与简历生成（M4）

**目标**：JD 抽取（技能+硬门槛）、Query 全量选材润色、详略取舍、Summary（成长主线）、结构化数据（JSON Resume + _meta sidecar）、缺口分析、多 JD 并存、草稿。

**执行方式**：
1. **JD 分析（M4-FR-01/02）**：抽技能、资深级别、行业、**硬门槛淘汰信号**、软技能等；存 `jd` 表（keywords_json）。
2. **Query（M4-FR-04~09）**：经 T-04 CLI（CWD=wiki/，输入全量 wiki+JD+length/language/region）。
   - 详略取舍（相关性×时效×成就强度，跨行转岗强调可迁移技能，M4-FR-05）。
   - 润色档位（轻/标/重，默认标准，M4-FR-06）。
   - Summary 含成长主线（M4-FR-11）。
   - 输出**结构化数据**：严格 JSON Resume 主数据 + `_meta` sidecar（`source/lang` 按路径挂、顶层 `gaps/params/positioning`，统一无前缀，M4 §3）。**附最小示例 JSON 固化契约**。
3. **参数枚举**：`region: cn|na|eu|other`、`length: 1page|2page|auto`、`language: zh|en|bilingual`。
4. **缺口分析（M4-FR-10）**：JD 要求但 wiki 薄弱/缺失 + 硬门槛是否满足；缺口项可一键转 M2 补素材。
5. **多 JD 并存（M4-FR-16）**：每个 generation 绑定 jd_id，多份不覆盖。
6. **内容质量规范执行（M4 §7）**：把禁用模式（Summary 反泛泛、动作动词黑名单、空泛软技能、技能数量上限、禁用代码行数等坏指标、好/坏 few-shot 示例）写进 Query 的系统 prompt。
7. 草稿：生成先以草稿存（generations.status=draft），可预览。

**交付物**：JD 分析 + Query 集成 + 结构化数据契约（含示例 JSON）+ 缺口分析 + 草稿持久化 + SSR 预览页 + 测试（真实 CLI）。

**验证**：
- 输入真实 JD→得到合法 JSON Resume + _meta；每条 bullet 有 `source`。
- Summary 不含万金油句式（人工抽检 + 内容规范检查）。
- `region=na`→basics 越界字段不在主数据（M4 主责不输出，M5 §7）。
- 同一 wiki 针对两个 JD→两个 generation 并存。
- 缺口列表含硬门槛判定。

**依赖**：T-04, T-06。

---

## T-09 Typst 渲染输出（M5）

**目标**：结构化数据 → ATS 友好单栏 PDF；地区规则驱动 basics/纸张；设计令牌；长度溢出回环；最小降级。

**执行方式**：
1. `typst/templates/standard.typ`：按 M5 §4/§8 实现 ATS 单栏模板（Header→Summary→Experience→Projects→Skills→Education→附加；应届生 Education 提前；字体嵌入；字号/配色/页边距按 §4）。
2. 渲染服务：`render(generation_id)` → 后端 subprocess 调 `typst compile`，输入 JSON+令牌，输出 PDF 落 outputs，写 `output_files`。
3. **地区规则（M5 §5b）**：`na`→禁全部敏感字段+Letter；`cn`→可渲染+A4；`eu`→禁照片年龄+A4；`other`→从严。**M5 渲染层强制过滤为兜底**（主责在 M4）。
4. **长度回环（M5-FR-07）**：渲染后检测页数，溢出则回传 M4 缩减低相关条目重渲染，**限 2 次**。
5. **最小降级（M5-FR-08）**：LLM 不可用时渲染最近成功定稿；若无则 basics+全部 bullet 平铺保守版。
6. ATS 硬规则（M5 §5）：真文本、标准区块标题、日期格式 `YYYY.MM – YYYY.MM`、文件名 `姓名_岗位_简历.pdf`。

**交付物**：standard.typ 模板 + 渲染服务 + 地区规则表 + 长度回环 + 降级 + 测试。

**验证**：
- 结构化数据→PDF，文字可选（非图片）、单栏、字体嵌入。
- `region=na`→PDF 无照片/年龄；`region=cn`+开启照片→有照片。
- `region=na`→纸张 Letter；`cn`→A4。
- 故意塞超量内容→触发回环→≤2 次后页数达标或达上限停止。
- LLM 不可用降级路径出 PDF。

**依赖**：T-08。

---

## T-10 后置校验与护栏（M6）

**目标**：生成草稿后一次性后置校验（GR-1 溯源 + GR-2 一致性 + GR-4 ATS/长度 + 翻译）；地区合规；注入防护；定稿前人工确认闸门。

**执行方式**：
1. **`verify_against_raw`（M6-FR-02/08）**：应用读 raw，按块锚点结构化匹配每条 bullet 的 `_meta.source`→`has_source`；faithfulness 软判定（raw"参与"被写成"主导"等）用 LLM 兜底，输出 `faithfulness_warning`。
2. **一致性校验（M6-FR-03）**：时间区间不重叠冲突、同机构一致、头衔年限自洽。
3. **ATS/长度校验（M6-FR-04）**：单栏/真文本/标准标题 + 是否超长（配合 T-09）。
4. **地区合规（M6-FR-11，GR-10）**：按 region 拦截越界敏感字段。
5. **翻译/英文质检（M6-FR-12）**：与中文事实一致、术语正确、动词力度、"非母语高频错误"自检清单（中式英语/冠词时态/直译腔），不依赖用户英语水平。
6. **注入防护（M6-FR-10，GR-9）**：检测输出偏离 schema / "忽略以上指令"类信号。
7. **一次性后置校验流程（M6-FR-01/§4）**：对草稿跑全部→全过放行定稿；有问题输出问题清单交回用户/M4（**不自动回环**；注：长度排版回环是另一回事，见 T-09）。
8. **定稿闸门（M6-FR-09）**：定稿须用户显式确认（含英文逐条过目）。
9. 校验报告存 `audit_log`（M6-FR-07）。

**交付物**：校验模块（各 GR 检查）+ 后置校验编排 + 定稿闸门 + 报告留存 + 测试。

**验证**：
- 草稿某 bullet 去掉 source→`has_source=false` 被拦下标注。
- 故意制造时间重叠→一致性报警。
- `region=na` 但数据含年龄→合规校验拦截。
- 全过→放行；有问题→问题清单（不自动回环）。
- 定稿无人工确认→拒绝转定稿。

**依赖**：T-06（raw 溯源）, T-09（长度/ATS 校验）。

---

## T-11 端到端串联与验收（集成）

**目标**：跑通 `REQUIREMENTS.md §7` 端到端验收，验证 MVP 成功标准。

**执行方式**：
1. 实现/打通主流程页面串联：登录→导入/问答录入→（看 wiki 编译）→输入 JD 生成草稿→后置校验→定稿→渲染 PDF。
2. 写端到端测试（真实 claude CLI + Typst + MySQL），覆盖 §7 八条验收。
3. 准备一份真实测试素材（几段真实经历）+ 一个真实 JD，完整跑一遍。
4. 修集成期暴露的跨模块问题。

**交付物**：端到端测试 + 真实素材/JD 的完整运行记录（产物 PDF）+ §7 逐条对照表。

**验证**（`REQUIREMENTS.md §7` 逐条）：
1. ✅ 单用户登录，raw/wiki 隔离（M1）。
2. ✅ 提问→作答→回显确认→入库（M2）。
3. ✅ Ingest 编译进 wiki 并双语、生成 index（M3）。
4. ✅ 输入真实 JD，可选长度（M4）。
5. ✅ Query 全量选材润色，未引入素材外事实（M4+M6）。
6. ✅ Typst 渲染 ATS 中/英/双语 PDF（M5）。
7. ✅ 每条经历可溯源（bullet 带来源 id）（M6）。
8. ✅ 全流程经 claude CLI，跑在 2核4G + 独立 MySQL（M7+M8）。
- **成功标准**：数分钟内得到内容真实、针对岗位、可投递的 ATS PDF。

**依赖**：T-07, T-08, T-09, T-10。

---

## 附：执行注意事项（给 agent）

1. **T-04 的 spike 是硬门**：在 T-04 完成前，T-05 及之后都可能因 CLI 假设不成立而返工。若 spike 失败，立即停下报告，不要硬写后续。
2. **真实 CLI 测试 vs mock**：单元测试用 fake/mock 隔离 CLI；涉及 LLM 行为的验证（T-04/06/07/08）用真实 CLI，单独 marker（如 `@pytest.mark.e2e`），CI 可选跳过。
3. **GR-9 注入防护贯穿**：任何把用户内容（作答/JD）拼进命令行参数的代码都是 bug。code review 必查。
4. **raw 写权限红线**：除 T-02/T-06 的应用层写入外，任何 LLM 路径不得写 raw（CWD=wiki/ 是机制保证）。
5. **状态表更新**：每开始一个任务，把本文件 §1 状态总表对应行改 `进行中`；完成验证后改 `已完成`，并在 PR 里贴验证命令输出。
6. **不实现"后续"项**：标 `后续` 的 FR（如多模板/DOCX/Lint/版本历史/多用户）不在本清单，勿顺手做。
7. **一个任务一个 PR**：PR 描述含"交付物"清单与"验证"命令及结果。
