# Day 15 学习指南：阶段 3 复盘与小型原型

> Day 11 学的是“工作目标怎样持久化”。  
> Day 12 学的是“当前执行槽位怎样活着跑”。  
> Day 13 学的是“未来触发怎样重新进入主循环”。  
> Day 14 学的是“task / runtime / notification / transcript 怎样分层”。  
>
> Day 15 要把这几层压成一个最小原型：
>
> **durable task 负责目标，runtime slot 负责现场，schedule queue 负责未来入口，query loop 负责把结果继续送回模型。**

## 0. 这一天到底在学什么

Day 15 是阶段 3 的收口日，不是再开一条新主线。

它要回答的问题是：

> 如果只保留最少结构，怎样搭出一个能长期推进工作的系统？

最小链路可以这样看：

```text
durable task graph
  记录长期目标和依赖
  ->
runtime slot
  记录当前执行单元、状态、输出文件
  ->
schedule queue
  记录未来什么时候把 prompt 再提起来
  ->
notification / queued attachment
  把异步结果或未来触发带回 query loop
  ->
query recursive turn
  让模型继续决定下一步
```

如果只记一句话，先记这个：

> **阶段 3 的成果，是系统从“当前对话能干活”升级成“会话外状态也能持续推动下一轮”。**

## 1. 推荐阅读顺序

这一天建议按“阶段复盘 -> 教学版三件套 -> 真实回注点”的顺序读。

### 必读

1. `docs/zh/s00d-chapter-order-rationale.md`
   先确认为什么 `s12-s14` 是一组：持久任务图、运行时槽位、时间触发器。
2. `docs/zh/s00e-reference-module-map.md`
   再确认参考仓库里对应模块：`tasks/*`、`ScheduleCronTool/*`、`query`、runtime task union。
3. `agents/s12_task_system.py`
   看 durable task graph 的最小状态：`TaskManager`、`blockedBy`、`blocks`、`owner`。
4. `agents/s13_background_tasks.py`
   看 runtime slot 的最小状态：`BackgroundManager.tasks`、`output_file`、notification queue。
5. `agents/s14_cron_scheduler.py`
   看 schedule queue 的最小状态：`CronScheduler.tasks`、`_check_tasks(...)`、`drain_notifications(...)`。

### 核心真实代码

- `/Users/hong.gao/python/src/claude-code-codex/src/query.ts:1478-1903`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`

这两段最值得读，因为它们回答真实系统里的两个问题：

- 哪些对象算 runtime task，哪些应该显示成后台任务
- 异步结果 / 队列消息怎样在 query 递归前被接回上下文

## 2. 阶段 3 的位置

`s00d` 把整套课程分成几段：

```text
s06 之后：单 agent 主骨架成立
s11 之后：更稳、更可控的单 agent 成立
s14 之后：工作系统从聊天过程升级成可持续运行时
```

所以 Day 15 的复盘点不是“又多学了三个功能”，而是：

> 系统第一次拥有了三类会话外状态：工作目标、执行现场、未来触发。

这三类状态分别来自：

| 来源 | 新增能力 | 关键边界 |
| --- | --- | --- |
| Day 11 / `s12` | durable task graph | 工作目标可以脱离当前 messages |
| Day 12 / `s13` | runtime slot | 执行单元可以脱离前台等待 |
| Day 13 / `s14` | schedule queue | 未来时间可以成为新输入入口 |
| Day 14 | notification / transcript 边界 | 异步结果有回注信封和追溯记录 |

## 3. `agents/s12_task_system.py`：durable task graph

Day 15 读 `s12_task_system.py`，不要再把它当作“任务工具 demo”。它在阶段 3 原型里承担的是目标层。

### 3.1 `TaskManager.create(...)`

核心字段是：

```text
id
subject
description
status
blockedBy
blocks
owner
```

这说明 durable task 关心的是：

- 这条工作是什么
- 现在处于什么工作状态
- 它被谁阻塞
- 它完成后解锁谁
- 谁在负责

它不关心：

- 当前有没有线程在跑
- 输出文件在哪
- 是否已经发过 notification

这就是 Day 15 原型里第一条硬边界：

> durable task 是目标层，不是执行层。

### 3.2 `TaskManager.update(...)`

`update(...)` 里最值得看的是两段：

```text
status == "completed" -> _clear_dependency(task_id)
add_blocks -> 同步写入对方 blockedBy
```

这说明任务图不是简单列表，它有 ready / blocked 语义。

最小规则是：

```text
status == pending 且 blockedBy 为空
  ->
这条工作现在可以被启动
```

所以 Day 15 的原型里，durable task 只需要能表达：

- create
- block
- unblock
- claim
- complete

## 4. `agents/s13_background_tasks.py`：runtime slot

Day 12 已经讲过后台执行，Day 15 要把它看成原型的第二层：执行现场。

### 4.1 `BackgroundManager.run(...)`

`run(...)` 创建的不是 durable task，而是 runtime slot：

```text
id
status: running
result: None
command
started_at
finished_at
result_preview
output_file
```

这里最关键的是 `output_file`。它说明执行结果不需要完整塞进消息流，通知里只需要带一个可读取入口。

### 4.2 `_execute(...)`

后台线程完成后做 4 件事：

1. 写完整输出到 `.runtime-tasks/<task_id>.log`
2. 更新 runtime slot 状态
3. 持久化 runtime record
4. 把一条 completion notification 放入 `_notification_queue`

也就是说：

```text
runtime slot finished
  ->
write output file
  ->
enqueue notification
```

这就是 Day 15 原型里第二条硬边界：

> runtime slot 自己只负责执行现场和完成通知，不直接决定 durable task 是否完成。

### 4.3 `agent_loop(...)` 的 drain

`s13` 的 `agent_loop(...)` 在模型调用前做：

```text
notifs = BG.drain_notifications()
messages.append(<background-results>...</background-results>)
```

这一步是阶段 3 的核心动作之一：

> 异步执行结果只有在主循环安全点才重新进入上下文。

## 5. `agents/s14_cron_scheduler.py`：schedule queue

Day 13 讲 schedule 时已经强调过，它不是 runtime task。Day 15 要把它看成原型的第三层：未来入口。

### 5.1 `CronScheduler.create(...)`

schedule record 的最小字段是：

```text
id
cron
prompt
recurring
durable
createdAt
jitter_offset
```

它回答的是：

- 什么时候触发
- 触发时注入什么 prompt
- 是否重复
- 是否跨会话保留

它不回答：

- 谁现在正在跑
- 输出文件在哪
- 这条工作目标依赖谁

### 5.2 `_check_tasks(...)`

`_check_tasks(now)` 是 schedule queue 的核心：

```text
for task in self.tasks:
  if expired: remove later
  check_time = now - jitter
  if cron_matches(task["cron"], check_time):
    queue.put("[Scheduled task ...]: prompt")
    task["last_fired"] = time.time()
    if one-shot: mark for removal
```

这里的重点是：

> scheduler 命中以后只 enqueue prompt，不直接启动 runtime slot。

是否启动 runtime slot，要等主循环读到 prompt 后由模型决定。

### 5.3 `agent_loop(...)` 的 drain

`s14` 的主循环在模型调用前：

```text
notifications = scheduler.drain_notifications()
for note in notifications:
  messages.append({"role": "user", "content": note})
```

这和 `s13` 的后台结果回注是同一种结构：

```text
queue
  ->
drain before LLM call
  ->
inject as user-like message
  ->
model decides next action
```

## 6. `tasks/types.ts:1-46`：真实 runtime task union

真实代码里，runtime task 不是一个类，而是一组具体 state 的 union：

```text
LocalShellTaskState
LocalAgentTaskState
RemoteAgentTaskState
InProcessTeammateTaskState
LocalWorkflowTaskState
MonitorMcpTaskState
DreamTaskState
```

这说明参考实现比教学版多了很多执行槽位类型，但它们仍然回答同一组问题：

- 是否 running / pending
- 是否应该显示在后台任务区
- 是否有前后台切换语义

`isBackgroundTask(...)` 的核心判断是：

```text
status 必须是 running 或 pending
如果有 isBackgrounded 字段，不能是 false
```

这段代码最适合拿来校准 Day 15 原型：

> 原型可以只有一种 runtime slot，但接口要预留“多种执行槽位同属一层”的心智。

## 7. `query.ts:1478-1903`：真实回注点

这段是真实系统里最该讲的核心代码。

它不是 task manager，也不是 scheduler，但它决定：

> 工具结果、通知、附件和下一轮 query 怎样接起来。

### 7.1 工具执行阶段

开头先判断有没有 `toolUseBlocks`，然后执行：

```text
runTools(...) 或 streamingToolExecutor.getRemainingResults()
for await update:
  yield update.message
  toolResults.push(normalized user tool_result)
  update new toolUseContext
```

这说明工具调用不是直接“返回字符串给模型”，而是先变成标准消息，再被追加到本轮 `toolResults`。

### 7.2 abort / hook stop 是递归前的硬停点

如果工具执行中被 abort：

```text
yield interruption message
finishTurn('aborted_tools')
return
```

如果 hook 要阻止续行：

```text
finishTurn('hook_stopped')
return
```

这一步提醒你：

> 不是所有工具执行后都会进入下一轮；控制面可以阻止递归。

### 7.3 queued command drain 是 notification 回注点

最重要的是这一段：

```text
const queuedCommandsSnapshot = getCommandsByMaxPriority(
  sleepRan ? 'later' : 'next',
).filter(...)
```

它处理的是 pending prompts / task-notifications：

- 主线程只 drain `agentId === undefined`
- subagent 只 drain 发给自己的 `task-notification`
- slash command 不在这里混进模型文本
- `Sleep` 跑过时可以消费 `later` 优先级，否则只消费 `next`

这就是 Day 15 原型必须模拟的动作：

> queue 不是随便 append 到 messages，而是在 query 的安全阶段按优先级和归属 drain。

### 7.4 queued command 变成 attachment

接下来：

```text
for await (const attachment of getAttachmentMessages(... queuedCommandsSnapshot ...)) {
  yield attachment
  toolResults.push(attachment)
}
```

这一步把 queue 中的 prompt / task-notification 转成模型可见的 attachment。

然后：

```text
removeFromQueue(consumedCommands)
```

这说明 notification 回注要有“消费确认”，否则下一轮会重复读到同一条通知。

### 7.5 递归进入下一轮

最后构造：

```text
next = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  turnCount: nextTurnCount,
  transition: { reason: 'next_turn' },
}
```

这就是阶段 3 的真实闭环：

```text
工具结果 / notification attachment
  ->
追加到 messages
  ->
transition: next_turn
  ->
下一轮模型继续推理
```

## 8. Day 15 最小原型应该长什么样

如果你要实现一个最小 “task + runtime slot + schedule queue” 原型，建议不要一开始追求产品完整性。

只需要 4 个结构：

```text
TaskStore
  - create / update / ready / complete

RuntimeSlotManager
  - spawn / check / finish / output_file

ScheduleQueue
  - create / tick / drain

AgentLoop
  - drain runtime notifications
  - drain schedule notifications
  - inject into messages
  - run one turn
```

最小状态机：

```text
durable task:
  pending -> in_progress -> completed
  pending + blockedBy -> blocked

runtime slot:
  running -> completed
  running -> failed
  running -> killed

schedule record:
  waiting -> fired
  fired -> waiting       recurring
  fired -> deleted       one-shot

notification:
  queued -> injected -> consumed
```

最小触发链：

```text
schedule fires
  ->
inject prompt
  ->
model chooses start runtime slot
  ->
runtime slot completes
  ->
notification injected
  ->
model updates durable task
```

## 9. 初学者最容易犯的错

### 9.1 把 schedule 直接做成 runtime slot

schedule 只是未来入口，真正执行要等 prompt 回到主循环后再决定。

### 9.2 把 runtime slot 写回 durable task 状态

runtime slot 完成不等于工作目标完成。模型可能还要读 output file、修代码、再跑测试。

### 9.3 不做 notification 消费语义

如果 queue drain 后不 remove，下一轮会重复注入同一条 notification。

### 9.4 忽略 query 的安全回注点

真实 `query.ts` 里非常小心：工具结果、attachments、queued commands 都在固定阶段汇合，避免和 API 的 tool_result 顺序冲突。

## 10. 一句话记住

**Day 15 的最小原型不是三个功能拼盘，而是一条状态闭环：目标在 task，执行在 runtime，未来入口在 schedule，回注和续行在 query loop。**
