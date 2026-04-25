# Day 12 学习指南：Runtime Task 与后台执行

> 这份讲义面向第一次把 `Task.ts`、`LocalShellTask`、`LocalAgentTask`、`RemoteAgentTask`、`InProcessTeammateTask` 串起来的人。  
> Day 12 最重要的不是背函数名，而是先把下面这句话讲顺：
>
> **Runtime Task 解决“现在谁在跑”，后台执行解决“主循环不用等它也能继续推进”。**

## 0. 这一天到底在学什么

到 Day 11 为止，你已经把 durable task graph 和工作目标层分开了。

Day 12 往前再走一步，开始回答另外一个问题：

> 当系统里真的有东西在跑时，它应该被怎样统一表示、跟踪、通知、回收？

所以这一天真正学的是这条链：

```text
工具或命令发起执行
  ->
创建 runtime task
  ->
登记到 AppState.tasks
  ->
执行体继续在后台推进
  ->
输出写入 output file / transcript
  ->
任务进入 completed / failed / killed
  ->
通过 task notification 回到主循环
  ->
被 UI / SDK / GC 回收
```

如果只记一句话，可以先记这个：

> **Runtime Task 是“执行槽位层”，不是 Day 11 的工作目标层。**

## 1. 这次重点看哪些代码

你给的重点代码基本正好覆盖了 Runtime Task 的四个核心分支：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:39-515`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:50-657`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:124-808`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:78-121`

为了把这些片段读顺，我建议再带上 3 个共享底座一起看：

- `/Users/hong.gao/python/src/claude-code-codex/src/Task.ts:44-125`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:12-46`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/task/framework.ts:48-117`

原因很简单：

- `Task.ts` 定义了所有 runtime task 的公共骨架
- `tasks/types.ts` 说明“哪些 task 算后台任务”
- `framework.ts` 说明不同 task 类型怎样接到同一棵状态树上

## 2. 先建立一张脑图

先不要急着分别读 3 个大文件。  
先把 Runtime Task 层想成一个统一外壳：

```text
统一 Task 外壳
  - id / type / status / description
  - outputFile / outputOffset
  - notified
  - startTime / endTime
  - kill(taskId)

不同执行体
  - LocalShellTask: 本地 shell 进程
  - LocalAgentTask: 本地 agent 生命周期
  - RemoteAgentTask: 远程 CCR session 的本地镜像
  - InProcessTeammateTask: 队友 / teammate 运行槽位
```

最值得先背下来的结论是：

> **Task 层做的是统一登记、统一状态、统一通知；真正执行工作的，不一定在同一个地方。**

这句话对 4 种 task 都成立：

- `LocalShellTask` 真正在跑的是 `ShellCommand`
- `LocalAgentTask` 真正在跑的是 `runAgent` 这条 async agent 生命周期
- `RemoteAgentTask` 真正在跑的是远程 session，本地只是在 poll
- `InProcessTeammateTask` 真正在跑的是 teammate runner / mailbox 协作流

## 3. 公共底座先看清

### 3.1 `Task.ts`：Runtime Task 的公共骨架

`/Users/hong.gao/python/src/claude-code-codex/src/Task.ts:44-125`

先看 `TaskStateBase`，它定义了所有 runtime task 都共享的字段：

- `id`
- `type`
- `status`
- `description`
- `toolUseId`
- `startTime`
- `endTime`
- `outputFile`
- `outputOffset`
- `notified`

这里最关键的不是字段本身，而是你要意识到：

> 这套字段描述的是“一个正在执行的运行单元”，不是“任务图里的工作目标”。

`Task.ts` 还定义了：

- `TaskType`
- `TaskStatus`
- `TaskContext`
- `Task.kill()`
- `createTaskStateBase()`
- `generateTaskId()`

所以你可以把它理解成：

> Runtime Task 层的最小协议。

任何具体 task，只要能回答下面几个问题，就能挂进来：

1. 它是什么类型
2. 它现在是什么状态
3. 它的输出写去哪
4. 它怎么被 kill

### 3.2 `tasks/types.ts`：哪些才算后台任务

`/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:12-46`

这个文件非常短，但很关键。

它做了两件事：

1. 把所有具体 task state 合成一个 `TaskState` union
2. 用 `isBackgroundTask()` 决定 UI 里哪些任务算后台任务

这里的判断规则很值得记住：

```text
status 必须是 running / pending
并且如果 task 有 isBackgrounded 字段，就不能是 false
```

这意味着：

- `LocalShellTask` 和 `LocalAgentTask` 支持“前台 -> 后台”切换，所以有 `isBackgrounded`
- `RemoteAgentTask` 没有这个字段，它天生就是后台语义
- `InProcessTeammateTask` 也没有这个字段，它默认就是一个后台运行槽位

这一步很重要，因为它说明：

> 后台任务不是看“名字像不像后台”，而是看状态层怎样建模。

### 3.3 `framework.ts`：统一登记与统一更新

`/Users/hong.gao/python/src/claude-code-codex/src/utils/task/framework.ts:48-117`

如果只挑两个函数记住，就是：

- `registerTask()`
- `updateTaskState()`

你可以把这两个函数理解成 Runtime Task 层的“总线接口”：

- 具体 task 创建自己那份 state
- 再用 `registerTask()` 放进 `AppState.tasks`
- 后续状态变化都通过 `updateTaskState()` 回写

这意味着不同 task 的差别主要不在“有没有统一管理”，而在：

1. 谁负责继续执行
2. 谁负责检测完成
3. 谁负责生成通知

## 4. 四类 Task 各自负责什么

### 4.1 `LocalShellTask`：本地进程型 runtime task

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:44-104`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:180-252`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:259-474`

先给一句最短定义：

> `LocalShellTask` 是对一个本地 shell 执行体的 task 化封装。

它的关键职责有 4 个：

1. 把 shell 命令注册成 runtime task
2. 管理前台 / 后台切换
3. 在完成时发 task notification
4. 处理“卡在交互输入上”的特殊提醒

#### 它怎么创建

`spawnShellTask()` 里最值得注意的点是：

- `taskId` 直接复用 `shellCommand.taskOutput.taskId`
- 注册 cleanup
- 构造 `LocalShellTaskState`
- `registerTask(taskState, setAppState)`
- 调用 `shellCommand.background(taskId)`

这说明它不是“先建 task，再随便找个地方写输出”，而是：

> **TaskOutput、taskId、磁盘输出路径从一开始就是绑死的。**

#### 它怎么从前台切到后台

这一段很值得讲，因为它体现了 Day 12 的设计味道。

`LocalShellTask` 不是只有一种创建方式：

- `spawnShellTask()`：一上来就后台
- `registerForeground()`：先作为前台登记
- `backgroundTask()`：把已登记的前台任务切到后台
- `backgroundExistingForegroundTask()`：避免重复注册，直接原地翻转 `isBackgrounded`

这里的设计重点是：

> **前台转后台时，系统尽量保留同一个 task，而不是新造一个 task。**

这样就能避免：

- 重复发 `task_started`
- 重复 cleanup
- 同一个命令在 UI 上出现两个任务

#### 它怎么判断完成

`LocalShellTask` 的完成信号非常直接：

> `shellCommand.result.then(...)`

也就是说，真正的执行者是 `ShellCommand`，而 `LocalShellTask` 只是订阅它的最终结果，然后把状态改成：

- `completed`
- `failed`
- `killed`

再顺手做几件事：

- `flushAndCleanup(shellCommand)`
- `enqueueShellNotification(...)`
- `evictTaskOutput(taskId)`

#### 它的额外亮点：stall watchdog

`startStallWatchdog()` 是这份代码里最值得专门看的细节之一。

它不是检测“命令慢不慢”，而是在检测：

> 输出停止增长以后，最后一段输出看起来像不像交互式 prompt。

如果像，就发一条 task notification 提醒模型：

- 这个后台命令可能在等人工输入
- 应该 kill 后改成非交互方式重跑

所以 `LocalShellTask` 不只是“跑 shell”，还承担了一部分：

> **把不可见的后台阻塞翻译成模型能理解的可操作通知。**

### 4.2 `LocalAgentTask`：本地 agent 生命周期型 runtime task

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:116-148`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:197-262`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:412-657`

一句话先定性：

> `LocalAgentTask` 不是包一个进程，而是包一个本地 agent 的完整生命周期。

这是它和 `LocalShellTask` 最大的不同。

#### 先看它的 state 形状

`LocalAgentTaskState` 里最值得你重点盯的字段是：

- `progress`
- `pendingMessages`
- `messages`
- `isBackgrounded`
- `retain`
- `diskLoaded`
- `evictAfter`

这几个字段说明它关心的不只是“有没有跑完”，还关心：

- agent 最近干了什么
- UI transcript 怎样保留
- 用户是否正在看这个 agent
- panel 里什么时候可以回收

所以你可以把它理解成：

> `LocalAgentTask` 比 `LocalShellTask` 更像一个有 UI 生命周期的执行槽位。

#### 它有两种启动模式

第一种是 `registerAsyncAgent()`：

> 一开始就按后台 agent 注册。

第二种是 `registerAgentForeground()`：

> 先作为前台 agent 运行，但随时可以转后台。

第二种模式里的关键设计是 `backgroundSignal`。

它会创建一个 promise，等未来某个时刻被 resolve。  
当 `backgroundAgentTask()` 被调用时：

- 它把 `isBackgrounded` 设为 `true`
- 然后 resolve 这个 signal

这意味着前台 agent 主循环可以用：

```text
Promise.race(
  nextMessagePromise,
  backgroundSignal
)
```

来判断自己是继续前台跑，还是交棒给后台生命周期。

#### 这份代码最值得学的点：前台 agent 如何优雅转后台

这一段不是在 `LocalAgentTask.tsx` 本身，而是在 `AgentTool.tsx` 的同步路径里真正发生：

- 前台 agent 一边读 `runAgent()` 的消息流
- 一边等 `backgroundSignal`
- 一旦 signal 触发，就停止前台那套 iterator
- 然后切换到 detached 的 async 生命周期继续跑

所以 `LocalAgentTask.tsx` 更像是：

> 给 `AgentTool` 提供 task state、progress、kill、background signal 这些基础设施。

#### 它怎么结束

终态入口主要有 3 个：

- `completeAgentTask()`
- `failAgentTask()`
- `killAsyncAgent()`

这里还要注意一个设计点：

> `LocalAgentTask` 自己负责状态落地，但最终通知很多时候是由 `AgentTool` 在更高层调用 `enqueueAgentNotification()` 统一发出的。

这说明 agent 任务比 shell 任务更“分层”：

- Task 文件管状态
- Tool 文件管执行编排和最终回传

#### 为什么它比 shell task 更复杂

因为 agent 不是一次性子进程，而是一个会不断产生新消息、新进度、新工具调用的执行体。

所以它需要额外处理：

- token/tool use 进度
- background summary
- pending user messages
- transcript retain / bootstrap
- panel grace period

一句话总结：

> `LocalShellTask` 管一个命令；`LocalAgentTask` 管一个活着的 agent 会话。

### 4.3 `RemoteAgentTask`：远程 session 的本地镜像

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:124-225`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:386-466`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:477-799`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:808-847`

一句话定义：

> `RemoteAgentTask` 本地并不执行 agent，它只是给远程 CCR session 建一份可恢复、可轮询、可通知的本地 task 镜像。

这是 Day 12 非常关键的一层，因为它说明：

> Runtime Task 不要求执行体一定在本地。

#### 它怎么创建

`registerRemoteAgentTask()` 做的事情非常清楚：

1. 生成 `taskId`
2. `initTaskOutput(taskId)`
3. 构造 `RemoteAgentTaskState`
4. `registerTask(...)`
5. 把身份信息写进 session sidecar
6. 启动 `startRemoteSessionPolling()`

你要特别注意这里的思路：

> 远程 session 一旦启动，本地马上建一个“影子 task”，负责承接 UI、恢复、通知和 output file。

#### 它真正的执行体在哪里

不在本地。

本地只是每秒 poll 一次远程 session：

- 拉新 event
- 把 delta 追加到 output file
- 更新 `log`
- 解析 `todoList`
- 解析 remote review progress
- 判断是否进入终态

所以 `RemoteAgentTask` 的本质不是 executor，而是：

> **poller + mirror + notifier。**

#### 这份代码最值得学的点：终态判断不是单一路径

`RemoteAgentTask` 的完成条件明显比 shell / local agent 复杂。

它可能来自：

- `sessionStatus === 'archived'`
- `result` event
- 自定义 `completionChecker`
- remote review 的 tag
- stable idle + output
- review timeout

这说明远程任务不能像本地进程那样只盯一个 promise。

它必须做的是：

> **把远端世界的多种结束信号，折叠成统一的本地 task 状态。**

#### 它为什么能跨恢复继续存在

`restoreRemoteAgentTasks()` 是这部分最值得专门看的函数之一。

它会：

- 扫 sidecar 里已持久化的 remote task metadata
- 向远程查询 session 当前状态
- 重建 `RemoteAgentTaskState`
- 重新启动 poll

所以这层设计回答的是一个很真实的问题：

> 终端重开后，后台远程任务怎么重新接回来？

答案不是“重新跑一遍”，而是：

> 从 sidecar 恢复本地镜像，再继续盯远程 session。

#### kill 为什么和其他 task 不一样

`RemoteAgentTask.kill()` 除了改本地状态，还会：

- 发 SDK terminated event
- `archiveRemoteSession(sessionId)`
- 删除 remote metadata

这说明 kill 远程任务不是单纯的 UI 动作，而是：

> 真正去结束远端资源占用。

### 4.4 `InProcessTeammateTask`：长寿命协作 worker 的 task 外壳

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:89-121`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:1-125`

一句话定义：

> `InProcessTeammateTask` 更像“长期存在的队友槽位”，不是一次性后台 job。

这份代码要特别注意两个点。

#### 第一，它是 teammate 语义，不是普通 async task 语义

从 `InProcessTeammateTask.tsx` 的注释就能看出来，它强调的是：

- team-aware identity
- plan mode approval flow
- idle / active 切换
- 同进程协作

所以它不像 `LocalShellTask` 那样“命令跑完就结束”，也不像 `LocalAgentTask` 那样“一个独立 subagent 会话”。

它更像：

> 一个可持续接收消息、持续协作的 teammate lane。

#### 第二，`TEAMMATE_MESSAGES_UI_CAP = 50` 很重要

`types.ts:89-121` 这段虽然很短，但信息量很大。

它说明：

- `task.messages` 只是 UI 镜像
- 完整历史不应该在 AppState 再复制一整份
- 否则并发 teammate 很多时，内存会爆

所以这里的设计重点不是“只保留 50 条很随意”，而是：

> **UI 展示层和真实执行层必须拆开，否则 runtime task 会变成内存炸弹。**

#### 这类 task 最值得记住的几个 helper

- `appendTeammateMessage()`
- `injectUserMessageToTeammate()`
- `findTeammateTaskByAgentId()`
- `getRunningTeammatesSorted()`

这几个函数反过来说明：

> `InProcessTeammateTask` 的重点不是启动方式，而是“如何在 task 层维持一个长期协作对象的可见状态”。 

## 5. 三条核心调用链

### 5.1 `BashTool -> LocalShellTask`

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tools/BashTool/BashTool.tsx:879-1015`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:180-474`

这条调用链可以简化成：

```text
BashTool.exec(...)
  ->
拿到 ShellCommand
  ->
如果明确 run_in_background
  -> spawnShellTask()

如果先前台运行太久
  -> registerForeground()
  -> backgroundExistingForegroundTask() / backgroundTask()

命令结束
  -> shellCommand.result.then(...)
  -> enqueueShellNotification(...)
```

这里最重要的理解点是：

> `BashTool` 负责决定“要不要后台化”，`LocalShellTask` 负责承接“后台化以后怎么登记、通知、回收”。

### 5.2 `AgentTool -> LocalAgentTask / RemoteAgentTask`

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:430-482`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:548-567`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:686-764`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:808-918`

这条链最值得讲，因为它把 3 条分支都收拢到一起了。

#### 分支 1：remote isolation

```text
AgentTool
  ->
effectiveIsolation === remote
  ->
teleportToRemote(...)
  ->
registerRemoteAgentTask(...)
  ->
返回 remote_launched
```

这里创建的不是本地执行体，而是远程 session 的本地镜像。

#### 分支 2：本地 async agent

```text
AgentTool
  ->
shouldRunAsync === true
  ->
registerAsyncAgent(...)
  ->
detached runAsyncAgentLifecycle(...)
  ->
LocalAgentTask 持续更新 progress / summary / terminal state
```

#### 分支 3：本地 sync agent，之后再后台化

```text
AgentTool
  ->
registerAgentForeground(...)
  ->
runAgent() 前台流式执行
  ->
Promise.race(nextMessage, backgroundSignal)
  ->
收到 backgroundSignal
  ->
切换到 detached background lifecycle
```

这条路径是 Day 12 最值得啃的一段，因为它展示了：

> 一个 agent 不需要“一开始就决定同步还是异步”，它可以先前台服务用户，再无缝切到后台继续跑。

### 5.3 `teammate runtime -> InProcessTeammateTask`

重点代码：

- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:24-125`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:89-121`

这条链和前两条不太一样。

它不是“启动一个任务，等它结束”，而更像：

```text
teammate 被创建
  ->
task 层记录身份、消息镜像、pendingUserMessages
  ->
运行中可不断注入新消息
  ->
idle / active 来回切换
  ->
必要时 shutdown / kill
```

所以队友任务更像：

> 一个长期挂在系统里的协作端口。

## 6. 后台执行到底是怎么成立的

如果你把上面 4 类 task 读完后还觉得乱，直接回到这张对照表。

| Task 类型 | 真正执行者 | 本地 Task 层负责什么 | 完成信号来自哪里 |
| --- | --- | --- | --- |
| `LocalShellTask` | `ShellCommand` / 本地进程 | 注册、前后台切换、stall 检测、通知、回收 | `shellCommand.result` |
| `LocalAgentTask` | `runAgent` 生命周期 | progress、retain、pendingMessages、panel 生命周期 | lifecycle callback / background signal / AgentTool 回调 |
| `RemoteAgentTask` | 远程 CCR session | 本地镜像、poll、restore、通知、远程归档 | poll 到的 status/result/tag/checker |
| `InProcessTeammateTask` | teammate runner / mailbox | 身份、消息镜像、UI cap、消息注入、排序 | teammate runtime 改状态或 kill |

这一页最应该记住的结论是：

> **Runtime Task 层是统一外壳，不同 task 只是在“执行者是谁、完成信号从哪来”上不同。**

## 7. 为什么这是“后台执行计划”，而不是零散补丁

从这些代码里可以看到，它不是几个功能各自做异步，而是有一套统一套路：

### 第一步：先建 runtime task，而不是直接等待

无论是 shell、agent 还是 remote session，只要要走后台路径，系统都会先：

- 生成 taskId
- 写入 `AppState.tasks`
- 准备 output file / transcript

### 第二步：真正执行体在别处继续跑

- shell 在本地进程里跑
- local agent 在 detached async lifecycle 里跑
- remote agent 在云端 session 里跑
- teammate 在自己的 runner/mailbox 里跑

### 第三步：结果不直接塞回 prompt，而是走 notification

这一步在三个主 task 里都很明显：

- `enqueueShellNotification()`
- `enqueueAgentNotification()`
- `enqueueRemoteNotification()`

这说明后台执行的核心思想是：

> **执行现场和主循环上下文解耦；结果通过通知回流，而不是靠主循环原地傻等。**

### 第四步：终态后尽快回收

这些 task 都会在终态后做类似的事情：

- 标记 `notified`
- `evictTaskOutput(...)`
- 删除 sidecar 或 cleanup
- 等待 UI / GC 进一步回收

所以 Day 12 的“计划”不是某个单独 planner，而是一套统一执行协议：

```text
创建
  ->
登记
  ->
后台推进
  ->
通知
  ->
回收
```

## 8. 读源码时应该抓住哪几个问题

如果你准备二刷这些代码，建议每个文件都只问 4 个问题：

### 对 `LocalShellTask` 问

1. 谁创建 task？
2. shell 真正在哪里继续跑？
3. 谁来判断完成？
4. 卡在交互 prompt 时系统怎么知道？

### 对 `LocalAgentTask` 问

1. 为什么它要比 shell task 多这么多 UI 字段？
2. 前台 agent 怎么切后台？
3. `backgroundSignal` 到底在解决什么问题？
4. 为什么通知有一部分在 `AgentTool` 层发？

### 对 `RemoteAgentTask` 问

1. 本地到底有没有真正执行 agent？
2. 为什么它必须持久化 metadata？
3. 为什么完成条件不能只看一个 result？
4. 为什么 kill 要真的去 archive 远程 session？

### 对 `InProcessTeammateTask` 问

1. 它为什么更像“长期协作者”而不是一次性任务？
2. 为什么 `messages` 只能保留 UI cap？
3. 为什么它允许在非终态时继续注入消息？
4. teammate task 和普通 local agent task 的根本区别是什么？

## 9. 推荐的真实阅读顺序

如果你现在要自己重新走一遍源码，我建议按这个顺序：

1. `Task.ts`
   先确认 runtime task 的公共骨架
2. `tasks/types.ts`
   先确认后台任务筛选逻辑
3. `utils/task/framework.ts`
   先确认统一登记和统一更新机制
4. `LocalShellTask.tsx`
   这是最像“经典后台任务”的一类，最好入门
5. `LocalAgentTask.tsx`
   看清 agent 生命周期怎样 task 化
6. `AgentTool.tsx`
   看 foreground -> background 的真实切换现场
7. `RemoteAgentTask.tsx`
   再理解“本地镜像远程执行体”这层抽象
8. `InProcessTeammateTask/types.ts`
   最后看 teammate 这类长寿命协作槽位

这个顺序比“按文件名顺序硬读”更好，因为它是：

> 从统一抽象，走到最简单执行体，再走到最复杂执行体。

## 10. 最后一页速记

如果最后你只能带走 5 句话，我希望是这 5 句：

1. `Task.ts` 里的 task 是 runtime task 公共骨架，不是 Day 11 的 durable task graph。
2. `LocalShellTask` 是进程包装器，关键是前后台切换、stall watchdog 和 completion notification。
3. `LocalAgentTask` 是本地 agent 生命周期包装器，关键是 progress、retain、pendingMessages 和 background signal。
4. `RemoteAgentTask` 是远程 session 的本地镜像，关键是 poll、restore 和多信号终态判断。
5. `InProcessTeammateTask` 是长期协作槽位，重点不是“一次跑完”，而是“持续接收消息并保持可见状态”。

最短总收尾：

> **Day 11 管“要做什么”，Day 12 管“现在谁在执行，以及执行完怎么回来说一声”。**
