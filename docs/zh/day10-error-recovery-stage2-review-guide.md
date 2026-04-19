# Day 10 教学讲义：Error Recovery 与阶段 2 复盘

> 这份讲义面向第一次读真实恢复控制面源码的人。  
> Day 10 最重要的不是记住几个报错名字，而是看懂：
>
> **单 agent 为什么不能只有一个 `continue`，而必须把“为什么继续、怎么恢复、何时停止”做成显式控制面。**

## 0. 这一天到底在学什么

到 Day 9 为止，你已经有：

- agent loop
- tool use
- todo / subagent / compact
- permission / hooks / memory
- system prompt pipeline

这时系统已经能做很多事，但还少最后一个关键能力：

> **遇到不完整输出、超长上下文、流式失败、临时 API 抖动时，系统不能直接崩，而要先判断走哪条恢复路径。**

所以 Day 10 真正学的是这条链：

```text
query loop
  ->
判断这轮为什么还要继续
  ->
continue / compact / backoff / fail
  ->
记录 transition reason
  ->
主循环安全推进或明确结束
```

如果只记一句话，可以先记这个：

> Error Recovery 不是“出错了重试一下”，而是“把继续推进 query 的理由显式化”。

## 1. 推荐阅读顺序

建议按下面顺序读，别一上来就闷头啃大文件：

1. `docs/zh/s11-error-recovery.md`
   先建立教学版里三条恢复路径：续写、压缩、退避
2. `docs/zh/s00c-query-transition-model.md`
   再把“为什么继续下一轮”这件事单独看清
3. `agents/s11_error_recovery.py`
   先看最小教学版如何把三类错误接进 while loop
4. `/Users/hong.gao/python/src/claude-code-codex/src/query.ts:950-1034,1152-1458`
   看真实产品里 continuation / compact / hook / token budget 怎么串成控制面
5. `/Users/hong.gao/python/src/claude-code-codex/src/query/tokenBudget.ts:13-93`
   看“继续”不只来自错误恢复，也可能来自主动预算策略
6. `/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:764-921,1358-1686`
   看 API 层怎么做 retry、fallback、请求清洗和 streaming 保护
7. `/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:3087-3182,3402-3588`
   最后补 usage 更新、参数修正、输出上限等底层边界

原因很简单：

- `s11-error-recovery.md` 告诉你“恢复分类”
- `s00c-query-transition-model.md` 告诉你“继续原因为什么要独立建模”
- `query.ts` 告诉你“控制面真正落在哪”
- `claude.ts` 告诉你“API 调用层怎么把这些恢复做稳”

## 2. 先背一张总图

先把 Day 10 的主线图背下来：

```text
模型调用一轮结束
  |
  +-- 正常工具往返 / 正常下一轮
  |      -> transition = next_turn
  |
  +-- 输出被截断
  |      -> 先 max_output_tokens_escalate
  |      -> 再 max_output_tokens_recovery
  |
  +-- prompt 太长 / media 太大
  |      -> 先 collapse_drain_retry
  |      -> 再 reactive_compact_retry
  |
  +-- hook 阻塞本轮结束
  |      -> transition = stop_hook_blocking
  |
  +-- token budget 觉得还能继续
  |      -> transition = token_budget_continuation
  |
  +-- API / transport / streaming 异常
  |      -> claude.ts 先 retry / fallback / cleanup
  |
  +-- 恢复预算耗尽或不可恢复
         -> surface error / stop / fail
```

这张图里最关键的是：

- `continue` 不是一种原因，而是一类动作
- 真正该读懂的是不同的 `transition.reason`

## 3. 读 `query.ts` 时只抓 5 个点

### 3.1 先抓 `State.transition`

这是 Day 10 最核心的字段：

```ts
transition: Continue | undefined
```

它回答的是：

> 上一轮为什么会继续到这一轮？

没有这个字段，后面所有 compact retry、续写恢复、hook 阻塞、token budget continuation 都会混成一锅。

### 3.2 prompt too long 恢复是两段式

读 `1152-1259` 时，只抓这条顺序：

```text
withheld 413
  ->
先 collapse drain
  ->
再 reactive compact
  ->
还不行再 surface error
```

不要只记“会 compact”，要记：

> 真实产品先尝试局部、便宜、保留颗粒度的恢复；不够时才上完整 compact。

### 3.3 max tokens 恢复分“提额”和“续写”两层

读 `1292-1353` 时，要分开看：

- `max_output_tokens_escalate`
- `max_output_tokens_recovery`

它不是一遇到截断就立刻发“继续写”，而是先看能不能同轮把输出上限提上去。

### 3.4 stop hook 也属于 continuation reason

读 `1381-1415` 时要意识到：

> hook 阻止本轮结束，不是“普通错误”，而是受控 continuation。

这会帮助你把 Day 7 hook 和 Day 10 recovery 真正连起来。

### 3.5 token budget continuation 说明“继续”也可能是主动策略

`1420-1458` 最值得抓的不是阈值，而是这个思想：

> 系统并不是只有在出问题时才继续；有时它会因为“还没到该收尾的时候”而主动推进一轮。

## 4. `tokenBudget.ts` 要看懂什么

这份文件很短，所以更容易读偏。

真正要读懂的是：

```text
budget 不是配额统计表
它是“现在该继续还是该收手”的判断器
```

重点看两个判断：

- `turnTokens < budget * COMPLETION_THRESHOLD`
- `isDiminishing`

把它翻成人话：

- 如果预算还没到危险区，而且最近推进还有效，就可以再 nudged 一轮
- 如果已经连续推进多轮，最近新增 token 又很少，说明收益递减，该停了

所以这一天它被放进来，是为了补全：

```text
什么时候因为失败继续
什么时候因为策略继续
什么时候该明确停止
```

## 5. `claude.ts` 不是“API 封装文件”，而是恢复底座

读你指定的几个区间时，建议按这几个问题看：

### 5.1 请求发出去前做了哪些防炸预处理

看 `1358-1686`：

- tool_use / tool_result 配对修复
- 不支持的 block 剥离
- 过多 media 剥离
- model 切换时的 tool search 字段清洗

这类逻辑都在说明：

> 最好的错误恢复，是别把明显会 400 的请求发出去。

### 5.2 streaming 路径怎么和 retry / fallback 接起来

看 `764-921` 和 `1840-2010`：

- `withRetry(...)`
- 流式请求
- watchdog / timeout
- non-streaming fallback
- model fallback

把它读成一句话：

> 调模型不是一个函数调用，而是一条带 attempt、timeout、fallback、cleanup 的执行链。

### 5.3 为什么 `cleanupStream()` 和 `updateUsage()` 也属于 Day 10

看 `3087-3182`：

- `cleanupStream()` 说明恢复不只是逻辑恢复，还包括资源回收
- `updateUsage()` 说明流式 usage 是累计值，不能被后续 0 覆盖坏

这些看起来不像“错误处理”，但它们都在保护恢复判断的地基。

### 5.4 non-streaming fallback 为什么还要修正参数

看 `3402-3588`：

- `adjustParamsForNonStreaming(...)`
- `getMaxOutputTokensForModel(...)`

这里最该懂的是：

> fallback 不只是“换条路再试”，还必须保证换路之后的参数仍满足 API 约束。

## 6. 和教学版怎么对照

先把 Day 10 教学版和真实产品版对成下面这张表：

| 教学版 | 真实实现里更细的对应 |
|---|---|
| `continue` | `next_turn`、`max_output_tokens_escalate`、`max_output_tokens_recovery`、`stop_hook_blocking`、`token_budget_continuation` |
| `compact` | `collapse_drain_retry`、`reactive_compact_retry` |
| `backoff` | `withRetry`、streaming fallback、model fallback、transport retry |
| `fail` | surface withheld error、stop hook prevent、预算耗尽、不可恢复 API error |

这样你就不会把教学版误解成“产品只有那三个 if”。

## 7. 阶段 2 复盘时要问自己什么

阶段 2 是 `s07-s11`。读完 Day 10，最该确认的是你能不能自己讲出这几句：

1. 权限系统决定“能不能做”
2. hook 系统决定“运行时能不能被拦截或扩展”
3. memory 系统决定“什么该长期记、什么只该当前会话记”
4. prompt pipeline 决定“模型最终到底看见什么”
5. error recovery 决定“遇到中断和异常时主循环怎么继续或停止”

如果这五句你能自己讲顺，说明你真的看懂“高完成度单 agent 的控制面已经成立”。

## 8. 推荐产出方式

这一天最适合交两样东西：

1. 一张恢复状态机
2. 一页阶段 2 复盘总结

状态机至少要有：

- continue
- compact
- backoff
- fail

复盘总结至少要回答：

- 单 agent 到 Day 10 已经具备哪些控制面
- 为什么 `transition.reason` 是关键状态
- 为什么恢复不只是一种 retry

## 9. 一句话收口

> Day 10 让你第一次真正看见：稳定的 agent 不是“模型没出错”，而是“系统知道出了什么错、为什么还能继续、何时必须停”。 
