# TechSpar Memory 子系统优化路线计划书

- 分析基线：`b3ccb74`（fork `stanGSA/TechSpar` 的 main）。上游 `AnnaSuSu/TechSpar` 的 main 目前在 `a309871`，比 fork 多 2 个提交（#93、#94），其中 #93 改动了 `profile-service.ts` 的统计部分，不影响本文结论。
- 分析日期：2026-11-10
- 范围：长期画像（`profile.json`）、向量记忆（`memory_vectors`）、会话提取缓存与待处理快照、个人 Agent 的对话记忆，以及它们的检索、注入、重建、迁移和删除路径。
- 上游现状：没有打开的 memory 相关 Issue 或 PR；唯一的专项改动是 #77（记忆投影失败重试），对应 `docs/memory-reliability-design.md`。

## 1. 结论摘要

1. **最严重的问题是数据丢失。** 用户在设置里切换 Embedding 模型，或点击“重建索引”，或导入个人归档，都会把该用户的全部会话语义记忆删掉，而且没有任何路径能恢复。这一条已经用真实 SQLite 跑脚本验证过，应该作为第一个 PR。
2. **写入侧的可靠性在 #77 之后基本成型，但留了四个缺口：** Embedding 故障时画像以降级匹配落盘且不可重做；待处理快照只能手动重试；幂等窗口只有 100 个会话；删除会话不清理它的记忆。
3. **读取侧只做了一个场景。** 语义记忆只在“专项训练出题”时被检索；简历面试、JD 备面、录音复盘、Copilot 和个人 Agent 都没有用到。个人 Agent 拿到的“长期画像”是对整份画像 JSON 的盲截断。
4. **有一类向量只写不读。** `weak_point` 类型的向量由三条路径写入，格式互不一致，核心代码里没有任何读取方；与此同时每次复盘会把画像里全部强弱点重新嵌入一遍。
5. **建议的推进顺序：** 先止血（数据丢失），再补一致性，然后建评测基线，之后才调检索质量和成本，最后做存储演进与用户可控。全部拆成 21 个可独立验证的小 PR，见第 5 节。

问题按严重度分级：P0 为数据丢失或静默失效，P1 为一致性与正确性，P2 为检索质量与成本，P3 为可维护性、可观测性与用户控制。

## 2. 分析方法与证据等级

读完了下列文件的全部代码，并对最重要的结论写了复现脚本，用仓库里的真实实现（`ProfileService`、`BunKnowledgeVectorRepository`、`FileCandidateProfileRepository`、`KnowledgeIndexService`）配合可控的 AI 与 Embedding 替身执行。

每个问题标注一种证据等级：

| 标记 | 含义 |
| --- | --- |
| 已执行验证 | 写脚本跑过，下文给出观测数据 |
| 代码走读 | 调用链读通，逻辑明确，但没有单独跑脚本 |
| 待对齐 | 属于产品语义或设计取舍，需要先和维护者确认 |

复现脚本的观测结果（40 个专项训练会话，单用户）：

| 场景 | 观测 |
| --- | --- |
| 40 次复盘后的记忆行数 | `session_summary` 40、`insight` 40、`weak_point` 40 |
| 调用 `invalidateUser`（切换 Embedding 与重建索引都会调用）之后 | 0 行 |
| 再执行重建索引的“记忆 / 薄弱点”步骤之后 | 只剩 `weak_point`；`semanticHistory` 命中 0 条；`summary()` 不再包含“历史语义洞察” |
| Embedding 维度变化但未重建 | `semanticHistory` 命中 0 条，无报错 |
| 删除一个会话之后 | 该会话的 3 行记忆仍在 |
| `GET /api/profile` 的 JSON 长度 | 25,380 字符，其中 `session_extractions` 10,271、`weak_points` 10,328；个人 Agent 只取前 14,000，`stats`、`session_extractions`、`due_reviews` 被截断 |
| Embedding 在复盘期间故障，随后手动重试成功 | “GIL 机制理解不清”和“全局解释器锁原理说不清”成为两条独立薄弱点，各出现 1 次 |

复现用的替身 Embedding 是低维哈希向量，所以上表的行数和字符数是结构性结论，不代表真实模型下的相似度表现。

## 3. 现状架构

### 3.1 组成

| 组件 | 存储 | 位置 | 说明 |
| --- | --- | --- | --- |
| 结构化画像 | `data/users/<id>/profile/profile.json` | `packages/platform/src/profile-repository.ts` | 薄弱点、强项、行为信号、掌握度、统计、SM-2 状态。整文件原子重写，进程内按用户串行 |
| 会话提取缓存 | `profile.json` 的 `session_extractions` | `profile-service.ts:186-190` | 幂等判据，最多保留最近 100 个已完成会话 |
| 待处理记忆快照 | `profile.json` 的 `_pending_memory` | `profile-service.ts:464-491` | #77 引入的 outbox，向量写成功后清除 |
| 向量记忆 | SQLite `memory_vectors` | `packages/db/src/knowledge-vector-repository.ts` | 与知识库分块、个人文档分块共用一张表，用 `chunk_type` 区分 |
| 个人 Agent 对话 | SQLite `personal_conversations` | `packages/db/src/personal-agent-repository.ts` | 每个对话一行，消息是一个 JSON 数组 |

### 3.2 `memory_vectors` 里各 `chunk_type` 的读写方

| chunk_type | 写入方 | 读取方 |
| --- | --- | --- |
| `session_summary` | `projectSessionMemory` | `semanticHistory` |
| `insight` | `projectSessionMemory`（内容已包含 summary） | `semanticHistory` |
| `weak_point` | ① `projectSessionMemory`，内容是 `point`；② `consolidate` 经 `appendMemories`，无 `session_id`；③ 重建索引，内容是 `topic: point`，用 `replaceChunks` 整体替换 | 无 |
| `topic_chunk` | 知识库索引 | 知识库检索 |
| `personal_document_chunk` | 个人资料库（`session_id` 列存的是 `document_id`） | 个人 Agent |

### 3.3 数据流

```text
复盘任务 ──> ProfileService.afterReview
              ├─ LLM 提取（校验后才使用）
              ├─ 嵌入：全部既有强弱点 + 本次强弱点 + 逐题观察      ← 失败被吞，退化为精确匹配
              ├─ repository.update（文件锁内）：
              │     薄弱点合并 / SM-2 / 掌握度 / 统计
              │     session_extractions[sid] 与 _pending_memory[sid]
              ├─ projectSessionMemory：嵌入 → replaceSessionMemories → 清除 pending
              └─ consolidate：跨领域规律（尽力而为，失败被吞）

专项训练出题 ──> profile.summary(userId, topic, memoryQuery)
                  ├─ 画像摘要（薄弱点、规律、强项、行为、掌握度、到期复习）
                  └─ semanticHistory：加载该用户全部记忆行 → JS 余弦 + 时间衰减 → 取 3 条

其余入口 ──> profile.summary(userId)          无语义检索
个人 Agent ──> JSON.stringify(profile).slice(0, 14000) + 最近错题 + 最近对话片段 + 文档检索
```

## 4. 问题清单

### 4.1 P0：数据丢失与静默失效

#### M-01 切换 Embedding 模型或重建索引会永久清空会话语义记忆（已执行验证）

- 调用链：`settings-service.ts:72`（Embedding 目标变化即调用）和 `settings-operations.ts:103`（重建索引的 cleanup 步骤）→ `index-service.ts:70-72` 的 `invalidateUser` → `knowledge-vector-repository.ts:155-157` 的 `DELETE FROM memory_vectors WHERE user_id = ?`，不区分类型。
- 重建只恢复三样：从 `profile.json` 生成的 `weak_point`、个人文档、知识库分块（`settings-operations.ts:104-111`）。`session_summary` 和 `insight` 的正文只存在于被删掉的行里，无处可恢复。
- 影响：用户只是换了一个 Embedding 模型，之后所有专项训练都拿不到“历史语义洞察”，界面没有任何提示。
- 修复方向：`invalidateUser` 不再删除画像记忆类型；重建索引增加“会话记忆”步骤，读出现有行的正文，按会话重新嵌入并用 `replaceSessionMemories` 原位替换。不需要改表。

#### M-02 向量行不记录模型与维度，过期向量被静默忽略或错误使用（已执行验证）

- `profile-service.ts:194-199` 的 `cosine` 在维度不一致时返回 -1，随后被 `score > 0` 过滤，结果为空且无报错。
- 同维度不同模型（例如两个 1024 维模型）时得分没有意义，但会照常排序返回。
- `EmbeddingService.signature`（`embedding-service.ts:48`）已经存在，但记忆路径没有使用它。
- 修复方向：给 `memory_vectors` 增加可空的 `embedding_model` 列，沿用 `knowledge-vector-repository.ts:56-70` 里已有的 `PRAGMA table_info` 增量迁移写法；读取时只取与当前签名一致的行；对外暴露“有 N 条记忆待重新嵌入”。

#### M-03 个人归档不携带会话记忆，导入后还会清空现有记忆（代码走读）

- 导出白名单 `data-migration-repository.ts:7-13` 不含 `memory_vectors`。
- 导入结束时 `data-migration-service.ts` 调用 `invalidateDerivedData`，在 `data-migration-repository.ts:128` 删除该用户全部向量，并返回 `requires_reindex: true`。
- 结合 M-01，重建索引无法恢复会话记忆，所以“导出再导入”等于丢失全部语义记忆。
- 修复方向：归档里携带记忆正文（不带向量，向量与模型绑定），导入后标记为待嵌入。涉及归档 schema 版本，需要先开 Issue。

#### M-04 切换 Embedding 后个人文档向量被删，但文档状态仍是 `ready`（代码走读）

- 个人 Agent 仓库与向量仓库用的是同一个数据库文件（`entry.bun.ts:94` 与 `:122`），所以 M-01 的删除同样清掉 `personal_document_chunk`。
- 导入路径会把文档标成 `needs_reindex`（`data-migration-repository.ts:130`），设置路径不会。结果是文档列表显示正常，Agent 却检索不到任何资料。
- 修复方向：与 M-01 同一个 PR，Embedding 变化时把该用户的文档标成 `needs_reindex`。

### 4.2 P1：一致性与正确性

#### M-05 Embedding 故障时，画像以降级匹配落盘且重试无法纠正（已执行验证）

- `profile-service.ts:405-412` 的 `vectorsFor` 吞掉嵌入错误并返回空表。随后的合并只剩精确文本匹配，语义相同但措辞不同的薄弱点成为重复条目，SM-2 评分也退回到整场平均分。
- 这次降级结果会连同 `session_extractions` 一起落盘。接着 `projectSessionMemory` 抛错，会话被标成失败；用户点“重试同步”时命中提取缓存，只补写向量，画像不再重算。
- 修复方向：复盘路径上的匹配嵌入改为必须成功，在 `repository.update` 之前失败。此时什么都没落盘，现有重试机制可以完整重做。

#### M-06 删除会话不清理它的向量记忆（已执行验证）

- `interview-repositories.ts:282-290` 只删 `sessions`、`resume_interview_state` 和 `tasks`。
- 用户删掉一场面试后，这场面试的摘要仍会被检索并注入后续出题提示词，`session_extractions` 和 `_pending_memory` 里的条目也还在。
- 修复方向：同一事务里删除该会话的画像记忆行；同时清掉对应的待处理快照。已计入画像的统计和薄弱点是否回滚属于待对齐问题，建议保留聚合结果，只删原始记忆。

#### M-07 待处理快照只能手动重试，没有启动对账（代码走读）

- `docs/memory-reliability-design.md` 的“兼容性与边界”已经写明：进程在写入失败标记之前退出时，可能没有界面重试入口。
- 修复方向：服务启动时扫描各用户的 `_pending_memory`，通过现有任务队列排入 `profile_only` 任务。

#### M-08 幂等窗口只有 100 个会话（代码走读）

- `trimExtractions`（`profile-service.ts:186-190`）淘汰第 100 个之前的已完成会话。更早的会话如果被重放，统计、掌握度和 SM-2 会被重复计入。
- 缓存的是完整提取结果，40 个会话就占 10 KB，却只是为了判断“处理过没有”。
- 修复方向：单独保存一份只含会话 ID 的已处理账本，完整提取结果可以更早淘汰。

#### M-09 `weak_point` 向量三条写入路径互相冲突，且无人读取（代码走读）

- 三条写入路径见 3.2 节。重建索引的 `replaceChunks` 会删掉会话级的 `weak_point` 行；`consolidate` 写入的行没有 `session_id`，无法被替换，写入失败也被吞掉（`profile-service.ts:461`）。
- `semanticHistory` 只请求 `session_summary` 和 `insight`（`profile-service.ts:416`），核心代码里没有其他读取方。
- 修复方向：二选一。要么删除这些写入，省掉嵌入费用；要么把它们变成 M-14 的嵌入缓存，统一格式和归属。

#### M-10 `profile.json` 承担了过多职责（已执行验证，方案待对齐）

- 一份文件里同时放着结构化画像、提取缓存、outbox 和不设上限的历史数组（薄弱点的 `history`、`stats.score_history`、强项列表）。每次更新整文件重写，并发保护只有进程内锁（`profile-repository.ts:41-50`）。
- `GET /api/profile` 把 `session_extractions` 原样返回给浏览器；Copilot 备面把整份画像快照存进 prep 结果（`prep-service.ts:198`、`:224`）。
- 修复方向：把 outbox、已处理账本和提取缓存迁到 SQLite 表，给历史数组设上限。这是架构调整，并且要保持便携归档兼容，需要先开 Issue 讨论。

### 4.3 P2：检索质量与成本

#### M-11 语义记忆只在专项训练出题时使用（代码走读）

- 唯一带语义检索的调用是 `interview-service.ts:306`。
- 简历面试（`resume-state-machine.ts:70`）、录音复盘（`recording-service.ts:130`、`:140`）、Copilot 备面（`prep-service.ts:197`）都调用不带主题的 `summary(userId)`，不触发 `semanticHistory`。
- 修复方向：给这些入口各自构造检索意图（岗位、JD 要点、转写主题），复用同一个检索函数。

#### M-12 个人 Agent 的画像上下文是对 JSON 的盲截断（已执行验证）

- `personal-agent-service.ts:81` 用 `JSON.stringify(profile).slice(0, 14000)`。
- 画像字段顺序是掌握度、薄弱点、强项、行为信号、统计、提取缓存。带长篇回顾的掌握度和带完整历史的薄弱点会先占满预算，后面的字段被截掉，尾部是不合法的 JSON。
- 架构文档把个人 Agent 描述为拥有“长期记忆”，但它没有检索任何会话记忆，尽管查询向量在同一个函数里已经算好了（`:74`）。
- 修复方向：用按用途构造、分段限额的画像摘要替代原始 JSON，并用已有的查询向量检索会话记忆。

#### M-13 个人 Agent 的跨对话记忆只按时间取片段（代码走读）

- `personal-agent-repository.ts:112-122` 取最近 8 个对话、每个对话最后 6 条、总共 12 条，每条截到 1200 字符，不看相关性。
- 当前对话只带最后 12 条消息（`personal-agent-service.ts:84`），更早的内容没有滚动摘要。
- 修复方向：对话结束或达到长度阈值时生成摘要并嵌入，检索时按相关性取。

#### M-14 每次复盘把画像里全部强弱点重新嵌入（代码走读）

- `profile-service.ts:688-692` 把既有的全部薄弱点和强项文本送去嵌入，费用随画像规模线性增长，而这些文本绝大多数在上一次复盘时已经嵌入过。
- 仓库里已有同类缓存的先例：`question_embeddings`。
- 修复方向：按“模型签名 + 文本哈希”缓存画像条目的向量。

#### M-15 相似度阈值和衰减参数硬编码，且与模型无关（代码走读）

- `profile-service.ts:58-66`：薄弱点合并 0.75、复习匹配 0.6、强项去重 0.8、半衰期 14 天、衰减权重 0.3。
- 不同 Embedding 模型的余弦分布差别很大，同一组阈值在 bge-m3 和各家 API 模型上的合并行为不会一致。目前没有任何评测集能说明这些数值是否合适。
- 修复方向：先建评测基线（见 PR-9），再决定是按模型校准还是改成基于排名的匹配。在有数据之前不建议直接改阈值。

#### M-16 检索是全量加载加 JS 暴力扫描（代码走读）

- `listProfileMemories`（`knowledge-vector-repository.ts:135-153`）的 SQL 取回三种类型的全部行和向量，再在 JS 里丢掉不需要的类型。每次专项训练开始都要解码该用户的全部向量。
- 表上只有单列索引，没有 `(user_id, chunk_type)` 复合索引。
- 修复方向：把类型过滤下推到 SQL，加复合索引。是否引入向量索引扩展，等评测和性能数据出来再定。

#### M-17 会话记忆只增不减，且每个会话存了两份重叠内容（代码走读）

- `insight` 的正文以 `session_summary` 开头（`profile-service.ts:174-184`），读取时再靠去重逻辑二选一。
- 没有压缩、合并或保留期限。
- 修复方向：每个会话只存一行；长期再考虑按主题滚动汇总。

#### M-18 到期复习按 UTC 日期计算，时钟不可注入（代码走读）

- `profile-service.ts:533`、`:585` 用 `new Date().toISOString().slice(0, 10)` 作为“今天”。UTC+8 的用户在每天 0 点到 8 点之间看不到当天到期的复习项。
- `salience`（`:256-260`）和整合冷却直接读 `Date.now()`，测试无法固定时间。
- 修复方向：引入时钟端口，按用户所在时区计算本地日期。

### 4.4 P3：可维护性、可观测性与用户控制

#### M-19 记忆路径上的错误被静默吞掉（代码走读）

- `semanticHistory`（`:450`）、`vectorsFor`（`:411`）、`appendMemories`（`:461`）、`consolidate`（`:631`）都是空 `catch`。
- 用户和维护者都无法区分“没有记忆”和“记忆坏了”。
- 修复方向：增加日志端口和记忆健康状态（行数、待处理数、过期数、最近一次错误）。

#### M-20 用户看不到也改不了自己的记忆（代码走读）

- 画像路由（`apps/api/src/routes/profile.ts`）只有对整合规律的反馈接口。用户不能查看被注入提示词的历史洞察，也不能删除或纠正一条错误的薄弱点。
- 修复方向：记忆查看、纠正和删除的接口与页面。

#### M-21 代码结构与重复定义（代码走读）

- `profile-service.ts` 有 759 行，混合了提示词、校验、SM-2、薄弱点合并、统计、检索、整合和投影。
- `cosine` 有三份实现，维度不一致时的返回值不同：`profile-service.ts:194` 返回 -1，`index-service.ts:37` 和 `personal-agent-repository.ts:19` 返回 0。`toBlob` 和 `fromBlob` 有两份。
- `memory_vectors` 的建表语句有三处：`knowledge-vector-repository.ts:32`、`personal-agent-repository.ts:47`、`schema.ts:79`，索引定义不一致。
- 修复方向：纯重构，先有特征测试再搬代码，不改行为。

#### M-22 整合后具体薄弱点退出复习队列（待对齐）

- `consolidate` 把支撑规律的具体薄弱点归档；`dueReviews`（`:584-587`）同时排除已归档项和整合项。结果是这些知识点不再进入 SM-2 复习，直到下次被再次观察到。
- 现有测试覆盖了归档行为，说明这是从旧版移植过来的设计。是否符合产品预期需要维护者确认。

#### M-23 遗留字段（代码走读）

- `communication` 和 `thinking_patterns` 在模型和响应契约里保留，但当前后端没有任何写入，它们的职责已经被 `behavior_signals` 取代。建议在文档里标明为只读兼容字段。

## 5. 路线图

规模标记：S 为一两个文件加测试，M 为跨包改动，L 为涉及存储结构或前后端联动。“先开 Issue”依据 `CONTRIBUTING.md`：新功能、架构调整或破坏兼容性的改动应先对齐范围。

### 阶段 0：准备

1. 把 fork 同步到上游 `a309871`，之后的分支都从最新 main 拉出。
2. 在上游开一个总览 Issue，贴出本计划的问题清单和阶段划分，请维护者确认范围和优先级，特别是 M-10、M-22 这类需要决策的条目。
3. 第一个 PR 直接提 PR-1。它是纯修复，不需要等总览 Issue 讨论完。

### 阶段 1：止血

| PR | 标题 | 对应问题 | 规模 | 先开 Issue |
| --- | --- | --- | --- | --- |
| PR-1 | `fix(memory): preserve session memories across embedding changes and index rebuilds` | M-01、M-04 | M | 否 |
| PR-2 | `fix(memory): tag vectors with their embedding model and skip stale rows` | M-02 | M | 否 |
| PR-3 | `fix(memory): delete session memories together with the session` | M-06 | S | 否 |
| PR-4 | `fix(migration): carry session memories through personal archives` | M-03 | M | 是 |

PR-1 的验收标准，沿用 #77 设计文档的表格写法：

| 阶段 | `session_summary` + `insight` 行数 | `semanticHistory` 命中 | 个人文档状态 |
| --- | --- | --- | --- |
| 切换 Embedding 之前 | 2N | 大于 0 | `ready` |
| 切换之后、重建之前 | 2N（保留，标记过期） | 0，且状态可见 | `needs_reindex` |
| 重建之后 | 2N（新维度） | 大于 0 | `ready` |

测试用临时目录、真实 SQLite 和可切换维度的 Embedding 替身，另加一条通过 HTTP 路由触发重建 SSE 的用例，并更新 `tests-ts/contracts/fixtures/sse/rebuild-index-stream.txt`。

### 阶段 2：一致性与可恢复性

| PR | 标题 | 对应问题 | 规模 | 先开 Issue |
| --- | --- | --- | --- | --- |
| PR-5 | `fix(profile): fail before persisting when matching embeddings are unavailable` | M-05 | S | 否 |
| PR-6 | `feat(memory): reconcile pending memory projections on startup` | M-07 | M | 否 |
| PR-7 | `fix(profile): keep a durable ledger of processed sessions` | M-08 | S | 否 |
| PR-8 | `refactor(memory): give weak-point vectors a single owner` | M-09 | S | 是，需先定去留 |

PR-5 的核心断言：Embedding 故障期间复盘失败，画像文件逐字节不变；恢复后重试，同义薄弱点合并为一条，`times_seen` 为 2。

### 阶段 3：检索质量与成本

| PR | 标题 | 对应问题 | 规模 | 先开 Issue |
| --- | --- | --- | --- | --- |
| PR-9 | `test(memory): add a retrieval and matching evaluation harness` | M-15 | M | 否 |
| PR-10 | `perf(profile): cache point embeddings by model and text hash` | M-14 | M | 否 |
| PR-11 | `perf(memory): filter chunk types in SQL and add a composite index` | M-16 | S | 否 |
| PR-12 | `feat(agent): build agent context from a budgeted profile digest and semantic memory` | M-12 | M | 是 |
| PR-13 | `feat(memory): semantic recall for resume, JD and recording modes` | M-11 | M | 是 |
| PR-14 | `feat(agent): summarize and retrieve cross-conversation memory` | M-13 | L | 是 |
| PR-15 | `fix(profile): compute due reviews on the user's local date` | M-18 | S | 否 |

PR-9 必须排在所有调参类改动之前。它提供一组带标注的中文面试语料（同义薄弱点对、不相关对、查询与应命中的记忆），默认用确定性替身跑在 CI 里，另提供一个可选脚本对真实模型输出合并准确率和召回率。之后任何改阈值、改衰减、改检索范围的 PR 都要附上这组数字的前后对比。

### 阶段 4：存储与结构演进

| PR | 标题 | 对应问题 | 规模 | 先开 Issue |
| --- | --- | --- | --- | --- |
| PR-16 | `refactor(memory): share vector codec and similarity, unify memory_vectors DDL` | M-21 | S | 否 |
| PR-17 | `refactor(profile): split ProfileService by responsibility` | M-21 | M | 否 |
| PR-18 | `refactor(profile): move outbox, ledger and extraction cache out of profile.json` | M-10 | L | 是 |
| PR-19 | `feat(memory): compact and retain session memories` | M-17 | L | 是 |

PR-17 只搬代码，现有 `tests-ts/profile.test.ts` 和 `tests-ts/profile-memory-retry.test.ts` 不改一行即全部通过，作为审查依据。PR-18 要保证旧画像文件和旧归档无需手工迁移。

### 阶段 5：可观测与用户可控

| PR | 标题 | 对应问题 | 规模 | 先开 Issue |
| --- | --- | --- | --- | --- |
| PR-20 | `feat(memory): expose memory health and log swallowed failures` | M-19 | M | 否 |
| PR-21 | `feat(profile): let users inspect, correct and delete memories` | M-20 | L | 是 |

M-22 和 M-23 不单独排 PR，在总览 Issue 里请维护者表态后再决定。

### 依赖关系

```text
PR-1 ──> PR-2 ──> PR-4
  └────> PR-3
PR-5、PR-6、PR-7 相互独立
PR-9 ──> PR-12、PR-13、PR-14、PR-19
PR-8 ──> PR-10
PR-16 ──> PR-17 ──> PR-18
PR-2 ──> PR-20 ──> PR-21
```

## 6. 提交策略与维护者路径

以下建议来自这个仓库已有的合并记录和 `CONTRIBUTING.md`，不是通用经验。

1. **你已经有基础。** 上游已合并你的 #74、#76、#78、#79、#84、#87、#93，其中 #76、#79、#93 直接触及画像。memory 目前没有固定的负责人，#77 的作者只提交过这一次。
2. **每个 PR 只做一件可独立验证的事。** 上游合并的 PR 基本都是单一目的，标题用 `类型(范围): 描述`。本计划的 21 个 PR 都按这个粒度拆分。
3. **沿用 #77 的证据写法。** PR 描述里写清问题、实现、故障演示表、兼容性与边界，并给出不需要 API Key 就能跑的测试命令。测试要走真实边界：真实 SQLite、临时画像文件、通过 Hono `app.request()` 的 HTTP 用例。
4. **提交前跑全量检查。** `bun run check`；改了请求、响应或路由就跑 `bun run gen:api` 并提交生成文件，CI 会校验差异。
5. **架构类改动先开 Issue。** 表中标了“是”的条目不要直接提 PR。
6. **把领域知识沉淀在仓库里。** 在 PR-17 之后提交一份 `docs/memory.md`，描述 3.1 到 3.3 节的现状架构并随改动更新。之后别人改 memory 时会先读它，也会自然地请你审查。
7. **承担提交之外的维护工作。** 主动审查其他人触及 `packages/core/src/profile`、`packages/db` 向量部分的 PR；回复 memory 相关的 Issue；在发布说明里补充相关条目。
8. **提交权限由上游所有者决定。** 这一点无法通过计划保证。能做到的是让自己成为这个领域里上下文最完整、改动最可靠的人；在阶段 1 和阶段 2 合并之后，再向维护者提出为 memory 相关路径设置 CODEOWNERS 会比较自然。

## 7. 风险与待对齐问题

| 问题 | 为什么需要维护者决定 |
| --- | --- |
| `weak_point` 向量是删除还是改造成缓存（M-09） | 影响 PR-8 和 PR-10 的方案 |
| 删除会话时是否回滚已计入画像的统计和薄弱点（M-06） | 涉及“删除”在产品上的含义 |
| 归档是否携带记忆正文，是否提升归档 schema 版本（M-03） | 影响旧版本导入兼容性 |
| outbox 和提取缓存是否迁出 `profile.json`（M-10） | 最大的一次存储结构变化 |
| 整合后具体薄弱点是否应继续参与复习（M-22） | 当前行为有测试覆盖，可能是有意设计 |
| 官方云端版本是否共享这部分代码 | `docs/editions.md` 说明官方版本来自独立私有仓库，修复需逐项移植。M-16 这类性能问题在多用户部署下更重要，值得确认优先级 |

其他风险：

- 本文的复现使用替身 Embedding，真实模型下的合并和召回表现需要 PR-9 的评测基线才能下结论。
- 没有在真实的多用户部署或大数据量下测量过性能，M-16 的优先级是按代码结构推断的。
- 没有审查官方私有仓库，也没有审查前端画像页面的交互细节。

## 附录：相关文件索引

| 文件 | 内容 |
| --- | --- |
| `packages/core/src/profile/profile-service.ts` | 提取、合并、SM-2、统计、检索、整合、记忆投影 |
| `packages/core/src/profile/model.ts`、`ports.ts`、`merge.ts` | 画像类型、端口、归档合并 |
| `packages/platform/src/profile-repository.ts` | 画像文件仓库与进程内锁 |
| `packages/db/src/knowledge-vector-repository.ts` | `memory_vectors` 读写、会话记忆替换 |
| `packages/db/src/personal-agent-repository.ts` | 个人文档向量、对话存储、跨对话片段 |
| `packages/core/src/personal-agent/personal-agent-service.ts` | 个人 Agent 上下文组装 |
| `packages/core/src/provider/settings-service.ts`、`settings-operations.ts` | Embedding 变更触发的清理、重建索引 |
| `packages/core/src/knowledge/index-service.ts` | `invalidateUser`、知识库检索 |
| `packages/core/src/data-migration/data-migration-service.ts`、`packages/db/src/data-migration-repository.ts` | 归档导入导出、派生数据清理 |
| `packages/core/src/interview/interview-service.ts`、`recording/recording-service.ts`、`copilot/prep-service.ts` | 记忆的消费方与复盘后的画像更新入口 |
| `tests-ts/profile.test.ts`、`tests-ts/profile-memory-retry.test.ts` | 现有画像与记忆测试 |
| `docs/memory-reliability-design.md` | #77 的设计说明与已知边界 |
