# Day 10 输出物：Error Recovery 状态机与阶段 2 控制面总结

Day 10 最重要的不是把错误名字背下来，而是把“主循环为什么还能继续”分清楚。

## 1. 恢复状态机

```text
一轮 query 执行
  |
  +-- 正常工具往返 / 正常主线继续
  |      -> continue
  |      -> transition.reason = next_turn
  |
  +-- 输出被截断
  |      -> continue
  |      -> 先 max_output_tokens_escalate
  |      -> 再 max_output_tokens_recovery
  |
  +-- prompt 太长 / media 太大
  |      -> compact
  |      -> 先 collapse_drain_retry
  |      -> 再 reactive_compact_retry
  |
  +-- API / transport / streaming 临时异常
  |      -> backoff
  |      -> withRetry / fallback / cleanup
  |
  +-- stop hook 阻塞 / token budget 要求再推进
  |      -> continue
  |      -> stop_hook_blocking / token_budget_continuation
  |
  +-- 恢复预算耗尽或不可恢复
         -> fail
```

## 2. 四类动作最土的解释

| 动作 | 最简单理解 | 真实源码里主要怎么体现 |
|---|---|---|
| `continue` | 这轮还没真结束，还得再推一轮 | `next_turn`、`max_output_tokens_recovery`、`stop_hook_blocking`、`token_budget_continuation` |
| `compact` | 桌子放不下了，先收拾上下文再继续 | `collapse_drain_retry`、`reactive_compact_retry` |
| `backoff` | 这次更像临时抖动，先等等再试 | `claude.ts` 里的 `withRetry`、fallback、stream cleanup |
| `fail` | 这不是能无限补救的情况，该显式停下 | withheld error surface、恢复预算耗尽、不可恢复 API error |

## 3. 三条关键边界

### 3.1 `continue` 不等于“没出错”

Day 10 最该记住的是：

```text
continue 是动作
reason 才是原因
```

所以真实实现里不是只写一个 `continue`，而是把原因分成：

- `next_turn`
- `max_output_tokens_escalate`
- `max_output_tokens_recovery`
- `collapse_drain_retry`
- `reactive_compact_retry`
- `stop_hook_blocking`
- `token_budget_continuation`

### 3.2 `compact` 不是只有一种

遇到 `prompt too long` 时，系统不是直接 full compact。

更接近真实的顺序是：

```text
先 drain staged collapse
不够再 reactive compact
还不行才把错误抛出来
```

所以 Day 10 学到的不是“会 compact”，而是“恢复也有分层优先级”。

### 3.3 `backoff` 不只是在 `query.ts` 里写 if

真实代码里，很多恢复底座在 `claude.ts`：

- 请求前清洗消息，避免明显 400
- streaming 请求包在 `withRetry(...)`
- 必要时转 non-streaming fallback
- 失败后 `cleanupStream()`
- `updateUsage()` 防止流式 0 覆盖真实 usage

也就是说：

> `query.ts` 管状态转移，`claude.ts` 管 API 层稳定性。

## 4. `tokenBudget.ts` 一句话说明

`tokenBudget.ts` 不是在做报错恢复，而是在决定：

> 现在是该继续推一轮，还是该收尾停下。

它的价值是补齐了另一类 continuation：

- 不是因为失败才继续
- 而是因为系统判断“还值得继续”

## 5. 阶段 2 复盘：为什么说控制面已经成立

阶段 2 是 `s07-s11`。  
到 Day 10 为止，单 agent 已经不只是“会调工具”，而是有了下面这 5 层控制面：

| 层 | 回答的问题 |
|---|---|
| Permission | 这件事能不能做 |
| Hook | 运行时能不能被拦截、扩展、回注 |
| Memory | 什么该当前记，什么该跨会话记 |
| Prompt Pipeline | 模型最终到底看见什么 |
| Error Recovery | 这轮为什么继续、何时恢复、何时停止 |

把这五层合起来，就是：

```text
会做事
  + 有权限边界
  + 有运行时切面
  + 有记忆分层
  + 有输入装配
  + 有恢复状态机
= 高完成度单 agent 的控制面成立
```

## 6. 一句话收口

> Day 10 真正补上的，不是一个“重试功能”，而是“单 agent 在异常、截断、超长和收益递减时，如何继续、压缩、退避或停止”的完整控制面。
