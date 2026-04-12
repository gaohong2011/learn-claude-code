# Day 3 输出物：`Todo / Plan / Subagent` 对比表

Day 3 把 `s03` 的会话规划和 `s04` 的一次性委派放在一起看，最容易混淆的点是：`Todo`、`Plan`、`Subagent` 不是三个平级机制。`Todo` 是写计划的入口，`Plan` 是被写出来的会话状态，`Subagent` 是被派出去执行局部任务的独立上下文。

## 0. 一些理解

- `Todo` 解决的是“当前会话怎么不乱撞”，`Subagent` 解决的是“局部工作不要污染主上下文”。
- `Todo` 不会创建新上下文，它只是把当前会话的计划写到外部状态里。
- `Todo` 的PROMPT:[TodoWriteTool/prompt.ts](/Users/hong.gao/python/src/claude-code-codex/src/tools/TodoWriteTool/prompt.ts:3)
- `Subagent`的PROMPT:[AgentTool/prompt.ts](/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/prompt.ts:190)相对复杂，分静态部分和动态部分，静态部分这部是不变的，方便LLM cache，降低成本，动态部分通过remind消息动态加入，因为可用的agent可能会变化的(会因为 MCP、插件、权限规则变化而变化)，如果放一起LLM cache就不能用了

## 1. 对比表

| 对象 | 本质定位 | 状态存放位置 | 上下文边界 | 生命周期 | 返回结果方式 |
|---|---|---|---|---|---|
| `Todo` | 改写当前会话计划的工具入口，不是计划本身 | 教学版里由 `todo -> TODO.update()` 写入 `TodoManager.state` / `PlanningState`；参考实现写入 `appState.todos[todoKey]`，其中 `todoKey = agentId or sessionId` | 不创建新上下文，直接作用于当前父会话或当前 agent 的计划槽位 | 工具调用本身是瞬时的；写下去的计划会持续存在，直到下次改写或会话结束 | 以 `tool_result` 立即回流。教学版返回渲染后的 checklist；参考实现返回 “Todos have been modified successfully...” 一类结果 |
| `Plan` | 被外显出来的会话计划状态，用来维持焦点和推进顺序 | 教学版是 `PlanningState.items + rounds_since_update`；参考实现是 `appState.todos[todoKey]` | 属于某个 session / agent 的状态，不是持久任务图，也不是多 agent 共享工作图 | 跨多轮存在，可反复重写；参考实现里如果 plan 属于 subagent，会在该 subagent 退出时清理 | 它本身不是执行器，不主动“返回”；通常通过 `todo` 的渲染文本、loop 的 reminder 注入、或状态快照被主循环消费 |
| `Subagent` | 一次性委派执行器，核心价值是干净上下文 | 教学版主要存在于子 `messages`、受限 `tools` 和局部执行状态；参考实现还有 `agentId`、subagent context、transcript、metadata 等 | 默认是 fresh child context；共享文件系统，但不共享父对话历史；fork 时才继承父消息 | `spawn -> 若干轮工具执行 -> summary -> 销毁 / cleanup`；教学版安全上限是 30 轮 | 同步路径下把 summary 作为 `tool_result` 返回父 agent；父 agent 再决定如何整理成面对用户的最终表述 |

## 2. 关系图

```text
用户提出多步任务
  |
  +-> Todo tool
  |     |
  |     v
  |   Plan state
  |   - [ ] pending
  |   - [>] in_progress
  |   - [x] completed
  |
  +-> task tool
        |
        v
      Subagent
      - fresh messages[]
      - filtered tools
      - local execution
        |
        v
      summary -> tool_result -> 父 agent 继续
```

最小归纳就是：

- `Todo` 是写入口。
- `Plan` 是当前会话的显式状态。
- `Subagent` 是一次性执行者。

## 3. 代码锚点

- 教学版 `PlanItem / PlanningState / TodoManager`：[`agents/s03_todo_write.py`](../../agents/s03_todo_write.py)
- 教学版 `todo` 更新与 reminder 回流：[`agents/s03_todo_write.py`](../../agents/s03_todo_write.py)
- 教学版 `run_subagent()` 与 `task` 工具：[`agents/s04_subagent.py`](../../agents/s04_subagent.py)
- 中文主线说明：[`s03-todo-write.md`](./s03-todo-write.md)、[`s04-subagent.md`](./s04-subagent.md)
- 实体边界图：[`entity-map.md`](./entity-map.md)
- 参考实现 `TodoWriteTool`：`src/tools/TodoWriteTool/TodoWriteTool.ts:31-115`
- 参考实现 `AgentTool / runAgent`：`src/tools/AgentTool/AgentTool.tsx`、`src/tools/AgentTool/runAgent.ts`

## 4. 一句话结论

`Todo / Plan` 属于会话内过程规划层，`Subagent` 属于执行隔离层；前者回答“现在按什么步骤推进”，后者回答“这段局部工作该在哪个上下文里做”。
