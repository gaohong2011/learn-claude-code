# Day 5 补图：Skill Loading、Tool Result Budget、Microcompact、Autocompact、TransitionReason

> 这份补图把 `s05`、`s06` 和 `s00c` 串成一条线。  
> 如果你已经读过 `docs/zh/s05-skill-loading.md`、`docs/zh/s06-context-compact.md` 和 `docs/zh/s00c-query-transition-model.md`，可以把这篇当成 Day 5 的总收束。

## 一张关系图

先把五个模块放回同一条 query 主线里看：

```text
用户任务
  |
  v
system prompt
  └─ 只放轻量 skill catalog
       - skill 名称
       - 一句话描述
  |
  v
query loop -> LLM call
  |
  +-- 模型判断“需要专门知识”
  |      |
  |      v
  |   load_skill
  |      |
  |      v
  |   读取 SKILL.md
  |      |
  |      v
  |   以 tool_result 注入当前 messages
  |
  +-- 模型调用普通工具
         |
         v
      执行工具
         |
         v
      产生 tool_result
              |
              v
      tool result budget
        - 单个结果太大 -> 落盘 + 留预览
        - 单条 message group 太大 -> 替换最大 fresh 结果
              |
              v
      microcompact
        - 旧 tool_result 退出活跃窗口
        - 保留最近热结果，较老结果改占位或 cache edit
              |
              v
      token estimation
        - 当前上下文是否逼近窗口上限
              |
              v
      autocompact
        - 轻量减压不够 -> 生成摘要 / 压缩历史
              |
              v
      TransitionReason
        - tool_result_continuation
        - budget_continuation
        - compact_retry
        - max_tokens_recovery
        - transport_retry
              |
              v
下一轮 query 继续
```

这张图里最重要的，不是“五个名词都出现了”，而是要看清它们各自管的是哪一层：

- `skill loading` 管知识入口：什么知识值得进当前上下文。
- `tool result budget` 管结果入口：单条结果以什么形态进入当前上下文。
- `microcompact` 管活跃窗口：哪些旧结果不该继续原样霸占工作面。
- `autocompact` 管整体容量：整段历史已经太长时，怎样保连续性地整体缩写。
- `TransitionReason` 管流程状态：这轮为什么继续，而不是结束。

因此，前四个模块主要在改**内容状态**，最后一个模块主要在改**流程状态**。

## 为什么这五块连起来，才叫“单 agent 主骨架已成立”

如果只看 `s01-s04`，你会觉得 agent 已经会循环、会调工具、会做规划、还会派子 agent，像是已经很完整了。  
但从系统骨架的角度看，它其实还缺两种决定性的能力：

第一种能力，是**按需引入知识，而不是把所有说明永远塞在 prompt 里**。  
`s05` 的 `skill loading` 解决的是这个问题。它把知识分成两层：平时只在 system prompt 里放一个廉价目录，让模型知道“有哪些可用能力”；真遇到某类任务时，再通过 `load_skill` 把完整说明作为 `tool_result` 注入当前上下文。这样做不是单纯省 token，而是在建立一种更成熟的知识边界：默认上下文里只保留当前任务最相关的知识，其余知识保持可发现、但不常驻。

第二种能力，是**在长会话里活下去，而不是靠一口气把历史全背着跑**。  
`s06` 把这件事拆成三层。`tool result budget` 先解决“新结果太大、根本不该原样塞进来”的问题；`microcompact` 再解决“老结果虽然曾经有用，但现在不值得继续占活跃窗口”的问题；`autocompact` 最后兜底，处理“整段历史已经整体过长”的问题。这样 agent 对上下文的态度，就从“全都带着”变成了“按热度和作用分层保留”。

但只有内容压缩还不够，因为系统还必须知道：**每次继续下一轮，到底是正常推进，还是恢复路径，还是压缩后重试。**  
这就是 `TransitionReason` 的位置。它不负责删内容，也不负责摘要历史；它负责把 query loop 的每一次继续都写成显式的流程原因。工具刚执行完继续，是 `tool_result_continuation`；预算或策略允许继续，是 `budget_continuation`；compact 完需要重发，是 `compact_retry`。有了这层显式状态，日志、测试和恢复逻辑才不会混成一团“反正 continue 了”。

所以，Day 5 真正收束出来的，不是一个“压缩功能点合集”，而是一条完整的单 agent 生存链：

```text
知道有哪些知识可用
  ->
需要时再把知识载入当前上下文
  ->
调用工具获得结果
  ->
控制结果如何进入上下文
  ->
清理旧结果，维持活跃窗口
  ->
必要时整体压缩历史
  ->
显式记录为什么继续下一轮
```

这条链一旦成立，单 agent 才不再只是一个“能调 API、能跑几个工具”的 demo，才开始像一个真正可持续工作的编码 agent。  
它已经具备了最核心的五种能力：

- 会循环，不是一轮问答。
- 会行动，不只是生成文字。
- 会选择知识，不把所有说明都常驻。
- 会管理上下文，不把所有结果都永远背着。
- 会解释继续原因，不把所有路径都混成同一种 `continue`。

这就是为什么可以说：到了 `s06 Day 5`，阶段 1 的**单 agent 主骨架已成立**。  
后面的 memory、system prompt assembly、recovery、permissions、hooks，本质上是在这条骨架上继续长器官；但骨架本身，到这里已经站住了。
