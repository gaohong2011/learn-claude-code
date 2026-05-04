# Day 14 学习指南：Task / Runtime / Notification 三层联读

> Day 11 分清了 durable work graph。  
> Day 12 分清了 runtime task。  
> Day 13 分清了 scheduler 和未来触发。  
>
> Day 14 要把这三件事接起来：
>
> **runtime task 不是消息，notification 不是执行体，REPL 才是把执行结果重新送回主循环的接线层。**

## 0. 这一天到底在学什么

Day 14 不是再新增一种 task，而是把前几天容易散掉的几条线合成一条完整链路：

```text
用户请求 / 外部触发 / 后台完成
  ->
REPL 组装一次 query turn
  ->
query(...) 推进工具调用与运行时执行
  ->
runtime task 写入 AppState.tasks 和 output file / transcript
  ->
任务结束或被停止
  ->
生成 task-notification / SDK terminated event
  ->
通知被排队、转成 attachment / user-like message
  ->
下一轮主循环继续读到它
```

如果只记一句话，先记这个：

> **Task 层管执行槽位，Runtime 层管一次 query turn 怎么跑，Notification 层管异步结果怎样重新进上下文。**

## 1. 推荐阅读顺序

这一天最好不要从大文件 `REPL.tsx` 的第一行开始读。建议按下面顺序读：

1. `docs/zh/s00b-one-request-lifecycle.md`
   先确认一条请求的主骨架：输入、动作意图、执行、结果写回、下一轮。
2. `docs/zh/s13a-runtime-task-model.md`
   再复习 runtime task 和 durable task 的边界。
3. `docs/zh/entity-map.md`
   对齐 message、tool result、runtime task、teammate、worktree、memory 分别在哪一层。
4. `agents/s_full.py`
   看教学版最小闭环：后台执行完成后先进 notification queue，主循环再 drain。
5. `/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:2663-2863`
   看真实 REPL 怎样组装并启动一次 query turn。
6. `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalMainSessionTask.ts:94-224,338-479`
   看主会话被后台化以后怎样变成 runtime task。
7. `/Users/hong.gao/python/src/claude-code-codex/src/tasks/stopTask.ts:10-100`
   看停止协议怎样复用 task 类型的 `kill()`。
8. `/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:1737-1938,4047-4120`
   最后看恢复、调度、主动 tick 这些外部入口怎样接回 REPL。

## 2. 三层总图

Day 14 最重要的是先把三层名字叫准：

| 层 | 它回答的问题 | 典型代码 |
| --- | --- | --- |
| Task State | 系统里现在有哪些执行槽位 | `AppState.tasks`、`LocalMainSessionTaskState`、`status`、`notified` |
| Runtime Turn | 这一轮 query 怎样组装、调用、消费事件 | `REPL.onQueryImpl(...)`、`query(...)`、`toolUseContext` |
| Notification | 异步完成结果怎样重新进入上下文 | `enqueuePendingNotification(...)`、`mode: 'task-notification'` |

最短判断法：

- `task` 是一条活着或刚结束的执行槽位。
- `query turn` 是主循环当前正在推进的一轮模型调用。
- `notification` 是把异步结果送回主循环的信封。

这三层会互相连接，但不能混成一个东西。

## 3. 教学版 `s_full.py` 的最小闭环

`agents/s_full.py` 里可以先看两段：

- `BackgroundManager.run/_exec/drain`
- `agent_loop(...)`

教学版链路很干净：

```text
background_run(command)
  ->
BackgroundManager.tasks[tid] = running
  ->
后台线程执行命令
  ->
完成后 self.notifications.put(...)
  ->
agent_loop 每轮模型调用前 BG.drain()
  ->
把结果包成 <background-results> 放回 messages
```

这里的关键不是 Python 线程，而是边界：

> 后台任务自己不会直接改写模型上下文；它先发 notification，再由主循环在安全点接住。

真实实现里的 `task-notification` 就是这条教学链路的产品化版本。

## 4. `REPL.tsx:2663-2863`：Runtime Turn 的入口

这段是 Day 14 的中心，因为它解释了“主循环的一轮到底从哪里开始”。

### 4.1 查询前先准备外部环境

`onQueryImpl(...)` 一开始处理 IDE、diagnostic、onboarding、会话标题这些外围状态：

- `diagnosticTracker.handleQueryStart(...)`
- `closeOpenDiffs(...)`
- `maybeMarkProjectOnboardingComplete()`
- `generateSessionTitle(...)`

这些不是 task 层，但它们说明一件事：

> 一次 query turn 不是裸调模型，而是 REPL 先把本轮运行环境整理好。

### 4.2 `shouldQuery=false` 是“不进模型”的本地分支

`shouldQuery` 为 false 时，代码只处理本地状态：

- compact 边界会刷新 `conversationId`
- proactive / kairos 的 context-blocked 状态可能恢复
- `resetLoadingState()`
- `setAbortController(null)`

这说明 slash command、本地命令、手动 compact 这类路径不一定进入模型调用。

### 4.3 `toolUseContext` 是工具、权限、状态的胶水层

真正进入 query 前，REPL 调用：

```text
getToolUseContext(messagesIncludingNewMessages, newMessages, abortController, mainLoopModelParam)
```

这个 context 里最重要的是：

- 当前工具池
- MCP clients
- 权限上下文
- `getAppState`
- `setAppState`
- abort signal

Day 14 读这里时要意识到：

> runtime task 的注册、停止、通知，最后都要通过这层 context 读写同一份 AppState。

### 4.4 prompt 不是静态字符串，而是本轮重新构建

代码并行加载：

- `getSystemPrompt(...)`
- `getUserContext()`
- `getSystemContext()`
- 权限 / auto mode 检查

然后用 `buildEffectiveSystemPrompt(...)` 合成最终 system prompt，并写回：

```text
toolUseContext.renderedSystemPrompt = systemPrompt
```

这说明主循环看到的上下文是本轮实时构建出来的，不只是历史 `messages`。

### 4.5 `query(...)` 是事件流，不是一次性返回

核心调用是：

```text
for await (const event of query(...)) {
  onQueryEvent(event)
}
```

读代码时要把它想成：

```text
REPL 组装上下文
  ->
query 产出 user / assistant / system / progress 等事件
  ->
onQueryEvent 把事件写回 UI 和消息状态
```

也就是说，REPL 不直接执行每一种 task。REPL 的职责是把 query turn 启动起来，并消费 query 产生的事件。

## 5. `LocalMainSessionTask.ts:94-224`：把主会话后台化成 runtime task

`registerMainSessionTask(...)` 解决的是一个特殊场景：

> 用户当前这条主会话 query 还在跑，但用户按下后台化入口，希望 UI 回到新 prompt，而旧 query 继续跑。

它做了 5 件关键事。

### 5.1 生成主会话专用 task id

`generateMainSessionTaskId()` 使用 `s` 前缀：

```text
s + 8 位随机字符
```

这是为了和普通 agent task 的 `a` 前缀区分。这里的重点不是随机数，而是：

> 主会话后台任务也是 runtime task 类型族的一员，但它需要能和普通 agent task 区分。

### 5.2 输出不写回主 transcript，而写 sidechain transcript

注册时调用：

```text
initTaskOutputAsSymlink(taskId, getAgentTranscriptPath(asAgentId(taskId)))
```

注释里讲得很关键：不能用主会话 transcript。原因是主会话可能已经 `/clear`，如果后台 query 继续写原 transcript，就会污染清空后的新会话。

所以这里的边界是：

> 后台主会话有自己的 transcript / output file，和前台新会话解耦。

### 5.3 abort controller 要复用正在跑的 query

`existingAbortController ?? createAbortController()` 这行很重要。

如果后台化的是一条已经在跑的 query，就必须复用它的 abort controller。否则用户之后停止这个 background task 时，只会停掉一个新的空 controller，真正的 query 还会继续跑。

### 5.4 task state 复用 `LocalAgentTaskState`

构造出的 state 里最关键的字段是：

- `type: 'local_agent'`
- `status: 'running'`
- `agentType: 'main-session'`
- `abortController`
- `isBackgrounded: true`
- `pendingMessages: []`
- `retrieved: false`
- `notified` 通过后续流程控制

这说明它不是一个全新的 task 类型，而是借用 `local_agent` 外壳，再用 `agentType: 'main-session'` 标出身份。

### 5.5 `completeMainSessionTask(...)` 决定是否发 notification

完成时先把 task 状态改成：

- `completed`
- `failed`

并记录：

- `endTime`
- 最后一条 message

然后有一个非常重要的分支：

```text
如果仍然 isBackgrounded:
  enqueueMainSessionNotification(...)
否则:
  不发 XML notification，只标记 notified 并发 SDK terminated event
```

原因很简单：

> 如果用户已经 foreground 回这个 task，用户正在看输出，就不需要再把同一个完成事件作为 XML notification 插回主循环。

## 6. `LocalMainSessionTask.ts:338-479`：后台 query 怎样继续跑

`startBackgroundSession(...)` 是真正把后台 query 跑起来的入口。

### 6.1 先注册 task，再写初始 transcript

函数一开始调用 `registerMainSessionTask(...)`，拿到：

- `taskId`
- `abortSignal`

然后把后台化前的 `messages` 写入 sidechain transcript：

```text
recordSidechainTranscript(messages, taskId)
```

这一步让 TaskOutput 一打开就能看到上下文，而不是只看到后台化之后的新输出。

### 6.2 用 agent context 隔离技能和清理范围

代码构造：

```text
agentId: taskId
agentType: 'subagent'
subagentName: 'main-session'
```

然后包在 `runWithAgentContext(...)` 里执行。

这里看起来像小细节，但很关键：

> 后台主会话需要自己的 agentId，这样 `/clear`、skill invocation、sidechain transcript 等机制才能按 task 粒度隔离。

### 6.3 后台 query 仍然走同一个 `query(...)`

真正执行是：

```text
for await (const event of query({ messages: bgMessages, ...queryParams }))
```

这说明后台主会话不是另写了一套 agent loop。它仍然使用同一套 query runtime，只是输入 messages 和输出 transcript 换成后台 task 自己的。

### 6.4 每个事件都写回 task transcript 和 progress

循环里只保留三类事件：

- `user`
- `assistant`
- `system`

每来一条事件都会：

- push 到 `bgMessages`
- `recordSidechainTranscript([event], taskId, lastRecordedUuid)`
- 统计 assistant text token
- 统计 assistant tool_use 数
- 维护最近 5 个 tool activities
- 更新 `AppState.tasks[taskId].progress`
- 更新 `AppState.tasks[taskId].messages`

所以它一边跑，一边让 UI / TaskOutput 能看到进度。

### 6.5 abort 分支和完成分支分开

如果 `abortSignal.aborted`，代码不会走 `completeMainSessionTask(...)`，而是：

- 检查 `task.notified`
- 必要时设置 `notified: true`
- 直接发 `emitTaskTerminatedSdk(taskId, 'stopped', ...)`
- return

正常跑完才调用：

```text
completeMainSessionTask(taskId, true, setAppState)
```

异常则调用：

```text
completeMainSessionTask(taskId, false, setAppState)
```

这里的边界是：

> stop/abort 是终止协议，completed/failed 是自然完成协议；两条路径都要关上 SDK/task 生命周期，但不一定都发 XML notification。

## 7. `enqueueMainSessionNotification(...)`：Notification 层的信封

虽然学习计划点到 `LocalMainSessionTask.ts:94-224`，但要看懂 notification，必须顺手看下面的 `enqueueMainSessionNotification(...)`。

它先用 `task.notified` 做幂等保护：

```text
如果已经 notified:
  不再 enqueue
否则:
  原子地设置 notified = true
```

然后拼出 XML：

```xml
<task-notification>
<task-id>...</task-id>
<tool-use-id>...</tool-use-id>
<output-file>...</output-file>
<status>completed|failed</status>
<summary>...</summary>
</task-notification>
```

最后：

```text
enqueuePendingNotification({ value: message, mode: 'task-notification' })
```

这就是 Day 14 的关键接口：

> runtime task 完成后，不是直接把大段结果塞进当前对话，而是把 output file path 和 summary 包成 task-notification，排队等待主循环消费。

## 8. `stopTask.ts:10-100`：停止协议为什么是共享逻辑

`stopTask(...)` 被两个入口复用：

- LLM 调用的 `TaskStopTool`
- SDK 的 `stop_task` control request

所以它不关心 UI，只做共享协议。

### 8.1 三个前置校验

函数先检查：

1. task 是否存在
2. task 是否仍是 `running`
3. task type 是否有实现

对应错误码是：

- `not_found`
- `not_running`
- `unsupported_type`

这一步说明 stop 的对象是 runtime task，不是 durable work graph task。

### 8.2 真正停止交给 task 类型自己的 `kill()`

核心调用是：

```text
await taskImpl.kill(taskId, setAppState)
```

这就是 runtime task 多态的意义：

- shell 怎么 kill shell
- agent 怎么 abort agent
- remote task 怎么停远端或本地 poll

共享层只负责找到 task 类型并调它的停止入口。

### 8.3 local shell 的 notification 要特殊处理

如果是 `local_shell`，`stopTask(...)` 会把 `notified` 设成 true，抑制后续 XML notification。

原因写在注释里：被 kill 的 shell 往往只会产生类似 `exit code 137` 的噪音。用户主动停止时，不应该再收到一条“命令失败”的 task-notification。

但 SDK 仍然需要生命周期闭合，所以代码直接调用：

```text
emitTaskTerminatedSdk(taskId, 'stopped', ...)
```

### 8.4 agent 类 task 不在这里抑制通知

注释里也说明了另一半：

> Agent tasks 不抑制，因为 AbortError catch 会发送携带 partial result 的 notification。

这和 `LocalMainSessionTask` 的 abort 分支能对上：停止不是简单删除 task，而是要决定该不该把已有部分结果送回系统。

## 9. `REPL.tsx:1737-1938`：Resume 为什么属于三层联读

`resume(...)` 表面看是会话恢复，但 Day 14 要从三层角度读它。

它主要做这些事：

- `deserializeMessages(log.messages)` 清理历史消息，避免 unresolved tool_use 破坏新 turn
- 匹配 coordinator / normal mode
- 对旧 session 跑 `SessionEnd` hooks
- 对恢复后的 session 跑 `SessionStart` hooks
- fork / resume 时复制 plan
- 恢复 file history、agent setting、standalone agent context
- 恢复 read file state
- 重置 loading / abort controller
- 切换 session id 和 project dir
- 重置 transcript pointer
- 恢复 session metadata
- 恢复 worktree
- `restoreRemoteAgentTasks(...)`
- 重建 content replacement state
- `setMessages(() => messages)`

这里最值得记的是：

> Resume 不只是把 messages 塞回 UI，它要把 messages、session 文件、worktree、agent、remote runtime task 镜像重新绑到同一个运行上下文上。

如果这层没恢复好，notification 可能回到错误 session，remote task 可能丢失本地镜像，工具结果替换状态也可能找不到对应 output。

## 10. `REPL.tsx:4047-4120`：外部触发怎样接回主循环

这一段展示了除了用户手打 prompt 之外，还有哪些入口能把新意图送进 REPL。

### 10.1 scheduled task 入口

在 `AGENT_TRIGGERS` 开启时：

```text
useScheduledTasks({ isLoading, assistantMode, setMessages })
```

这和 Day 13 能接上：

- schedule record 只记未来触发条件
- 到点后生成 scheduled notification / prompt
- REPL hook 负责把它送回 messages / queue

注释里提到 assistant mode 会绕过 `isLoading` gate，是为了避免 proactive tick / Sleep 循环把 scheduler 饿死。

### 10.2 task list watcher 和 proactive tick

外部构建下还会挂：

- `useTaskListWatcher(...)`
- `useProactive(...)`

它们的共同点是：

> 都不是另起一套主循环，而是最终调用 `handleIncomingPrompt(...)` 或 `enqueue(...)`，把新输入接回 REPL。

### 10.3 priority now 会中断当前运行

这一段：

```text
if (queuedCommands.some(cmd => cmd.priority === 'now')) {
  abortControllerRef.current?.abort('interrupt')
}
```

说明 queued command 不只是排队文本，也可能携带优先级语义。`now` 优先级到来时，当前 query turn 会被打断，让更高优先级输入接管。

## 11. 一条完整链路：主会话后台化

把指定代码串起来，主会话后台化大概是这样：

```text
用户触发 session backgrounding
  ->
REPL abortController.abort('background')
  ->
REPL 移走当前队列里的 task-notification，准备转交给后台 query
  ->
REPL 构建 toolUseContext / systemPrompt / userContext / systemContext
  ->
startBackgroundSession(...)
  ->
registerMainSessionTask(...)
  ->
AppState.tasks[taskId] = running local_agent(main-session)
  ->
后台 query(...) 继续消费原 messages
  ->
事件写入 sidechain transcript + task progress
  ->
completeMainSessionTask(...)
  ->
后台状态：enqueue <task-notification>
  ->
下一轮 query drain queued command
  ->
模型看到 summary + output file path
  ->
必要时再 Read output file
```

注意最后两步：

> notification 只把结果入口送回来，不等于把完整输出塞进上下文。

这就是为什么 task notification 里最重要的字段之一是 `output-file`。

## 12. 一条完整链路：停止一个运行时任务

停止路径可以这样记：

```text
TaskStopTool / SDK stop_task
  ->
stopTask(taskId)
  ->
检查 AppState.tasks[taskId]
  ->
确认 status === running
  ->
getTaskByType(task.type)
  ->
taskImpl.kill(taskId, setAppState)
  ->
local_shell: suppressed XML notification + direct SDK terminated
agent/main-session: 让 abort catch / task 自己的终止逻辑决定通知
```

这里最重要的边界是：

> stopTask 不是“从列表里删除 task”，而是“把停止请求路由到对应 runtime task 的 kill 协议”。

## 13. 和 Day 11-13 的关系

### Day 11：durable task 管目标

```text
work graph task #12
  subject: 实现 auth 模块
  blockedBy: [...]
```

它回答的是“要做什么、谁挡着谁”。

### Day 12：runtime task 管执行槽位

```text
runtime task s8k2m1qz
  type: local_agent
  status: running
  outputFile: ...
```

它回答的是“现在谁在跑、输出在哪、怎样停止”。

### Day 13：schedule 管未来入口

```text
schedule record
  cron: 每周一 9 点
  prompt: 生成周报
```

它回答的是“未来什么时候把意图重新提起来”。

### Day 14：REPL + notification 把它们接回主循环

```text
runtime task completed
  ->
task-notification queued
  ->
query turn sees it
  ->
model decides next action
```

它回答的是：

> 异步世界里发生的事，怎样重新变成模型下一轮能理解的上下文。

## 14. 初学者最容易犯的错

### 14.1 把 `task-notification` 当成 `tool_result`

`tool_result` 是一次工具调用的直接返回。  
`task-notification` 是异步任务完成后的回注信封。

它们最后都可能被模型读到，但来源和时机不同。

### 14.2 把 `stopTask` 理解成删除 task

停止不是删除。停止要：

- 校验状态
- 调类型自己的 `kill()`
- 处理是否通知
- 关闭 SDK 生命周期

### 14.3 忽略 `notified`

`notified` 是幂等阀门。没有它，后台任务可能重复发完成通知，SDK 也可能收到重复终态事件。

### 14.4 认为 foreground 后还应该发 XML notification

如果用户已经把后台主会话 foreground 回来，完成时不再发 XML notification 是合理的。用户正在看这条输出，再回注一次反而会重复。

### 14.5 把 resume 当成纯 UI 恢复

Resume 必须恢复 session、worktree、agent、file history、remote task 镜像、content replacement。否则后续 notification 可能没有正确上下文。

## 15. 一句话记住

**Day 14 的核心不是“又有一种任务”，而是：REPL 负责把 runtime task、外部触发和异步通知重新接回同一条 query loop。**
