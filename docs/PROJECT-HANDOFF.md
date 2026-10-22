# PickData Copilot · 项目交接文档

> 生成时间：2026-10-21
> 用途：新会话接手时先读这份文档，不要重复推导已经定过的设计。

---

## 0. 一句话定位

面向大数据时代的**通用型数据 Agent**：把自然语言问数、指标分析、异常检测与波动归因做成完整闭环，并把 Agent 的数据检索、SQL 生成、安全校验、查询执行和分析解读过程做成**可实时查看、可回放、可追溯**的 Trace。架构不绑定具体行业，首个物流拣运场景仅用于验证通用能力。

参考来源：简历项目《Story Maker · Agent 应用开发》里的 New-Agent Runtime / 拜访通。
原始 PDF：`c:\Users\wangzhao\.trae-cn\attachments\766f8133-5b96-4447-8856-7701722e8bae_ab9e928a-eac5-4bd9-8ab3-245759fca0ab_Story Maker_Agent应用开发.pdf`

---

## 1. 背景与定位

| 项目 | 内容 |
|---|---|
| 用户背景 | 数据平台与 AI Agent 工程实践 |
| 目标用户 | 业务运营、数据分析师、研发人员及需要快速理解数据的决策者 |
| 核心问题 | 数据分散、口径不统一、查询依赖 SQL、分析过程不透明、结果难以追溯 |
| 业务场景 | 自然语言问数、指标分析、异常检测、波动归因和结果解读 |
| 产品定位 | 面向多行业、多数据源的通用型数据 Agent |
| 核心卖点 | 全链路可观测，而不是只让模型生成 SQL |
| 首个演示场景 | 使用物流拣运样例数据验证问数与归因闭环，架构与其他行业解耦 |
| 项目目标 | 求职作品集，要求能本地一键跑起来、可演示、面试讲得清 |

**结论先行**：这个项目解决的是大数据环境下「数据很多，但业务人员获取洞察仍然困难」的问题。卖点是「Agent 执行过程可查看、可解释、可续跑」，所以设计顺序是：
先定 Trace 事件模型与状态机 → 再写 Agent 逻辑 → 最后接模型与中间件。

---

## 2. 协作约定（新会话必须遵守）

1. **代码由用户自己写，AI 不代写。**
2. AI 只负责：
   - 写之前：给模块职责、接口契约、字段清单、验收标准
   - 写之后：review 代码，指出问题和修改方向
   - 跑不通：定位原因，说明改哪一行
3. 只有用户明确说「帮我写这个文件」，才可以创建或修改代码。
4. 按层推进，每层写完、用户确认后再进入下一层。
5. 不要恢复已被删除的 L0 实现；那是一轮越界产物，用户明确要求清空。

> 历史教训：曾一次性写完 L0 的 8 个契约文件 + 27 个测试 + 53 个模块占位，用户不满，全部删除，只保留目录骨架。**不要再犯。**

---

## 3. 核心设计原则

1. **事件是事实，其他都是投影。** 任务状态、工具轨迹、步骤耗时全部由事件流派生。
2. **SSE 只是视图。** 断线/刷新后按 `seq` 从事件表续读，不能只靠内存转发。
3. **依赖只能向下。** 上层调下层接口，下层禁止反向 import。
4. **每个 Port 至少两个适配器。** 真实中间件 + 测试替身，业务代码不认识驱动。
5. **append-only。** 压缩不删原文，只追加摘要事件。

---

## 4. 技术栈与中间件

### 4.1 技术栈

| 层次 | 选型 | 用途 | 阶段 |
|---|---|---|---|
| 语言 | Python 3.12 | 全项目 | 已具备 |
| Web | FastAPI + Uvicorn | REST + SSE | 一期 |
| Agent Runtime | 自研（FSM + Tool Loop） | 编排、断点续跑 | 一期 |
| 模型接入 | OpenAI 兼容协议 + httpx | DeepSeek / 通义 / OpenAI 可切 | 一期 |
| 关系库 | MySQL 8 + SQLAlchemy 2.0 async | 会话、任务、事件 | 一期 |
| 缓存 | Redis 7 + `redis.asyncio` | 会话锁、幂等、检索缓存 | 一期 |
| 消息队列 | RocketMQ 5.x | 异步归因 | 二期 |
| 向量库 | Milvus 2.5 + pymilvus | 指标语义检索 | 一期 |
| SQL 解析 | sqlglot | AST 级安全网关 | 一期 |
| 前端 | 原生 HTML/JS + SSE + SVG | Trace 时间轴面板 | 一期 |
| 部署 | Docker Compose | 中间件一键起 | 一期 |
| 测试 | pytest | 契约与单测 | 一期 |
| 配置 | pydantic-settings + `.env` | 多环境切换 | 一期 |

### 4.2 中间件

| 中间件 | 版本 | 职责 | 备注 |
|---|---|---|---|
| MySQL | 8.0 | 业务库 + 样例数仓 | 3 张元数据表 + 1 张事实表 |
| Redis | 7.x | 会话级分布式锁、幂等、检索缓存 | 不存业务事实 |
| RocketMQ | 5.x | 归因任务异步化 | NameServer + Broker |
| Milvus | 2.5 | 指标目录向量检索 | 依赖 etcd + MinIO |

### 4.3 明确不用的东西

- **不用 LangChain / LangGraph**：框架会把执行细节藏起来，与「全链路可观测」冲突。
- **一期不引入 Kafka / Hive / ES / ASR**：先控制 MVP 范围，后续通过 Port 与 Adapter 按数据源和业务规模扩展。
- **不直接依赖 Pi Agent**：它是 Node/TypeScript 生态，Python 无法复用，只借鉴设计。

---

## 5. 分层与分包

### 5.1 七层模型

```
L6 展示层       Trace 时间轴页面（只走 HTTP/SSE）
      ↓
L5 接口层       REST / SSE / 事件回放 / 会话与任务查询
      ↓
L4 编排层       FSM / Runtime 主循环 / Skill 装载 / 上下文 / 续跑 / HITL
      ↓
L3 能力层       工具注册 / 工具实现 / 权限判定 / LLM 网关 / 事件发射
      ↓
L2 数据领域层   指标语义 / SQL 编译 / SQL 安全 / 查询执行 / 异常判定 / 归因
      ↓
L1 基础设施层   MySQL·Redis·Milvus·RocketMQ 适配器
      ↓
L0 契约层       枚举 / 事件 / Port / 领域结构 / 错误（零依赖）
```

### 5.2 Java 分层对照

| Java 里的层 | 本项目包 |
|---|---|
| Controller | `app/api/` |
| Service | `app/runtime/` |
| Domain Service | `app/domain/` |
| Manager / Client | `app/capabilities/` |
| DAO / Mapper | `app/infra/db/` |
| RPC / 外部客户端 | `app/infra/{cache,vector,mq,llm}/` |
| DTO / 枚举 / SPI | `app/contracts/` |
| Config / 组合根 | `app/infra/config.py`、`app/infra/bootstrap.py` |
| Static | `app/web/` |

### 5.3 当前目录（仅骨架，无实现）

```
pickdata-agent/
├── app/
│   ├── api/
│   ├── capabilities/
│   │   ├── prompts/
│   │   └── tools/
│   ├── contracts/
│   ├── domain/
│   ├── infra/
│   │   ├── cache/
│   │   ├── db/
│   │   ├── llm/
│   │   ├── mq/
│   │   └── vector/
│   ├── runtime/
│   ├── seed/
│   ├── skills/
│   ├── web/
│   └── __init__.py          # 空文件
├── deploy/
├── tests/
│   ├── api/
│   ├── capabilities/
│   ├── domain/
│   ├── infra/
│   └── runtime/
└── pyproject.toml
```

项目路径：`D:\python\projectCode\pickdata-agent`

---

## 6. Pi Agent 对齐（已确认三项全对齐）

来源：`https://github.com/earendil-works/pi`（即 pi.dev）
文档：
- `packages/coding-agent/docs/sessions.md`
- `packages/coding-agent/docs/session-format.md`
- `packages/coding-agent/docs/compaction.md`
- `packages/coding-agent/docs/skills.md`

### 6.1 会话树

| 项目 | 设计 |
|---|---|
| 会话 | `chat_session(session_id, title, active_leaf_id, version, parent_session_id)` |
| 节点 | 复用 `agent_event`，`parent_id` 指向父事件 |
| 分支 | 不建 branch 表；分支 = 根到 `active_leaf_id` 的路径 |
| 切换 | `/tree` 改 `active_leaf_id`，旧节点不删 |
| 分叉 | `/fork` 复制到新 session，`parent_session_id` 指回原会话（二期） |
| 上下文 | 只取 active branch |

### 6.2 压缩（Compaction）

| 项目 | 设计 |
|---|---|
| 触发 | `context_tokens > context_window - 16384`；保留最近约 20k token；支持手动 |
| 落库 | 追加 `CONTEXT_COMPACTED` 事件 |
| 原文 | 永久保留，只是不再进入模型上下文 |
| 重建 | system prompt + 当前分支最新摘要 + `first_kept_event_id` 之后的节点 |
| 重复压缩 | 从上一条的 `first_kept_event_id` 开始截取 |

### 6.3 Skills 渐进披露

| 项目 | 设计 |
|---|---|
| 存放 | `app/skills/<skill-name>/SKILL.md`，可带 `references/`、`scripts/`、`assets/` |
| 启动 | 只注入 `name`、`description`、路径 |
| 运行时 | 模型调用 `load_skill` 工具读取正文，记 `SKILL_LOADED` |
| 工具边界 | frontmatter `allowed-tools` |
| 命名 | 小写 + 连字符，≤64 字符（Agent Skills 规范） |

### 6.4 对契约的改动

| 层 | 文件 | 改动 |
|---|---|---|
| L0 | `contracts/enums.py` | 新增 `BRANCH_CREATED`、`BRANCH_SUMMARIZED`、`CONTEXT_COMPACTED`、`SKILL_LOADED`；新增 `Stage.COMPACT`、`Stage.SKILL_LOAD` |
| L0 | `contracts/events.py` | 补 4 个新事件的 payload 必填键 |
| L0 | `contracts/skills.py` | `SkillSpec` → `SkillMeta` + `SkillDocument` |
| L1 | `infra/db` | session 加 `active_leaf_id/version/parent_session_id`；event 加 `branch_id` |
| L3 | `capabilities/tools/` | 新增 `load_skill`（只读） |
| L4 | `runtime/context_builder.py` | 按 active branch + 最新压缩摘要重建 |
| L4 | `runtime/checkpoint.py` | 从 `first_kept_event_id` 还原 |

### 6.5 不照搬

- JSONL 文件存储 → 改用 MySQL 事件表
- TypeScript SDK / Extension Hooks → 跨语言，不复用
- system message 按 section 打补丁 → 简化成上下文快照

---

## 7. L0 契约设计（尚未实现）

> 这一节是已经对齐过的契约。**用户需要自己按此实现，AI 不要直接写文件。**

### 7.1 枚举

**RunState（9）**

```
CREATED, UNDERSTANDING, RETRIEVING, COMPILING, EXECUTING,
ANALYZING, WAITING_CONFIRM, COMPLETED, FAILED
```

**Stage（15，含 Pi 对齐新增）**

```
CONTEXT, UNDERSTAND, RETRIEVE, SLOT_FILL, COMPILE, GUARD,
EXECUTE, ANOMALY, ANALYZE, ATTRIBUTE(预留), CONFIRM,
PERSIST, DONE, COMPACT, SKILL_LOAD
```

**EventType（22）**

```
RUN_CREATED RUN_STARTED RUN_RESUMED RUN_COMPLETED RUN_FAILED
STATE_CHANGED
STAGE_ENTERED STAGE_EXITED
LLM_REQUEST LLM_TOKEN LLM_RESPONSE
TOOL_REQUESTED TOOL_POLICY_CHECKED TOOL_RESULT TOOL_BLOCKED
ARTIFACT_CREATED
HITL_REQUESTED HITL_RESOLVED
BRANCH_CREATED BRANCH_SUMMARIZED CONTEXT_COMPACTED SKILL_LOADED
```

**其余枚举**

| 枚举 | 值 |
|---|---|
| Actor | SYSTEM / AGENT / LLM / TOOL / USER |
| EventStatus | STARTED / SUCCEEDED / FAILED / BLOCKED / SKIPPED |
| RiskLevel | READ / WRITE / DANGEROUS |
| ArtifactKind | CONTEXT / SLOTS / METRIC / SQL / RESULT_SET / SERIES / CHART / ATTRIBUTION / ANSWER |
| ErrorCode | USER_INPUT_INVALID / SLOT_MISSING / METRIC_NOT_FOUND / LLM_ERROR / LLM_OUTPUT_INVALID / POLICY_BLOCKED / TOOL_ERROR / DATA_ERROR / STATE_CONFLICT / TIMEOUT / INTERNAL |

### 7.2 TraceEvent 字段

| 字段 | 说明 |
|---|---|
| `event_id` | UUID |
| `run_id` / `session_id` | 必填 |
| `branch_id` | 分支 ID，可空 |
| `seq` | run 内严格递增，由 TraceStore 分配；0 表示未分配 |
| `ts` | UTC ISO8601；naive 时间要补 UTC |
| `state` / `stage` / `type` / `actor` | 枚举 |
| `name` | 人类可读，如 `metric_search` |
| `status` | EventStatus |
| `latency_ms` | 可空，>=0 |
| `parent_id` | 父事件，支持投影成树 |
| `attempt` | 默认 1 |
| `schema_version` | 默认 1 |
| `payload` | 落库前必须脱敏 |

### 7.3 payload 必填键

| 事件 | 必填键 |
|---|---|
| RUN_CREATED | user_input, skill |
| RUN_STARTED | state |
| RUN_RESUMED | from_state, from_seq |
| RUN_COMPLETED | answer |
| RUN_FAILED | error_code, message, failed_stage |
| STATE_CHANGED | from_state, to_state, reason |
| LLM_REQUEST | provider, model, messages |
| LLM_TOKEN | index, text |
| LLM_RESPONSE | provider, model, content |
| TOOL_REQUESTED | tool, args |
| TOOL_POLICY_CHECKED | tool, risk, allowed, reason |
| TOOL_RESULT | tool, ok, summary |
| TOOL_BLOCKED | tool, reason_code, rule |
| ARTIFACT_CREATED | kind, name, summary |
| HITL_REQUESTED | action, confirm_token, preview |
| HITL_RESOLVED | approved, actor |
| BRANCH_CREATED | from_event_id, branch_id |
| BRANCH_SUMMARIZED | branch_id, summary, first_kept_event_id |
| CONTEXT_COMPACTED | summary, first_kept_event_id, tokens_before, keep_recent_tokens |
| SKILL_LOADED | skill, path, trigger |

`STAGE_ENTERED` / `STAGE_EXITED` 无必填键。

### 7.4 状态迁移表

| From | 允许的 To |
|---|---|
| CREATED | UNDERSTANDING |
| UNDERSTANDING | RETRIEVING, WAITING_CONFIRM, FAILED |
| RETRIEVING | COMPILING, WAITING_CONFIRM, FAILED |
| COMPILING | EXECUTING, WAITING_CONFIRM, FAILED |
| EXECUTING | ANALYZING, WAITING_CONFIRM, FAILED |
| ANALYZING | COMPLETED, WAITING_CONFIRM, FAILED |
| WAITING_CONFIRM | EXECUTING, ANALYZING, COMPLETED, FAILED |
| COMPLETED | — |
| FAILED | 仅 resume 时回到 `failed_stage` 对应状态 |

Stage→State 恢复映射：
CONTEXT/UNDERSTAND → UNDERSTANDING；RETRIEVE/SLOT_FILL → RETRIEVING；
COMPILE/GUARD/COMPACT → COMPILING；EXECUTE → EXECUTING；
ANOMALY/ANALYZE/ATTRIBUTE/PERSIST/DONE → ANALYZING；CONFIRM → WAITING_CONFIRM；
SKILL_LOAD → 保持当前状态（实现时单独处理）。

### 7.5 领域结构

- `TimeRange`：`label, start, end, baseline_start, baseline_end`；当前窗口与基期等长，基期必须在当前窗口之前。
- `QuerySlots`：`time, metric_code, dimensions, filters, need_attribution, confidence(0~1), clarify, confirm_action`。
- `MetricDef`：`code, name, description, unit, table, agg_expr, aliases, dimensions, higher_is_better, precision, owner`。
- `ToolSpec`：`name, description, risk, input_schema, timeout_sec, idempotent, requires_confirm`。
  - WRITE 必须 `requires_confirm=True`
  - DANGEROUS 不允许确认，只能禁用
- `ToolResult`：`ok, data, summary, row_count, elapsed_ms, error`；失败必须带 `ErrorInfo`。
- `SkillMeta`：`name, description, path, allowed_tools, intents`
- `SkillDocument`：`meta, content, references, scripts`
- `LLMResponse`：`content, provider, model, usage, finish_reason, latency_ms`
- `ErrorInfo`：`code, message, retryable, failed_stage`
- `RunRecord`：run 的投影，含 `state, attempt, failed_stage, answer, error, artifacts`
- `SessionRecord`：`session_id, title, active_leaf_id, version, parent_session_id`

### 7.6 Ports

```
TraceStorePort       append(event)->event / list_after(run_id,seq,limit) / last_seq(run_id)
RunRepositoryPort    create / get / save
SessionRepositoryPort create / get / update_active_leaf / list_runs
CachePort            get_json / set_json / delete / acquire_lock / release_lock / once
VectorPort           upsert(collection, docs) / search(collection, query, top_k, filters)
MessageBusPort       publish(topic, payload) / subscribe(topic, handler)
LLMPort              chat(messages,...)->LLMResponse / stream(...)->AsyncIterator[str]
EmbeddingPort        embed(texts)->list[list[float]]
```

### 7.7 错误与脱敏

- 可重试：`TIMEOUT / LLM_ERROR / TOOL_ERROR / DATA_ERROR`
- 不可重试：`POLICY_BLOCKED / STATE_CONFLICT / LLM_OUTPUT_INVALID / USER_INPUT_INVALID / SLOT_MISSING / METRIC_NOT_FOUND / INTERNAL`
- 脱敏：字段名黑名单（api_key / token / password / secret / authorization / cookie / credential）+ 正则掩码手机号、邮箱、身份证。
- 原始输入不能被修改，返回深拷贝。

### 7.8 seq 语义

- `seq=0` 表示未分配。
- TraceStore 在锁内分配并返回落库后的新事件。
- 并发 append 必须严格递增，`UNIQUE(run_id, seq)` 兜底。
- SSE 用 `Last-Event-ID` 作为已读 seq，`list_after` 只返回更大 seq。

---

## 8. 数据模型

### 8.1 三张表

**chat_session**

| 字段 | 说明 |
|---|---|
| session_id | PK |
| title | 会话名 |
| active_leaf_id | 当前分支末端事件 ID，可空 |
| version | 会话结构版本，默认 1 |
| parent_session_id | fork 来源，可空 |
| created_at | |

**agent_run**

| 字段 | 说明 |
|---|---|
| run_id | PK |
| session_id | 索引 |
| user_input | |
| skill_name | |
| state | 索引 |
| attempt | 默认 1 |
| failed_stage | 可空 |
| answer | |
| error_json | |
| artifacts_json | |
| last_seq | 该 run 的事件计数器 |
| created_at / updated_at | |

**agent_event**

| 字段 | 说明 |
|---|---|
| event_id | PK |
| run_id / session_id | |
| branch_id | 可空 |
| seq | run 内递增 |
| parent_id | 父事件 |
| ts / state / stage / type / actor / name / status | |
| latency_ms / attempt / schema_version | |
| payload_json | 脱敏后 |

索引：
- `UNIQUE(run_id, seq)`
- `INDEX(run_id, seq)`
- `INDEX(session_id, ts)`
- `INDEX(branch_id, ts)`

### 8.2 seq 原子分配

```
BEGIN
  SELECT last_seq FROM agent_run WHERE run_id = ? FOR UPDATE
  UPDATE agent_run SET last_seq = last_seq + 1
  INSERT INTO agent_event (seq = last_seq + 1, ...)
COMMIT
```

单 run 单写者，行锁竞争很低。

### 8.3 LLM_TOKEN 落库策略

**聚合落库**：每 50 个 token 或每 500ms 合并成一条 `LLM_TOKEN` 事件。
前端仍逐字推送，数据库只保留块级轨迹，避免事件表爆炸。

### 8.4 样例数仓

表：`dwd_pick_order_di`

| 字段 | 说明 |
|---|---|
| dt | DATE |
| region | 华东 / 华北 / 华南 / 西南 |
| warehouse | 上海拣运中心 / 北京拣运中心 / 广州拣运中心 / 成都拣运中心 |
| pick_type | 人工拣货 / 自动分拣 / 夜间补货 |
| shift | 白班 / 夜班 |
| order_cnt | 拣货单量 |
| on_time_cnt | 及时单量 |
| pick_duration_sec | 拣货总时长（秒） |
| labor_hours | 人时 |

数据量：约 90 天 × 4 区域 × 4 仓 × 3 类型 = 4320 行。

**埋入的异常**：最近 7 天，华东 / 上海拣运中心 / 夜间补货 的人均拣货时长 +30%~35%，及时率 -6pp。
这样归因链路能真实找到「上海仓夜间补货」这个主因。

### 8.5 指标目录

| code | 名称 | 口径 | 单位 | 方向 |
|---|---|---|---|---|
| avg_pick_duration | 平均拣货时长 | `SUM(pick_duration_sec)/SUM(order_cnt)` | 秒 | 越低越好 |
| pick_timeliness_rate | 拣货及时率 | `SUM(on_time_cnt)/SUM(order_cnt)*100` | % | 越高越好 |
| pick_order_cnt | 拣货单量 | `SUM(order_cnt)` | 单 | 中性 |

维度：`region / warehouse / pick_type / shift`。

---

## 9. 首个业务闭环

### 9.1 流程

```
用户提问
  → 意图识别（问数 / 归因 / 推送）
  → 时间解析（当前窗口 + 等长基期）
  → 指标语义检索（Milvus）
  → 槽位抽取（LLM + 规则校验）
  → SQL 编译（指标模板 + 参数）
  → SQL 安全网关（sqlglot）
  → 执行查询
  → 异常判定（环比 / 标准差）
  → 结果解读（LLM 流式）
  → 图表配置 + Trace 面板
```

### 9.2 SQL 安全规则

- 只允许单条 SELECT
- 表白名单：`dwd_pick_order_di`
- 强制 LIMIT（默认 500）
- 禁止 DDL/DML、多语句、`INTO OUTFILE`、`SLEEP`、`LOAD_FILE`
- 非白名单表、非 SELECT 一律 `POLICY_BLOCKED` 并落事件

### 9.3 归因（二期）

- 触发：问题包含「为什么 / 原因 / 下降 / 上升 / 异常」
- 维度下钻：region / warehouse / pick_type / shift
- 计算：当前窗口 vs 基期，按维度值算差异与权重，输出 Top 贡献项
- 执行方式：RocketMQ 异步任务，消费者回写 run 结果，前端轮询或续流

### 9.4 HITL

- 写工具（推送报告）必须人工确认
- 任务进入 `WAITING_CONFIRM`，生成 `confirm_token`
- 确认后继续执行，拒绝则记录 `HITL_RESOLVED` 并结束

---

## 10. 建设顺序与验收

| 顺序 | 层 | 验收标准 |
|---|---|---|
| 1 | L0 契约层 | 枚举完整；事件序列化往返；非法迁移被拒；Port 可被 mock |
| 2 | L1 基础设施 | 三张表可建；seq 并发严格递增；重启后事件可回放 |
| 3 | L2 领域层 | SQL 编译快照；安全网关拦截 DELETE / 多语句 / 非白名单表；样例数仓可查 |
| 4 | L3 能力层 | 每次工具调用都有事件；越权调用被拦截并留痕 |
| 5 | L4 编排层 | Mock LLM 跑通一句问数，事件序列符合预期 |
| 6 | L5 接口层 | SSE 断线重连按 seq 续传，不丢不重 |
| 7 | L6 展示层 | 实时看一条 run，也能回放历史 run |

一期不做：真实模型调用（先用 Mock）、异步归因、HITL、权限鉴权、多租户、部署脚本。

---

## 11. 当前状态

| 项目 | 状态 |
|---|---|
| 项目路径 | `D:\python\projectCode\pickdata-agent` |
| 目录骨架 | 已按图片搭好，只有目录 + 空的 `app/__init__.py` + `pyproject.toml` |
| 代码 | **没有任何实现**，用户要求自己写 |
| L0 实现 | 已删除（曾写过 8 个文件 + 27 个测试，属越界） |
| 备份 | `C:\Users\wangzhao\Documents\Trae-CN\temp_20261021_01\pickdata-agent-l0-backup.zip`（临时目录，可删） |
| 本机环境 | Python 3.12.7 + pytest 7.4.4；**没有 Docker，没有模型 API Key** |
| 旧路径 | `D:\python\projectCode\visit-tong\pickdata-agent` 已迁走并删除 |

---

## 12. 下一步

用户准备从 `app/contracts/enums.py` 开始自己写。

建议先写的顺序：

1. `app/contracts/enums.py`
   - 9 个 RunState
   - 15 个 Stage
   - 22 个 EventType
   - Actor / EventStatus / RiskLevel / ArtifactKind / ErrorCode
2. `app/contracts/events.py`
   - TraceEvent 字段 + payload 必填校验 + 脱敏
3. `app/contracts/state.py`
   - 迁移表 + 纯函数校验
4. `app/contracts/domain.py`
5. `app/contracts/tools.py` / `skills.py` / `ports.py`

**AI 的正确动作**：等用户提交代码，然后 review。不要主动创建文件。

---

## 13. 风险与待定

| 风险 | 说明 | 当前决定 |
|---|---|---|
| RocketMQ Python 客户端 | 3.12 + Windows 可能无预编译包，源码编译需要 CMake/librocketmq | 官方客户端优先，装不上用兼容实现兜底，MessageBusPort 不变 |
| 向量来源 | 没有模型 Key 就无法调 embedding | 先用确定性哈希向量（256 维）跑通 Milvus，接口可切厂商 embedding |
| 本机无 Docker | 真实中间件无法本地验证 | 用户侧起 Docker；本地先用测试替身验证契约 |
| 事件表膨胀 | 每次问答事件较多 | LLM_TOKEN 聚合落库；其余事件按阶段性保留 |
| 简历措辞 | 不能写「使用 Pi Agent」 | 只能写「参考 Pi Agent 的会话管理与上下文工程设计」 |

---

## 14. 简历六条亮点映射

| 简历亮点 | 本项目落地 | 差异 |
|---|---|---|
| 可插拔 Agent Runtime | Port + Skill 目录 + Tool 注册表 | 自研 Runtime 替代 Pi Agent |
| Context Engineering & Durable Execution | 会话树 + 压缩事件 + 断点续跑 | 一期先同步链路，续跑二期 |
| 全流程可观测 | 22 类事件 + SSE 续传 + 时间轴面板 | 升级为第一优先级 |
| 业务 Agent 编排 | 9 段 FSM + 总控 Skill + 槽位抽取 | 用指标语义层替代 BizSessionState |
| 可靠性与安全治理 | SQL 网关 + 三级权限 + HITL + 幂等 + 锁 | 降级原生表单不做 |
| 知识与多模态链路 | 指标目录 embedding → Milvus | 不做 ASR / hive2kafka |

一句话总结：**换业务域，不换骨架。**

---

## 15. 参考资料

- 简历 PDF：`c:\Users\wangzhao\.trae-cn\attachments\766f8133-5b96-4447-8856-7701722e8bae_ab9e928a-eac5-4bd9-8ab3-245759fca0ab_Story Maker_Agent应用开发.pdf`
- Pi 仓库：https://github.com/earendil-works/pi
- 会话与上下文：https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/sessions.md
- 会话文件格式：https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/session-format.md
- 压缩参考：https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/compaction.md
- Skills：https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/skills.md
