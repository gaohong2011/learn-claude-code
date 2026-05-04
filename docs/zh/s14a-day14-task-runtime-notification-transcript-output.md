# Day 14 输出物：durable task / runtime task / notification / transcript 四层边界

Day 14 最容易混的不是某个函数，而是四个看起来都像“任务进展”的对象其实在不同层：

> **durable task 管工作目标，runtime task 管执行槽位，notification 管异步回注，transcript 管可追溯记录。**

## 1. 一页结论

如果把这四层压成一页纸，最该记住的是这些：

- `durable task` 是“要做什么”的持久工作节点
- `runtime task` 是“现在谁在跑”的执行槽位
- `notification` 是“异步结果怎样回到主循环”的信封
- `transcript` 是“实际发生过什么”的会话/任务记录

它们会串在同一条链上，但不是同一个东西：

```text
durable task: 修复登录失败
  ->
runtime task: 后台运行 pytest / main-session query / agent
  ->
notification: <task-notification> 说明运行结束和 output file
  ->
transcript: 保存这次 query / task 的消息和工具过程
```

## 2. 四层边界表

| 层 | 它回答的问题 | 存放/流动位置 | 真实例子 |
| --- | --- | --- | --- |
| `durable task` | 还有什么工作目标、依赖谁、谁负责 | `.tasks/task_N.json` 或任务图管理器 | `agents/s_full.py` 里的 `TaskManager.create("Implement auth module")` 会写出一条带 `id / subject / status / blockedBy / blocks / owner` 的持久任务 |
| `runtime task` | 当前哪个执行单元正在跑、状态是什么、输出在哪 | `AppState.tasks[taskId]` | `LocalMainSessionTask.registerMainSessionTask(...)` 创建 `s...` 开头的 `local_agent(main-session)`，状态是 `running`，并带 `abortController / outputFile / isBackgrounded` |
| `notification` | 异步完成后，怎样让主循环重新知道这件事 | command queue / queued attachment / user-like message | `LocalMainSessionTask.enqueueMainSessionNotification(...)` 生成 `<task-notification>`，里面有 `task-id / output-file / status / summary`，再 `enqueuePendingNotification(...)` |
| `transcript` | 这次执行过程实际发生过哪些消息和工具事件 | session transcript / sidechain transcript | 后台主会话调用 `recordSidechainTranscript(messages, taskId)` 和 `recordSidechainTranscript([event], taskId, lastRecordedUuid)`，把后台 query 写到独立 transcript，避免污染前台会话 |

## 3. 为什么它们不能合并

### 3.1 `durable task` 不是正在跑的东西

一条 durable task 可以是：

```json
{
  "id": 12,
  "subject": "Implement auth module",
  "status": "pending",
  "blockedBy": [],
  "blocks": [13],
  "owner": "alice"
}
```

它最关心的是工作图：谁被谁阻塞、谁可以开始、谁负责推进。

真实例子：`agents/s_full.py` 的 `TaskManager.update(..., status="completed")` 会把后续任务的 `blockedBy` 里移除当前 task id。这个动作是在解锁工作目标，不是在停止某个进程。

### 3.2 `runtime task` 才是正在跑的执行槽位

后台主会话被切出去时，真实代码会创建一个 runtime task：

```text
taskId: s8k2m1qz
type: local_agent
agentType: main-session
status: running
isBackgrounded: true
abortController: ...
```

它回答的是“这个执行单元还活着吗、能不能停、输出写到哪里”。  
所以 `stopTask(...)` 查的是 `AppState.tasks[taskId]`，并要求 `status === "running"`，然后再调用对应 task 类型的 `kill()`。

### 3.3 `notification` 不是执行结果本体

notification 更像一张取件单：

```xml
<task-notification>
<task-id>s8k2m1qz</task-id>
<output-file>.task_outputs/s8k2m1qz.txt</output-file>
<status>completed</status>
<summary>Background session "..." completed</summary>
</task-notification>
```

它不会把完整输出直接塞进上下文。模型看到它以后，通常应该再读 `output-file`。

真实例子：`LocalMainSessionTask` 完成时先设置 `notified: true` 防重复，再把 XML 以 `mode: 'task-notification'` 放入队列。下一轮 query 会把它转成 queued attachment / user-like message。

### 3.4 `transcript` 是记录，不是调度指令

transcript 负责让系统能回看发生过什么：

- 用户说了什么
- assistant 说了什么
- tool_use 和 tool_result 怎样交替
- 后台 task 过程中产生了哪些事件

真实例子：后台主会话不能继续写主 session transcript，因为前台可能已经 `/clear` 并开始新对话。所以 `LocalMainSessionTask` 用 `getAgentTranscriptPath(asAgentId(taskId))` 和 `recordSidechainTranscript(...)` 写独立记录。

## 4. 一条完整链

```text
用户创建 durable task: "修复登录失败"
  ->
模型启动 runtime task: 后台 main-session / local_agent 继续调查
  ->
runtime task 写 sidechain transcript 和 output file
  ->
runtime task 完成后发 <task-notification>
  ->
REPL 下一轮把 notification 接回 messages
  ->
模型读取 output file，再决定 durable task 是否完成
```

这条链里每层都只做自己的事：

- durable task 不保存完整执行日志
- runtime task 不代表长期工作目标
- notification 不承载完整输出
- transcript 不决定下一步调度

## 5. 一句记忆钩子

> **目标看 durable task，现场看 runtime task，回来看 notification，追溯看 transcript。**
