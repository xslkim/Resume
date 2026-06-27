# M8 存储与数据模型 — 详细需求

> 框架见 [`../REQUIREMENTS.md`](../REQUIREMENTS.md)。
> **模块职责**：定义两类存储——**本机文件系统 + git**（raw/wiki 事实源、二进制产物）与**独立 MySQL**（结构化/事务数据）。提供落盘、git 提交、MySQL 表结构与指针约定。

---

## 1. 职责与边界
- **做**：文件/目录布局与 git 操作；MySQL 表结构；文件指针约定；（可选）备份到私有 GitHub remote。
- **不做**：业务逻辑（各模块）。不使用 OSS。

## 2. 存储分工
| 数据 | 存储 |
|---|---|
| raw + wiki markdown + git | 本机文件系统（事实源） |
| 生成 PDF/DOCX、上传图片/证件照 | 本机文件系统（MySQL 存指针+元数据） |
| 账号、会话、JD、生成记录、审计、文件指针 | 独立 MySQL（不在本机） |
| 远程备份（可选） | 私有 GitHub remote（只推 markdown 仓库） |

## 3. 文件布局
```
users/<user_id>/            # git 仓库（raw+wiki，见 M3 §2）
outputs/<user_id>/<generation_id>/resume.pdf   # 二进制产物（本机）
uploads/<user_id>/...        # 上传图片/证件照（本机）
```
- 每次 Ingest/Lint 一次 git 提交（可回滚）。
- 二进制不入 git（避免膨胀）；只在文件系统 + MySQL 指针。

## 4. MySQL 表结构（第一阶段核心）
```sql
users(id, email UNIQUE, password_hash, created_at)   -- 登录用 email（M1-FR-01）

sessions(                                      -- 服务端会话，支持登出/改密吊销（M1-FR-02/04/05）
  id, user_id, token_hash, expires_at, revoked_at, created_at,
  INDEX(user_id))

interview_sessions(                            -- 字段对应 M2 §4 状态机 / §6 会话状态
  id, user_id, state, current_focus,
  target_json, pending_extraction_json,
  covered_set_json, asked_set_json, skipped_set_json,
  created_at, updated_at,
  INDEX(user_id))

jd(id, user_id, raw_text TEXT, lang, keywords_json JSON, created_at,
  INDEX(user_id))

generations(
  id, user_id, jd_id, params_json JSON,        -- {length, language, region, template, positioning}
  status,                                       -- draft | final
  structured_data_json JSON,                    -- JSON Resume 主数据 + 并行 _meta sidecar（见 M4 §3）
  created_at,
  INDEX(user_id), INDEX(jd_id))

output_files(
  id, generation_id, type,                      -- pdf | docx
  path, meta_json JSON, created_at,             -- path 指向本机文件
  INDEX(generation_id))

audit_log(
  id, user_id, operation,                        -- interview|ingest|query|lint|render
  model, cost, source_trace TEXT, status, created_at,
  INDEX(user_id), INDEX(created_at))             -- 见 M7-FR-06
```
- `asked_set_json` 与 M2 会话状态字段一致（M2 prompt 注入仅用 covered/skipped，asked_set 用于服务端防重复）。
- 大 JSON（structured_data/source_trace）作为 JSON/TEXT 列；如单行过大再评估拆表。
- 后续表：`generation_versions`（版本历史）、`profiles`（画像）、`wiki_pages`（页索引缓存）。

## 5. 功能需求
| 编号 | 需求 | 优先级 |
|---|---|---|
| M8-FR-01 | 按 user_id 隔离文件目录与 MySQL 行。 | MVP |
| M8-FR-02 | 提供文件落盘与读取（raw/wiki/outputs/uploads）。 | MVP |
| M8-FR-03 | git 初始化、提交、回滚（每次 Ingest/Lint 提交）。 | MVP |
| M8-FR-04 | 提供 §4 核心表的读写。 | MVP |
| M8-FR-05 | 二进制存文件、MySQL 存指针（path+meta），不存大文件入库。 | MVP |
| M8-FR-06 | （可选）配置私有 GitHub remote 并 `git push` 备份（SSH key/PAT 走环境变量）。事实源单机单点为**已知接受风险**（个人自用）。 | 可选 |
| M8-FR-07 | 核心表带索引（user_id/generation_id/jd_id 等高频字段）。 | MVP |
| M8-FR-08 | MySQL 为独立服务，其自身备份由该服务负责（本系统不另做）。 | 说明 |
| M8-FR-09 | 版本历史表 / 画像表 / 页索引缓存表。 | 后续 |

## 6. 依赖
- 被几乎所有模块依赖（文件+数据落地）。git 操作配合 M3；审计表配合 M7。

## 7. 验收标准
- ✅ 文件按用户隔离布局；每次编译有 git 提交、可回滚。
- ✅ MySQL 核心表可读写；二进制只存指针。
- ✅ （若开启）能 push 到私有 GitHub remote。
