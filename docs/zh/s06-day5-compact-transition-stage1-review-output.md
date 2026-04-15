# Day 5：Compact、Transition 与阶段 1 复盘

> 对应学习清单：`docs/zh/claude-code-4-week-sprint-checklist.md` 里的 “Day 5 Compact、Transition 与阶段 1 复盘”。
>
> 这份笔记不是逐行翻译源码，而是把你指定的几个片段串成一条能读懂的主线：
>
> `tokens.ts` 先回答“上下文到底有多满”  
> `toolResultStorage.ts` 再回答“单条工具结果怎么别把窗口撑爆”  
> `microCompact.ts` 再回答“旧工具结果怎么退场”  
> `autoCompact.ts` 最后回答“整段历史什么时候必须整体压缩”

## 这一天真正要学会什么

如果把 `s01-s06` 看成一个单 agent 的成长过程，那么 Day 5 在补最后一块骨架：

1. agent 不只是会 `tool_use`，还得会控制上下文体积。
2. 压缩不是一种机制，而是三层机制叠起来。
3. `TransitionReason` 不负责压缩内容，它负责解释“为什么主循环还能继续往前走”。

最短结论可以先记成一句话：

> Day 5 的核心不是“删历史”，而是“让 agent 在有限窗口里继续工作，并且知道自己为什么还能继续”。

## 一张总图

```text
工具返回大结果
   |
   v
toolResultStorage.ts
  ├─ 单工具过大 -> 写磁盘 + 留预览
  └─ 单条 wire message 总量过大 -> 选最大结果替换成预览
   |
   v
microCompact.ts
  ├─ cache 还热 -> 用 cache_edits 删旧 tool_result
  └─ cache 已冷 -> 直接把旧 tool_result 内容清空，只留占位
   |
   v
tokens.ts
  └─ 持续估算“当前上下文到底用了多少 token”
   |
   v
autoCompact.ts
  ├─ 还没超阈值 -> 继续
  ├─ 先试 session memory compact
  └─ 不够 -> full compact，生成摘要继续
   |
   v
query loop 用 TransitionReason 解释：
“这轮为什么继续，而不是结束”
```

## 1. `tokens.ts`：所有压缩判断的地基

### 1.1 `getTokenCountFromUsage()` 在做什么

你给的片段里，`getTokenCountFromUsage()` 把一次 API 返回的上下文总量定义成：

- `input_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`
- `output_tokens`

也就是说，它算的不是“模型本轮说了多少”，而是“这一整次上下文调用总共多大”。

这件事很关键，因为 compact 关心的是窗口占用，不是账单里某一项。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/tokens.ts:46-52`

### 1.2 `tokenCountWithEstimation()` 为什么是关键入口

`tokenCountWithEstimation()` 是 Day 5 最值得盯住的函数之一。

它的思路是：

1. 从后往前找最近一条带 `usage` 的 assistant message。
2. 如果同一次模型响应被拆成了多段 assistant record，而且它们 `message.id` 相同，就继续往前找到第一段。
3. 用那条 `usage` 当“已知真实值”。
4. 对这条响应之后新增的消息，再做粗略估算。

这解决了一个非常工程化的问题：

> 并行工具调用时，一次模型响应可能被拆成多段 assistant message，并且中间穿插多个 `tool_result`。如果你只看最后一段 assistant，就会漏算前面插进去的那些 `tool_result`。

所以它本质上是在保证：

> auto-compact 的触发判断，基于“下一次真正会送给模型的上下文大小”，而不是基于一个看起来差不多但其实偏小的假数。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/tokens.ts:226-261`

## 2. `toolResultStorage.ts`：先挡住“单条结果太大”

这一层解决的不是“整段历史太长”，而是更早的问题：

> 某一次工具调用的结果，或者某一条 user/tool_result 消息，已经大到不该直接塞给模型。

可以把它理解成 compact 之前的“局部减压阀”。

### 2.1 `persistToolResult()`：大结果先落盘

`persistToolResult()` 的动作很直接：

1. 只接受纯文本结果。
2. 给这次 `tool_use_id` 找到一个稳定文件路径。
3. 用 `flag: 'wx'` 写文件。
4. 如果文件已经存在，就不重复写。
5. 返回预览信息，而不是返回整份正文。

这里最重要的工程点有两个：

- `tool_use_id` 被当成稳定主键，所以同一个结果不会每轮都重写。
- `wx` 避开了 “先判断文件不存在，再写入” 的竞争窗口。

这就是教学版 `agents/s06_context_compact.py` 里 `persist_large_output()` 的生产版思路。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:137-183`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:189-198`

### 2.2 为什么还要有 “per-message budget”

只做“单工具过大时落盘”还不够。

因为还有一种情况：

> 每个 `tool_result` 自己都不算特别大，但几个结果并排塞进同一条 wire-level user message 之后，总和超预算了。

于是这份文件又做了第二层控制：

- `getPerMessageBudgetLimit()` 取单条消息的总预算。
- `provisionContentReplacementState()` 决定本线程要不要启用替换状态。
- `ContentReplacementRecord` 把“当时替换成了什么字符串”写进 transcript，保证恢复时字节级一致。

这个状态设计非常重要，因为它不是只关心“现在要不要替换”，它还关心：

> 一旦模型已经看过某个版本的内容，之后就不能随便改写同一段前缀，否则 prompt cache 会失稳。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:421-498`

### 2.3 `fresh / frozen / mustReapply` 是整份文件最核心的心智模型

在 `enforceToolResultBudget()` 里，候选结果被分成三类：

- `fresh`
  从没被模型见过，这次可以做新决策。
- `frozen`
  模型已经见过原文，而且当时没替换；以后就不能再替换了。
- `mustReapply`
  之前已经替换过；这次必须把同一份 replacement 原样重放。

这三类状态回答的是一个比“省 token”更高级的问题：

> 怎样在压缩上下文的同时，不把模型已经缓存过的前缀搞乱。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:641-667`

### 2.4 `collectCandidatesByMessage()` 为什么很难但很重要

这段代码最容易被低估。

它不是简单按本地 `messages[]` 一条一条算，而是尽量模拟 API 真正看到的分组方式：

- 连续的 user message 可能在发给 API 前被合并。
- progress / attachment / system(local_command) 并不一定构成新的 wire boundary。
- 同一个 assistant `message.id` 的碎片，即使中间夹了别的东西，也仍然可能属于同一次响应。

所以这段逻辑本质上在做一件事：

> 预算检查必须对齐“模型真正会收到的消息边界”，不能对齐“本地状态数组看起来的边界”。

如果不这样做，就会出现本地看起来每条都没超，但发给 API 时合并成一条以后突然爆掉。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:575-639`

### 2.5 `enforceToolResultBudget()` 的动作顺序

这段函数可以读成 5 步：

1. 找出每个 API-level message group 里的候选 `tool_result`。
2. 对每组候选做 `mustReapply / frozen / fresh` 分桶。
3. 如果总量超预算，就从 `fresh` 里挑最大的结果替换。
4. 被选中的结果并发落盘，生成 replacement message。
5. 用 `replacementMap` 回写到消息数组。

它有一个很重要的保守原则：

> 已经被模型看过但当时没替换的内容，宁可继续超一点，也不在后续 turn 里临时改写。

这就是注释里说的 prompt cache stability。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:739-909`

### 2.6 `applyToolResultBudget()` 与恢复逻辑

`applyToolResultBudget()` 是 query loop 真正接入的入口；它调用 `enforceToolResultBudget()`，并把新的 replacement records 写进 transcript。

`reconstructContentReplacementState()` 则负责会话恢复时把这些决策重建回来。

所以这一层不是“一次性压缩”，而是“带记忆的压缩决策”。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:924-936`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:960-1001`

## 3. `microCompact.ts`：旧结果别一直霸占活跃窗口

如果说 `toolResultStorage.ts` 是在处理“新结果太大”，那 `microCompact.ts` 处理的就是：

> 老结果虽然当时有价值，但现在不值得继续原样留在活跃窗口里。

### 3.1 `estimateMessageTokens()` 与 `calculateToolResultTokens()`

这两个函数提供 micro-compact 所需的粗估能力。

尤其 `calculateToolResultTokens()` 做了一个很务实的近似：

- 文本按 rough token estimation 算。
- image / document 一律按固定 2000 token 算。

它不追求精确计费，而追求“足够可靠地估算清掉这些内容大概能省多少”。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:138-164`

### 3.2 `microcompactMessages()` 的顺序很值得看

`microcompactMessages()` 里最关键的不是某个 if，而是执行顺序：

1. 先清掉 warning suppression。
2. 先看 time-based trigger。
3. 如果 time-based trigger 触发，直接短路返回。
4. 否则再考虑 cached microcompact。
5. 如果 cached microcompact 不适用，就不在这里做其他压缩，交给 autocompact。

这说明 microcompact 在生产实现里已经不是“单一功能”，而是两条分支：

- cache 还热时，优先做不破坏前缀的 cache-edit。
- cache 已冷时，直接改本地消息，把老结果清掉。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:226-293`

### 3.3 `cachedMicrocompactPath()`：缓存热的时候，不改本地消息

这段虽然在你给的主阅读范围外延伸了一点，但它正好解释了前面注释。

它做的事是：

1. 找出可 compact 的工具调用 id。
2. 在状态里登记这些 tool result。
3. 算出哪些旧结果可以删。
4. 生成 `cache_edits`，交给 API 层。
5. 本地 `messages` 不改，只返回待插入的 cache edit 信息。

这个思路很高级：

> 如果 prompt cache 还是热的，就尽量在 API 层对缓存前缀做“外科手术”，而不是直接改本地消息内容。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:305-399`

### 3.4 `evaluateTimeBasedTrigger()`：时间一长，缓存就默认冷了

这段代码回答的是：

> 什么时候不值得再做 cache-edit，而应该直接清旧内容？

它检查三件事：

1. time-based MC 配置是否开启。
2. `querySource` 是否显式属于主线程。
3. 距离上次 assistant message 是否已经超过阈值。

特别值得注意的是第二点：

> 这里要求必须有显式 `querySource`，避免 `/context`、`/compact`、`analyzeContext` 这类分析型调用误触发真正的内容清理。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:422-443`

### 3.5 `maybeTimeBasedMicrocompact()`：保留最近 N 条，其余清空

它的动作可以概括成：

1. 先根据 trigger 决定是否进入。
2. 把所有可 compact 的 tool ids 收集出来。
3. 至少保留最近 1 条。
4. 对更早的 `tool_result`，把正文替换成固定清空标记。
5. 统计省下的大致 token。
6. 重置 cached MC state，并通知 cache-deletion detector。

所以它和 `toolResultStorage.ts` 的差别非常清楚：

- `toolResultStorage.ts` 主要保住“大结果如何安全进入上下文”。
- `microCompact.ts` 主要处理“旧结果什么时候该退出活跃上下文”。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:446-520`

## 4. `autoCompact.ts`：局部减压不够时，做整段历史压缩

前面几层都还算“轻量处理”。  
`autoCompact.ts` 才是全量 compact 的触发器。

### 4.1 `getEffectiveContextWindowSize()`：先给摘要预留输出空间

这段不是简单拿模型窗口上限直接用。

它会：

1. 读模型上下文上限。
2. 预留最多 `20_000` token 给 compact summary 的输出。
3. 如果有 `CLAUDE_CODE_AUTO_COMPACT_WINDOW`，再进一步收紧窗口。

所以这里算的不是“模型理论最大窗口”，而是：

> 在还要产出 compact summary 的前提下，这次真正可安全使用的窗口。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:33-49`

### 4.2 阈值分层：warning、error、autocompact、blocking

`autoCompact.ts` 里其实有四个层次：

- warning
- error
- autocompact
- blocking

最重要的是这三个常量：

- `AUTOCOMPACT_BUFFER_TOKENS = 13_000`
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`
- `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`

可以把它理解成：

> 离真正撞墙还有不同距离时，系统会给出不同强度的处理。

其中 `blocking` 最狠，表示离极限太近，已经不该继续发 API 请求了。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:62-93`

### 4.3 `shouldAutoCompact()`：不是超阈值就一定压

这段函数里最值得学的不是阈值，而是各种“不要压”的条件：

- `session_memory` / `compact` 自己不能递归压自己。
- `marble_origami` 这种 context-collapse agent 不能误触发。
- 如果 reactive-only 模式开启，就不做主动 auto-compact。
- 如果 context collapse 已经接管上下文管理，也不做主动 auto-compact。
- 只有前面都通过，才用 `tokenCountWithEstimation(messages) - snipTokensFreed` 去比较阈值。

所以 `shouldAutoCompact()` 的真正作用不是“算个大于号”，而是：

> 先判断当前这个调用场景，到底适不适合由 autocompact 接管。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:160-239`

### 4.4 `autoCompactIfNeeded()`：先试轻一点的，再试重一点的

执行顺序是：

1. 如果整套 compact 被禁用，直接返回。
2. 如果连续失败次数已经到上限，触发 circuit breaker。
3. 调 `shouldAutoCompact()`。
4. 如果该压，先试 `trySessionMemoryCompaction()`。
5. 不够再进 `compactConversation()`。
6. 成功后做 cleanup。
7. 失败则累计 `consecutiveFailures`。

这里很能体现工程取舍：

> 系统不是一过阈值就直接做最重的 full compact，而是先试对工作连续性伤害更小的 session memory compact。

同时它还加了 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 的熔断器，防止已经救不回来的会话在每一轮里无限重试。

源码位置：

- `/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:241-351`

## 5. Compact 和 Transition 是什么关系

这一天的标题里有 `Transition`，但你给的源码片段大多是 compact 相关。  
这不是冲突，反而刚好说明了两层分工：

- compact 负责改“内容状态”
- transition 负责改“流程状态”

说得更直接一点：

- `toolResultStorage.ts`、`microCompact.ts`、`autoCompact.ts`
  回答的是“消息内容怎么变小还能继续用”。
- `TransitionReason`
  回答的是“主循环为什么会继续到下一轮”。

最典型的几种继续理由是：

- 工具刚执行完，继续喂 `tool_result`
- 输出被截断，要继续补全
- compact 刚做完，要重新发请求
- transport 出错后要退避重试

所以 Day 5 最好的心智不是把 compact 和 transition 分开看，而是连起来看：

> compact 解决“能不能继续装得下”，transition 解决“为什么继续”。两者一起，单 agent 才真的能稳定跑多轮。

建议回读：

- `docs/zh/s00c-query-transition-model.md`
- `docs/zh/data-structures.md` 里的 `TransitionReason`

## 6. 为什么这一天就意味着“阶段 1 主骨架已成立”

阶段 1 是 `s01-s06`，顺序是：

1. `s01` 让 agent 先能循环起来。
2. `s02` 让 agent 真正能做事。
3. `s03` 让 agent 不至于做大任务时乱掉。
4. `s04` 让 agent 学会上下文隔离。
5. `s05` 让 agent 不把所有知识一口气塞进 prompt。
6. `s06` 让 agent 在长会话里还能活下去。

所以阶段 1 复盘时，最该确认的不是“功能点我都背了吗”，而是下面这 5 句你能不能自己讲出来：

1. agent 为什么必须把 `tool_result` 回流进 `messages`。
2. 为什么主循环之外还得长出 planning / subagent / skills。
3. 为什么光会读写文件还不够，还得会控制上下文体积。
4. 为什么 compact 不是一把梭，而是多层机制。
5. 为什么系统必须显式知道 `TransitionReason`，而不是只写 `continue`。

如果这 5 句你都能讲清楚，那阶段 1 的主骨架就真的立住了。

## 7. 我建议你按这个顺序重读源码

### 第一遍：先抓“谁在做判断”

1. `tokens.ts:46-79,226-261`
2. `autoCompact.ts:33-93,160-239`

先搞清楚：系统是怎么判断“已经太满了”的。

### 第二遍：再抓“谁在减压”

1. `toolResultStorage.ts:137-232`
2. `toolResultStorage.ts:536-924`
3. `microCompact.ts:226-305,422-520`

再搞清楚：结果是怎么分层减压的。

### 第三遍：最后抓“为什么还能继续”

1. `autoCompact.ts:241-351`
2. `docs/zh/s00c-query-transition-model.md`

最后把“压缩动作”和“继续原因”接回主循环。

## 8. 最后用一句人话收束

`s06` 不是在教“怎么删消息”，而是在教：

> 一个真正可工作的 agent，必须同时具备两种能力:
> 一种能力叫“把细节挪出去，但不断线”；
> 另一种能力叫“知道自己为什么还能继续下一轮”。

前者对应 compact，后者对应 transition。  
Day 5 把这两件事连起来，阶段 1 的单 agent 主骨架就闭环了。
