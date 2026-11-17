# Memory v2：时序学习者记忆（设计稿）

状态：草案，待评审。不改变公开 API，前端在第一阶段无需改动。

## 1. 背景

### 1.1 现状

```
afterReview ─► LLM 提取 ─► profile.json（画像主数据，整文件读写，进程内锁）
                              ├ weak_points / strong_points / behavior_signals（文本即身份）
                              ├ topic_mastery / stats / score_history
                              ├ session_extractions（最近 100 条，去重用）
                              └ _pending_memory（向量写入前的补偿快照）
                    └─► SQLite memory_vectors（session_summary / insight / weak_point）
读取：ProfileService.summary() 拼接文本 → 各模式 prompt
```

### 1.2 问题

| # | 问题 | 位置 | 影响 |
| --- | --- | --- | --- |
| P1 | insight 把当时的薄弱点写死，改善后仍被检索召回 | `profile-service.ts` `sessionMemory` | 注入过时事实，只能靠 prompt 声明兜底 |
| P2 | 规律归纳会归档具体薄弱点，`dueReviews` 又排除归档项和规律 | `consolidate`、`dueReviews` | 被归纳的知识点不再进入 SM-2 复习 |
| P3 | `weak_point` 向量只写不读 | `sessionMemory`、`consolidate`、`semanticHistory` | 浪费 Embedding 调用和存储 |
| P4 | 语义检索只在专项训练生效 | `resume-state-machine.ts`、`recording-service.ts`、`prep-service.ts` 调用 `summary(userId)` | 简历、录音、Copilot 拿不到历史会话洞察 |
| P5 | 画像与向量分属两个存储，靠 `_pending_memory` 手写补偿；锁仅在进程内 | `profile-repository.ts` | 多进程不安全，整文件重写，数组无限增长 |
| P6 | 事实没有稳定 ID，以文本为身份，归纳靠数组下标对齐 | `feedback`、`merge.ts`、`consolidate` | 无法改名，反馈和合并脆弱 |
| P7 | 去重只靠固定阈值；Embedding 失败时退化为精确匹配 | `bestTextMatch`、`vectorsFor` | 重复薄弱点，阈值与模型绑定却未校准 |
| P8 | 个人 Agent 对话不回写长期记忆 | `personal-agent-service.ts` | 用户自述的目标、时间、偏好丢失 |

## 2. 目标与非目标

目标：

- **时序正确**：事实有状态变化历史，检索结果按当前状态展示。
- **单事务一致**：画像事实、事件、向量在同一个 SQLite 事务中写入，去掉补偿快照。
- **可追溯**：每次状态变化都能指向来源会话和证据。
- **可度量**：去重、时效、检索、成本都有离线评测和基线。
- **全模式覆盖**：所有模式和个人 Agent 通过同一个上下文装配器使用记忆。
- **可重建**：向量是纯派生索引，换模型或换算法时可重放。

非目标：

- 不引入图数据库、独立向量数据库或 Python 服务。
- 不让 LLM 自主编辑学习者模型；数值状态（掌握度、复习调度）只由确定性算法更新。
- 不做多设备实时同步；归档导入仍是离线合并。

## 3. 选型

| 方案 | 结论 | 借鉴点 |
| --- | --- | --- |
| Mem0 | 不采用：扁平字符串事实无法承载 SM-2、掌握度等结构化状态，且会成为第三个存储 | 召回相似旧事实后由 LLM 判定 ADD / UPDATE / NOOP，用于模糊区间对齐和 Agent 回写 |
| Zep / Graphiti | 不采用：需要图数据库和 Python，每段内容多次 LLM 调用，与 Electron + Bun + 自带 Key 冲突；实体类型少且固定，用不上开放 schema | 双时间轴（发生时间 / 记录时间）、事实失效而非覆盖、事实到来源会话的追溯 |
| Letta | 不采用：需要接入整个运行时；Agent 自编辑与确定性算法、严格校验相冲突 | 有预算的常驻上下文块、后台整理（sleep-time） |
| LangMem | 不采用：仅 Python，绑定 LangGraph | profile（单 schema 文档）与 collection（多条记忆）的区分 |

本项目的记忆对象本质是**学习者模型**（智能辅导系统中的 learner model）：有限的实体类型 + 数值状态 + 少量情景记录。单用户只有几百到几千场会话，SQLite 加应用层余弦计算足够。

## 4. 总体架构

```
            写入（每场复盘一次）                      读取（出题 / 评估 / Agent）
                    │                                         ▲
                    ▼                                         │
 ┌──────────── MemoryWriter ─────────────┐     ┌──── MemoryContextBuilder ────┐
 │ 1 提取（LLM，结果落库，不重复调用）   │     │ 按用途选查询、分段、按预算裁剪 │
 │ 2 对齐（精确 → 阈值 → LLM 仲裁）       │     │ 情景按当前事实状态渲染         │
 │ 3 单事务应用（事件 + 投影 + 向量）     │     └──────────────▲───────────────┘
 └──────────────────┬────────────────────┘                    │
                    ▼                                         │
  ┌─────────────────────────────── SQLite ───────────────────┴───────────┐
  │ memory_episodes   情景层：每场会话的提取结果与摘要（不可变）         │
  │ memory_events     事件层：只追加的状态变化与证据                     │
  │ memory_facts      事实层：当前状态投影（可由事件重放）               │
  │ memory_fact_links 规律与支撑事实                                     │
  │ memory_topic_mastery / memory_scores  掌握度与得分                   │
  │ memory_vectors    索引层：纯派生，可删除重建                         │
  └──────────────────────────────────────────────────────────────────────┘
  profile.json 只保留用户手填字段（name、target_role 等）
```

## 5. 数据模型

```sql
-- 情景层：替代 session_extractions 缓存与 _pending_memory
CREATE TABLE memory_episodes (
  user_id TEXT NOT NULL,
  session_id TEXT NOT NULL,
  mode TEXT NOT NULL,
  topic TEXT,
  summary TEXT NOT NULL DEFAULT '',
  extraction TEXT NOT NULL,            -- 校验后的 LLM 提取结果，写入后不可变
  extractor_version TEXT NOT NULL,     -- 提示词 / schema 版本
  state TEXT NOT NULL,                 -- extracted | applied
  occurred_at TEXT NOT NULL,           -- 会话发生时间
  recorded_at TEXT NOT NULL,           -- 写入时间
  applied_at TEXT,
  PRIMARY KEY (user_id, session_id)
);

-- 事实层：当前状态投影
CREATE TABLE memory_facts (
  fact_id TEXT PRIMARY KEY,            -- ULID
  user_id TEXT NOT NULL,
  kind TEXT NOT NULL,                  -- weak | strong | behavior | pattern | stated
  axis TEXT NOT NULL DEFAULT 'knowledge',
  topic TEXT,
  text TEXT NOT NULL,
  norm_key TEXT NOT NULL,              -- 规范化文本，精确匹配
  source TEXT NOT NULL,                -- observed | predicted | consolidated | user_stated
  status TEXT NOT NULL,                -- active | improved | archived | merged
  merged_into TEXT,
  confidence REAL,
  times_seen INTEGER NOT NULL DEFAULT 1,
  first_seen TEXT NOT NULL,
  last_seen TEXT NOT NULL,
  valid_to TEXT,                       -- stated 类事实的失效时间
  schedule TEXT,                       -- 复习调度状态 JSON
  next_review TEXT,                    -- 冗余列，便于查询到期复习
  attrs TEXT NOT NULL DEFAULT '{}',    -- 行为信号的 namespace、polarity 等
  version INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT NOT NULL
);
CREATE INDEX idx_mf_user_kind ON memory_facts(user_id, kind, status);
CREATE INDEX idx_mf_user_topic ON memory_facts(user_id, topic);
CREATE INDEX idx_mf_user_review ON memory_facts(user_id, next_review);
CREATE UNIQUE INDEX idx_mf_norm ON memory_facts(user_id, kind, topic, norm_key) WHERE status != 'merged';

-- 事件层：只追加
CREATE TABLE memory_events (
  event_id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  fact_id TEXT NOT NULL,
  session_id TEXT,                     -- 用户反馈等无会话来源时为空
  type TEXT NOT NULL,                  -- created | observed | reviewed | improved | regressed
                                       -- | merged | archived | unarchived | confirmed | refuted | expired
  score REAL,
  evidence TEXT,
  occurred_at TEXT NOT NULL,
  recorded_at TEXT NOT NULL
);
CREATE INDEX idx_me_fact ON memory_events(fact_id, occurred_at);
CREATE UNIQUE INDEX idx_me_once ON memory_events(user_id, fact_id, session_id, type) WHERE session_id IS NOT NULL;

CREATE TABLE memory_fact_links (
  user_id TEXT NOT NULL,
  pattern_id TEXT NOT NULL,
  fact_id TEXT NOT NULL,
  PRIMARY KEY (pattern_id, fact_id)
);

CREATE TABLE memory_topic_mastery (
  user_id TEXT NOT NULL, topic TEXT NOT NULL,
  score REAL, session_count INTEGER NOT NULL DEFAULT 0, notes TEXT,
  last_assessed TEXT, retrospective TEXT, retrospective_at TEXT,
  PRIMARY KEY (user_id, topic)
);

CREATE TABLE memory_scores (
  user_id TEXT NOT NULL, session_id TEXT NOT NULL, mode TEXT NOT NULL, topic TEXT,
  avg_score REAL NOT NULL, dimension_scores TEXT, occurred_at TEXT NOT NULL,
  PRIMARY KEY (user_id, session_id)
);

-- 索引层：沿用 memory_vectors，新增列
ALTER TABLE memory_vectors ADD COLUMN fact_id TEXT;
ALTER TABLE memory_vectors ADD COLUMN embedding_model TEXT;
-- chunk_type 新增 episode、fact；session_summary / insight 在迁移后不再写入
```

要点：

- `stats` 中的会话计数和平均分由 `memory_episodes`、`memory_scores` 聚合得到，不再存累加器，避免计数漂移。
- 向量表变成纯派生索引：正文保存在 `memory_episodes` 和 `memory_facts` 中，重建时可以直接删除再生成，#96 中“保留正文、标记失效”的复杂度随之消失。

## 6. 写入流程

### 6.1 状态机

```
afterReview(session)
  ├─ episode.state = applied        → 直接返回（幂等，不限条数）
  ├─ episode 不存在                 → LLM 提取 + 校验 → INSERT episode(state=extracted)   [事务 1]
  ├─ 计算 Embedding（新观察、候选事实、情景摘要）                                         [事务外 I/O]
  ├─ 对齐（6.2）
  ├─ BEGIN IMMEDIATE                                                                       [事务 2]
  │    复核 episode.state = extracted，复核候选事实的 version
  │    写 memory_events → 更新 memory_facts 投影 → 掌握度、得分 → memory_vectors
  │    episode.state = applied
  │  COMMIT
  └─ 入队 memory_consolidation 后台任务（24 小时冷却）
```

- 任一步失败或进程崩溃：事务 2 要么完整生效、要么完全不生效。重试时 `state=extracted` 会跳过 LLM 提取，从 Embedding 开始重做。
- `BEGIN IMMEDIATE` 依靠 SQLite 文件锁，多进程也安全；version 复核失败时重新对齐一次。
- 现有的 `profile_extract_failed` 与“重试同步”入口保持不变，只是内部改为驱动这个状态机。

### 6.2 对齐（reconcile）

对每条新观察（薄弱点、强项、逐题得分），在同 kind、同 axis、同 topic（或无 topic）的事实中召回余弦最高的 5 条，包含已归档事实，以便取消归档：

| 条件 | 决策 |
| --- | --- |
| `norm_key` 相同 | 同一事实 |
| 相似度 ≥ τ_high（初值 0.88） | 同一事实 |
| 相似度 < τ_low（初值 0.62） | 新事实 |
| 介于两者之间 | 汇总到本场的一次批量 LLM 仲裁 |

- 仲裁只输出 `{"index": n, "decision": "same", "fact_id": "..."}` 或 `{"index": n, "decision": "new"}`，`fact_id` 必须属于候选集，沿用 `structured-output.ts` 的校验。
- 阈值与 Embedding 模型相关，按默认模型在评测集上校准，并记录到配置中。
- Embedding 不可用时，仍按精确匹配写入，但把新事实标记为 `attrs.needs_reconcile`；之后由后台任务补做对齐，重复事实以 `merged` 事件合并到目标事实（`merged_into`），历史事件保留。

## 7. 读取：MemoryContextBuilder

```ts
buildMemoryContext({
  userId, purpose,              // drill | resume | jd | recording | copilot | agent
  topic?, query?, budgetChars,  // 默认 3000
}): Promise<{ text: string; items: Array<{ factId?: string; sessionId?: string; section: string }> }>
```

各用途的查询来源：

| 用途 | query |
| --- | --- |
| drill | 主题名 + 本轮知识点 + 高频考点（沿用现有 memoryQuery） |
| resume | 简历要点 |
| jd | JD 正文摘要 |
| recording | 转写前段 |
| copilot | 公司 + 岗位 + JD |
| agent | 当前用户消息 |

分段与优先级（未用完的预算顺延给下一段）：

1. 到期复习：`next_review <= today`，按调度难度排序。
2. 与 query 相关的薄弱点：向量相似度 × 显著性（沿用 `salience`）。
3. 跨会话规律：显示支撑事实数；已入选的支撑事实折叠到规律下面，节省预算。
4. 行为短板。
5. 历史会话：情景向量 × 时间衰减（沿用半衰期 14 天、下限 0.7），**按事实当前状态渲染**：
   `[2026-10-02 · Redis] 摘要…… 涉及：缓存击穿（已于 10-20 改善）、AOF 重写（仍薄弱）`
6. 用户自述：未过期的 stated 事实，标注“用户自述”。
7. 强项。

所有段落按 `fact_id` 去重。`ProfileService.summary()` 先保留原签名作为适配层，内部改用构建器；各调用方随后逐个改为传入 `purpose` 与 `query`，解决 P4。

## 8. 时间语义

- 两个时间：`occurred_at`（证据发生时间，导入的录音和归档以原始时间为准）与 `recorded_at`（写入时间）。
- 任意时刻 t 的事实状态 = 截至 t 的事件重放结果。领域回顾可以据此生成准确的“某天之前 / 之后”对比。
- 事实不删除只失效：改善、归档、合并、过期都是事件。检索层只读投影，审计和回放读事件。
- stated 事实可带 `valid_to`（例如“下周三面字节”）；过期后自动写 `expired` 事件，不再注入上下文。

## 9. 规律归纳 v2

- 从 `afterReview` 的请求路径移到后台任务 `memory_consolidation`，触发条件不变（≥ 5 个活跃观察型薄弱点、≥ 2 个领域、24 小时冷却）。
- 规律通过 `memory_fact_links` 关联支撑事实，**支撑事实保持 active，继续参与复习调度**，修复 P2；只在展示时折叠。
- 所有支撑事实都改善后，规律自动写 `improved` 事件。
- 用户反馈写 `confirmed` / `refuted` 事件，置信度规则沿用现有实现。

## 10. 复习调度

- 抽出 `ReviewScheduler` 接口：`next(state, score, at) → { schedule, nextReview }`。
- 默认实现 `Sm2Scheduler`，与现有 `sm2Update` 的输出完全一致，由回归测试保证。
- FSRS（`ts-fsrs`）作为可选实现，需先在评测中证明收益再切换；10 分制得分到 FSRS 评级的映射需要单独校准。

## 11. 个人 Agent 回写

- 对话空闲后或每 N 轮，入队后台任务，从最近消息中抽取 stated 事实：目标公司 / 岗位 / 面试时间、准备约束、回答偏好、自我评估。
- 召回已有 stated 事实（带 ID），由 LLM 判定 `ADD / UPDATE(id) / DELETE(id) / NOOP`；DELETE 写 `archived` 事件，不做物理删除。
- 自我评估不改变测得的薄弱点和掌握度，只作为“用户自述”注入。
- 画像页提供 stated 事实的查看和删除入口；归档导出包含这些事实。

## 12. 兼容与迁移

- **公开 API 不变**：`ProfileUseCases.get` 从表中组装出原有的 `CandidateProfile` 结构（`history` 由事件生成），契约和前端在第一阶段不改。
- **惰性迁移**：用户首次访问时，若 profile.json 含派生字段且尚无 `memory_facts` 记录，则在一个事务中完成：
  - weak / strong / behavior → facts；嵌套 `history` → events（无会话来源时 `session_id` 为空）。
  - `session_extractions` → episodes（state = applied）；`_pending_memory` → episodes（state = extracted），由重试流程补齐。
  - 旧的 session_summary / insight 向量关联到对应 episode。
  - 原文件另存为 `profile.v1.json`，写迁移标记，保证可重入。
- **归档导入导出**：导出新表；导入时情景按 `(user_id, session_id)` 去重，事实按 `fact_id` 去重；来自不同设备的同义事实再走一次对齐并合并。旧格式归档先经过迁移器，再合并。
- **功能开关**：`MEMORY_V2` 在灰度期间默认关闭；开启前跑迁移往返测试（旧画像 → 新表 → 组装出的旧结构，与原文件逐字段比较）。

## 13. 评测与验收

### 13.1 确定性测试（`tests-ts/`，不需要 API Key）

沿用 `profile-memory-retry.test.ts` 的做法：临时目录、真实 SQLite、可控的 AI / Embedding 替身。

- 幂等：同一会话重复调用，事实、事件、计数不变。
- 故障注入：在提取后、Embedding 后、事务中途、提交后分别抛错或终止，重试后结果与一次成功的结果一致。
- 并发：两个进程同时处理同一用户的不同会话。
- 迁移往返、归档合并、用户隔离、P2 回归（归纳后支撑事实仍有 `next_review`）。

### 13.2 质量评测（`scripts/memory-eval/`，需要模型）

数据集：合成的多会话用户轨迹（例如 20 个用户 × 10 场），标注“哪些观察是同一事实”和“每个事实的状态时间线”，另标注一组检索查询的相关会话。

| 指标 | 定义 |
| --- | --- |
| 对齐 F1 | 成对“同一事实”判定的精确率 / 召回率 |
| 重复率 | (事实数 − 真实簇数) / 真实簇数 |
| 过时率 | 注入上下文中与当前状态矛盾的条目占比 |
| 复习覆盖率 | 归纳后仍有复习调度的具体薄弱点占比 |
| 检索 Recall@3 / nDCG@3 | 标注查询上的情景检索质量 |
| 成本 | 每场会话的 LLM 调用次数与 token（读 `llm_usage`），`afterReview` P50 / P95 耗时 |

**在动代码前先在 main 上跑出基线**，每个阶段结束后重跑，结果记录在本节表格中。

| 指标 | 基线（main） | v2 |
| --- | --- | --- |
| 对齐 F1 | 待测 | 待测 |
| 重复率 | 待测 | 待测 |
| 过时率 | 待测 | 待测 |
| 复习覆盖率 | 待测 | 待测 |
| Recall@3 | 待测 | 待测 |
| 每场 LLM 调用 / token | 待测 | 待测 |

## 14. 分期交付

每个 PR 可独立合并：

| PR | 内容 | 数据模型变更 |
| --- | --- | --- |
| A | P0 修复：归纳不再中断复习；停止写入无人读取的 `weak_point` 向量；所有模式传入检索 query；检索结果按当前状态标注 | 无 |
| B | 评测脚本与基线数据 | 无 |
| C | 新表、仓库、惰性迁移器、旧结构适配层、写入状态机（`MEMORY_V2` 开关） | 有 |
| D | 对齐：模糊区间 LLM 仲裁、`needs_reconcile` 后台合并 | 无 |
| E | MemoryContextBuilder 与各模式查询；开关默认开启；移除 `_pending_memory` 路径 | 无 |
| F | 个人 Agent 回写与画像页 stated 事实管理 | 无 |
| G | 归档导入导出 v2 | 归档格式 |

## 15. 风险与边界

- **迁移风险**：迁移错误会损坏画像。缓解：保留 `profile.v1.json`、惰性可重入、往返比较测试、开关灰度。
- **成本**：LLM 仲裁只覆盖模糊区间，每场最多一次批量调用；以评测中的成本指标为准，超出预算就收窄区间。
- **阈值依赖模型**：更换 Embedding 模型后需要重新校准；未校准时自动放宽到“更多交给仲裁”的区间。
- **与官方版本分叉**：官方版本在独立私有仓库。保持 `ProfileUseCases`、`CandidateProfilePort` 等端口不变，改动集中在 `packages/core/src/profile` 与 `packages/db`，降低移植成本。
- **不覆盖**：多设备实时同步；跨用户共享记忆；大规模（单用户十万级条目）向量检索。
