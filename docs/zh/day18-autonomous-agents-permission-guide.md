# Day 18 学习指南：Autonomous Agents 与权限分层

> Day 17 学的是“团队请求怎样变成可追踪协议”。
> Day 18 学的是“空闲 teammate 怎样在规则内自己找活，并在权限边界内继续推进”。
>
> 这一天最重要的不是让 agent 永远运行，而是先把下面这句话讲顺：
>
> **自治不是乱跑；自治是 idle teammate 按 claim policy 找 ready task，启动执行槽位，遇到高风险动作时把权限请求桥接回 leader。**

## 0. 这一天到底在学什么

到 Day 17 为止，团队已经有：

- 长期 teammate
- mailbox message bus
- request_id 协议
- task board
- coordinator 编排纪律

但如果每条任务仍然要 leader 手动点名，team 规模一大，leader 就会变成瓶颈。

Day 18 要回答的是：

> **当队友空闲时，能不能自己找下一条合适任务，并在权限安全边界里继续工作？**

所以这一天真正要学的是这条链：

```text
teammate enters idle
  ->
先检查 inbox
  ->
没有显式消息时扫描 task board
  ->
按 status / owner / blockedBy / role 判断 claimable
  ->
原子认领 task
  ->
启动或恢复 worker runtime loop
  ->
工具调用遇到权限边界
  ->
worker permission request 经 mailbox 桥接给 leader
  ->
leader approve / reject
  ->
worker 继续、取消或进入 idle
  ->
结果 / idle notification 回到 leader
```

如果只记一句话，先记这个：

> **Day 18 的自治链是“任务认领 -> worker 启动 -> 权限桥接 -> 结果回收 -> 恢复续行”。**

## 1. 这次重点看哪些代码

### 必读

- `docs/zh/s17-autonomous-agents.md`
- `docs/zh/team-task-lane-model.md`
- `agents/s17_autonomous_agents.py`
- `/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:80-194`
- `/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/coordinatorHandler.ts:26-65`
- `/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40-159`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:519-689,883-1040,1544-1552`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/spawnInProcess.ts:104-227`

在教学版 `agents/s17_autonomous_agents.py` 里，优先盯这几个入口：

- `is_claimable_task(...)`
- `scan_unclaimed_tasks(...)`
- `claim_task(...)`
- `_append_claim_event(...)`
- `ensure_identity_context(...)`
- `TeammateManager._loop(...)`
- `idle` teammate tool

原因很直接：

- `is_claimable_task(...)` 定义什么任务能被自治认领
- `claim_task(...)` 是防重复抢任务的原子边界
- `claim_events.jsonl` 让自治行为可观察
- `ensure_identity_context(...)` 说明恢复续行时要先补身份
- `_loop(...)` 把 `WORK -> IDLE -> WORK` 的生命周期串起来

## 2. 先建立最小心智模型

先把 Day 18 想成下面这 5 个盒子：

```text
1. idle loop
2. claim policy
3. runtime worker state
4. permission bridge
5. result / idle notification
```

它们的分工分别是：

- `idle loop`：空闲 teammate 先看 inbox，再看 task board
- `claim policy`：判断这条 task 当前能不能由这个角色认领
- `runtime worker state`：记录 worker 是否 running / idle / shutdown
- `permission bridge`：worker 遇到风险操作时，把请求发给 leader
- `result / idle notification`：worker 完成、阻塞、失败或空闲时通知 leader

这里最值得先背下来的结论是：

> **自治执行者仍然是长期 teammate，不是一次性 subagent。**

它有自己的身份、inbox、runtime loop 和权限上下文。
它只是多了一种能力：空闲时可以按规则主动找活。

## 3. Day 18 的主线到底怎么跑

### 3.1 idle 不是消失，而是等待下一份输入

教学版 teammate 生命周期是：

```text
WORK
  ->
IDLE
  +-- inbox has message -> WORK
  +-- ready task exists -> claim -> WORK
  +-- timeout -> shutdown
```

所以 idle 阶段不是“agent 结束了”。
它只是：

> 当前没有活，但这个 actor 仍然在线，等待下一条消息或下一份可认领任务。

### 3.2 claim policy 至少要看 4 件事

教学版的 `is_claimable_task(...)` 很适合作为最小规则：

```python
task.status == "pending"
and not task.owner
and not task.blockedBy
and role matches claim_role / required_role
```

这说明自治不是“谁空谁拿”。

一条 task 要被自动认领，至少要满足：

- 还没开始
- 没有 owner
- 没有未完成阻塞
- 当前 teammate 的 role 符合要求

如果只看 `pending`，就会让 worker 抢走不该做的任务。

### 3.3 认领必须是原子动作

两个 teammate 可能同时看到同一条 ready task。

所以 `claim_task(...)` 要在锁里做：

```text
load task
  ->
重新检查 claimable
  ->
写 owner / status / claimed_at / claim_source
  ->
save task
```

这一步要么完整成功，要么失败。

认领后还要写一条事件：

```text
.tasks/claim_events.jsonl
  event=task.claimed
  task_id
  owner
  role
  source=auto/manual
  ts
```

只看 task 当前 owner 不够。
事件日志说明自治系统什么时候做了什么。

### 3.4 worker 启动后要有独立 runtime state

参考实现里的 `spawnInProcessTeammate(...)` 会创建：

- `agentId`
- `taskId`
- 独立 `AbortController`
- `TeammateIdentity`
- `TeammateContext`
- `InProcessTeammateTaskState`

关键点是：

> teammate 的工作不只是一个函数调用，而是一个可追踪、可中止、可展示的 runtime task。

`InProcessTeammateTaskState` 里会记录：

- `identity`
- `prompt`
- `permissionMode`
- `isIdle`
- `shutdownRequested`
- `pendingUserMessages`
- recent `messages`

这让 leader 和 UI 能知道 worker 当前到底是 running、idle 还是等待权限。

### 3.5 权限桥接把风险操作交还给 leader

自治不等于绕过权限系统。

参考实现里的 `swarmWorkerHandler` 很关键：

```text
worker wants risky tool use
  ->
try classifier if available
  ->
create permission request
  ->
register callback before sending
  ->
send request to leader via mailbox
  ->
worker waits
  ->
leader approves or rejects
  ->
callback resolves
  ->
worker continues or aborts
```

这一步有两个重要细节：

- callback 要先注册，再发请求，避免 leader 很快响应导致竞态
- abort signal 要能取消等待，避免 worker 永远卡住

所以 Day 18 的权限分层可以这样记：

```text
worker 可以自治找活
但高风险 tool permission 仍然回到 leader / user control plane
```

### 3.6 结果回收不是普通聊天

参考实现里 worker 完成、空闲或失败后，会向 leader mailbox 写通知。

`inProcessRunner.ts` 里有两类值得看：

- `sendMessageToLeader(...)`
- `sendIdleNotification(...)`

这说明 worker 状态变化不是“静悄悄发生”。
它要回到 leader，让 leader 知道：

- worker 当前可用
- 任务完成、阻塞或失败
- 是否可以继续分配或等待新的 task claim

coordinator mode 还会把 worker result 作为 `<task-notification>` user-role message 注入 coordinator 回合。
它看起来像 user message，但语义上是内部执行结果。

### 3.7 恢复续行要补身份

教学版的 `ensure_identity_context(...)` 很重要。

当 teammate 从 idle 恢复，或经过 compact 后继续运行，它可能忘掉：

- 我是谁
- 我的 role 是什么
- 我属于哪个 team

所以恢复前要补：

```text
<identity>You are 'alice', role: frontend, team: default. Continue your work.</identity>
```

这不是装饰。
它是长期 teammate 能稳定续行的身份锚点。

## 4. 自治协作图

Day 18 的产出项要求画出：

```text
任务认领 -> worker 启动 -> 权限桥接 -> 结果回收 -> 恢复续行
```

完整链路可以这样画：

```text
idle teammate
  ->
scan task board
  ->
claim ready task atomically
  ->
write claim event
  ->
resume / start worker runtime loop
  ->
worker calls tools
  ->
permission needed?
  -> no  -> continue work
  -> yes -> send permission request to leader mailbox
             ->
             leader approve / reject
             ->
             worker continue / abort
  ->
worker sends result or idle notification to leader
  ->
leader sees result
  ->
teammate returns to idle
  ->
check inbox or claim next task
```

一句话压缩：

> **自治是 idle teammate 自己接下一份安全任务；权限和结果仍然回到 leader 控制面。**

## 5. team / task / runtime / permission 的边界

| 层 | 它回答的问题 | Day 18 中的表现 |
| --- | --- | --- |
| `teammate` | 谁长期在线 | alice idle 后继续扫描 inbox / task |
| `task` | 要做什么 | `.tasks/task_*.json` 里的 pending work |
| `claim policy` | 谁能安全接 | status、owner、blockedBy、role |
| `runtime task` | 当前谁在跑 | `InProcessTeammateTaskState` |
| `permission` | 风险动作能不能执行 | worker request 桥接给 leader |
| `notification` | 结果怎样回来 | mailbox / `<task-notification>` |

最容易混的是 task 和 runtime task：

- task 是工作目标
- runtime task 是当前执行槽位

自治认领的是 task。
启动或恢复的是 runtime loop。

## 6. 参考实现里最值得看的产品化细节

### 6.1 `inProcessRunner.ts`

这里最值得看 3 个点：

- `tryClaimNextTask(...)`：worker 启动或 idle 后尝试从 task list claim
- `sendIdleNotification(...)`：worker 可用、失败或被中断时通知 leader
- `runInProcessTeammate(...)`：把 teammate identity、system prompt、tool set 和消息循环串起来

它说明真实系统里的自治不是一个孤立函数，而是 runner 生命周期的一部分。

### 6.2 `spawnInProcess.ts`

这里最值得看的是 runtime state 初始化：

- 独立 abort controller
- identity 写入 task state
- permission mode 按 planModeRequired 设置
- `isIdle`、`shutdownRequested`、`pendingUserMessages` 初始化

这让 worker 后续可以被停止、被展示、被恢复。

### 6.3 `swarmWorkerHandler.ts`

这里最值得看的，是 worker 权限不直接弹给自己处理，而是通过 mailbox 请求 leader。

它还处理了两个并发细节：

- 先注册 callback，再发送 permission request
- abort 时清理 pending request 并取消工具调用

这就是“自治但不越权”的关键实现。

### 6.4 `coordinatorHandler.ts`

coordinator permission handler 先尝试 hooks，再尝试 classifier，最后才落到人工决策。

这说明 coordinator 侧不是一律手工阻塞，而是仍然复用已有 permission control plane。

## 7. 初学者最容易犯的错

### 7.1 把自治理解成无限循环

真正要学的是 idle 阶段的安全检查，不是让 worker 永远占资源。

### 7.2 只按 `pending` 认领任务

必须同时看 owner、blockedBy 和 role policy。

### 7.3 认领没有锁

没有原子认领，两个 teammate 会抢到同一条任务。

### 7.4 worker 权限自己说了算

自治 worker 可以找活，但不能绕过 leader / permission control plane。

### 7.5 结果不回收

如果 worker 完成后不通知 leader，leader 只能猜系统状态。

### 7.6 恢复时不补身份

长期 teammate 经过 compact 或 idle resume 后，要重新注入身份锚点。

## 8. 今天的输出物该怎么写

学习计划给 Day 18 的产出要求是：

> 画出 “任务认领 -> worker 启动 -> 权限桥接 -> 结果回收 -> 恢复续行” 的自治协作图。

建议输出物压成两部分：

### 第一部分：主链路图

```text
idle -> scan -> claim -> runtime -> permission bridge -> result -> idle/resume
```

### 第二部分：一句硬边界

```text
自治只改变“谁来发现下一份工作”，不改变“权限必须回到控制面”的规则。
```

## 9. 读完 Day 18 应该能自己说清的话

如果这一天真的吃透了，你应该能不用看文档，自己说清下面这几句：

- idle teammate 没有消失，它在等待 inbox 或可认领 task
- claimable task 至少要满足 pending、无 owner、无 blockedBy、role 匹配
- task claim 必须是原子动作
- worker 启动后要注册 runtime state，不只是跑一个函数
- swarm worker 的高风险权限请求要通过 mailbox 桥接给 leader
- 结果和 idle 状态要回到 leader，后续才能恢复续行

## 10. 最低验证

```sh
cd learn-claude-code
python agents/s17_autonomous_agents.py
```

这个脚本会启动交互式 agent loop，并依赖 `MODEL_ID` 和可用的 Anthropic API 配置。
如果当前环境不适合真实调用模型，可以先做语法级验证：

```sh
python -m py_compile agents/s17_autonomous_agents.py
```

真实运行时重点观察三件事：

1. ready task 是否会被 idle teammate 自动认领
2. blocked task 或 role 不匹配 task 是否不会被误认领
3. `.tasks/claim_events.jsonl` 是否记录了 `task.claimed` 事件

最后把这句话记住就够了：

> **Day 18 的关键词是“自治认领，但权限不自治”。**
