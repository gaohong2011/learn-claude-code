# Day 16 学习指南：Agent Teams 与消息总线

> Day 13 学的是“未来时间怎样重新进入主循环”。  
> Day 16 学的是“长期队友怎样通过消息边界继续协作”。
>
> 这一天最重要的不是“多开几个模型调用”，而是先把下面这句话讲顺：
>
> **teammate 是长期存在的 actor；mailbox 是 actor 之间传递新输入的边界；runtime task 只是当前活着的执行槽位。**

## 0. 这一天到底在学什么

到 Day 15 为止，你已经能把工作拆成三层：

- durable task：要做什么
- runtime task：现在谁在跑
- schedule / notification：未来或异步事件怎样回到主循环

Day 16 开始进入第 4 周的平台层，回答另一个问题：

> **如果系统里不止一个 agent，而且某些 agent 要长期留下来继续接活，应该怎么组织？**

所以这一天真正要学的是这条链：

```text
leader 创建 team
  ->
team roster 记录长期成员身份
  ->
leader spawn teammate
  ->
teammate 拥有独立 runtime loop
  ->
leader / teammate 通过 mailbox 发消息
  ->
recipient 在下一轮先读取 mailbox
  ->
新消息进入 recipient 自己的 messages
  ->
recipient 继续工作或回到 idle
```

如果只记一句话，先记这个：

> **Agent Teams 解决的是“长期协作身份”，不是“一次性子任务委派”。**

## 1. 这次重点看哪些代码

学习计划里的 Day 16 已经把阅读范围列得很具体，建议按下面顺序读。

### 必读

- `docs/zh/s15-agent-teams.md`
- `docs/zh/team-task-lane-model.md`
- `agents/s15_agent_teams.py`
- `/Users/hong.gao/python/src/claude-code-codex/docs/04-multi-agent-coordinator.md`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/TeamCreateTool/TeamCreateTool.ts:74-220`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/shared/spawnMultiAgent.ts:72-294,760-1093`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:149-266,520-760`
- `/Users/hong.gao/python/src/claude-code-codex/src/context/mailbox.tsx:8-37`

### 选读

- `/Users/hong.gao/python/src/claude-code-codex/src/utils/teammateMailbox.ts:1-190`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:680-790`
- `/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:1-121`

在教学版 `agents/s15_agent_teams.py` 里，优先盯这几个入口：

- `MessageBus.send(...)`
- `MessageBus.read_inbox(...)`
- `MessageBus.broadcast(...)`
- `TeammateManager.spawn(...)`
- `TeammateManager._teammate_loop(...)`
- `TOOL_HANDLERS`
- `agent_loop(...)`

原因很直接：

- `MessageBus` 说明消息怎样进入每个 teammate 的 inbox
- `TeammateManager` 说明长期成员身份怎样被注册和恢复
- `_teammate_loop(...)` 说明 teammate 怎样带着自己的 `messages` 继续跑
- `agent_loop(...)` 说明 leader 也有自己的 inbox，不是只有 worker 收消息

## 2. 先建立最小心智模型

先别急着看 tmux、iTerm2、in-process backend。

先把 Day 16 想成下面这 5 个盒子：

```text
1. leader
2. team roster
3. teammate identity
4. mailbox / message bus
5. teammate runtime loop
```

它们的分工分别是：

- `leader`：创建 team、spawn teammate、分派消息、接收回报
- `team roster`：记住 team 里有谁、角色是什么、当前是什么状态
- `teammate identity`：一个长期可点名的 actor，不等于一次性 subagent
- `mailbox`：每个 actor 的收件箱，也是消息总线的最小实现
- `teammate runtime loop`：teammate 当前活着的执行槽位

这里最值得先背下来的结论是：

> **message bus 不是共享一根 `messages[]`，而是按 recipient 分开的 mailbox boundary。**

也就是说，leader 给 alice 发消息时，不应该把内容直接塞进全局上下文。  
正确模型是：

```text
leader
  ->
write alice inbox
  ->
alice poll / drain inbox
  ->
alice appends message into alice's own messages
```

## 3. Day 16 的主线到底怎么跑

### 3.1 先创建 team，而不是直接散开模型调用

参考实现里的 `TeamCreateTool` 会先检查 leader 是否已经在一个 team 里。  
一个 leader 当前只能管理一个 team。

创建成功后，它会写入 team file，并把 leader 自己也注册成 team member：

```text
team file
  name
  description
  leadAgentId
  leadSessionId
  members: [leader]
```

同时，AppState 里会出现 `teamContext`，后续 `spawn / send / broadcast` 都靠这个上下文知道自己属于哪个 team。

教学版里这层被压缩成：

```text
.team/config.json
  {"team_name": "default", "members": [...]}
```

这里最重要的边界是：

> **team 是协作空间，teammate 是这个空间里的长期成员。**

### 3.2 spawn teammate 时同时产生身份和执行槽位

教学版 `TeammateManager.spawn(...)` 做 4 件事：

1. 查找或创建 `member`
2. 把 `member.status` 置为 `working`
3. 保存 `.team/config.json`
4. 启动一个 thread 跑 `_teammate_loop(...)`

参考实现更复杂，但结构相同：

```text
spawnTeammate(...)
  ->
resolve model / agent type
  ->
generate agent id and color
  ->
start pane or in-process runner
  ->
register runtime task
  ->
append member to team file
  ->
deliver initial prompt
```

这里要把两层分开：

- `team member` 是长期身份
- `runtime task` 是当前执行槽位

同一个 teammate 可以 idle、继续被唤醒、被 shutdown。  
runtime task 描述的是它当前如何活着、能不能被中止、UI 里怎么显示。

### 3.3 message bus 的最小实现就是 per-teammate mailbox

教学版用 JSONL 文件：

```text
.team/inbox/alice.jsonl
.team/inbox/bob.jsonl
.team/inbox/lead.jsonl
```

发送消息时追加一条 envelope：

```python
{
    "type": "message",
    "from": "lead",
    "content": "Please review auth module.",
    "timestamp": 1710000000.0,
}
```

读取消息时：

1. 打开自己的 inbox
2. 逐行 parse JSON
3. 读完后清空文件

参考实现里的 mailbox 放在 team 目录下：

```text
.claude/teams/{team_name}/inboxes/{agent_name}.json
```

它还会做几件更产品化的事：

- inbox 以 agent name 和 team name 作为路由键
- 写入时加 lock，避免多个 agent 并发写坏文件
- message 带 `read` 字段，UI 可以区分已读和未读
- 发送时可带 `summary` 和 `color`，方便团队 UI 展示

所以 Day 16 的消息总线不是抽象概念，它至少要回答：

- 消息写到谁的 inbox
- 谁有权读取
- 读完后是否 drain / mark read
- sender / timestamp / type 怎样保留

### 3.4 teammate 每轮先看 inbox，再继续自己的 loop

教学版 `_teammate_loop(...)` 的关键动作是：

```python
inbox = BUS.read_inbox(name)
for msg in inbox:
    messages.append({"role": "user", "content": json.dumps(msg)})
```

这说明新消息不是重建 teammate，也不是污染 leader 的上下文。  
它只是进入这个 teammate 自己的消息历史。

参考实现里的 in-process runner 会在 idle 状态轮询 mailbox：

```text
waitForNextPromptOrShutdown(...)
  ->
readMailbox(identity.agentName, identity.teamName)
  ->
优先处理 shutdown request
  ->
再处理普通新消息
  ->
把消息作为下一轮 prompt 交给 teammate
```

这就是“长期队友”的关键：

> teammate 可以工作、idle、再被 mailbox 唤醒，而不是每次都从零创建。

### 3.5 SendMessage 只是通信入口，不是完整协议层

`SendMessageTool` 负责把内容写到 recipient inbox：

```text
SendMessage(to=alice, message=...)
  ->
validate recipient and message shape
  ->
writeToMailbox(alice, envelope, teamName)
  ->
return routing metadata
```

它也已经支持一些结构化 message type，例如 shutdown 和 plan approval。  
但 Day 16 不应该提前把重点放到 request lifecycle。

这一天最该守住的边界是：

> **mailbox message 是 actor 间通信记录；带 `request_id` 的协议对象是 Day 17 的重点。**

## 4. team member、mailbox、runtime task、leader 怎么分

Day 16 的产出项要求明确四者关系。  
可以直接用这张表做判断：

| 对象 | 它回答的问题 | 典型字段或位置 | 不要把它误认为 |
| --- | --- | --- | --- |
| `leader` | 谁在创建团队、分派工作、接收回报 | `leadAgentId`、`leadSessionId`、lead inbox | 不是所有 teammate 的共享上下文 |
| `team member` | 这个长期 actor 是谁 | `name`、`role / agentType`、`status`、`agentId` | 不是 task，也不是一次 model call |
| `mailbox` | 新消息怎样跨 actor 投递 | `.team/inbox/*.jsonl` 或 `.claude/teams/.../inboxes/*.json` | 不是共享 `messages[]` |
| `runtime task` | 当前执行槽位怎样被追踪和停止 | `InProcessTeammateTaskState`、`abortController`、`isIdle` | 不是长期身份本身 |

最短判断法：

```text
leader     = 协调者
member     = 谁
mailbox    = 怎么把话送到谁那里
runtime    = 现在这个谁是否活着、怎么跑
```

## 5. 参考实现里最值得看的产品化细节

### 5.1 `TeamCreateTool.ts`

最值得看的是三类约束：

- 一个 leader 只能有一个当前 team
- team file 和 AppState teamContext 都要写
- team 同时也是 task list 的边界，创建 team 会重置对应 task list

这说明真实系统里 team 不是 UI 装饰，而是协作、任务编号和成员发现的共同边界。

### 5.2 `spawnMultiAgent.ts`

这部分最值得看的不是 pane 创建细节，而是 spawn 同时做了两件事：

- 创建或注册 teammate identity
- 创建或注册 teammate runtime slot

pane 模式下，初始 prompt 通过 mailbox 送给 teammate。  
in-process 模式下，初始 prompt 直接交给 runner，mailbox 主要用于后续消息和 pane teammate。

这说明实现可以不同，但抽象边界不变：

> teammate identity 要可寻址，runtime loop 要能接收下一条消息。

### 5.3 `SendMessageTool.ts` 和 `teammateMailbox.ts`

`SendMessageTool` 做输入校验和路由，`writeToMailbox(...)` 做持久写入。

这里值得注意的是：

- `to` 是裸 teammate name，不是随便写一个 `name@team`
- broadcast 会遍历 team file members
- cross-session bridge 需要额外 permission check
- 文件 mailbox 需要 lock，避免并发写入破坏消息数组

这说明 message bus 的难点不是“写一行 JSON”，而是：

> 多个 actor 同时写、不同 backend 同时读、UI 还要能展示未读状态。

### 5.4 `inProcessRunner.ts`

这里最值得看的，是 teammate idle 后不是退出，而是等待下一条 prompt 或 shutdown：

- 先看 in-memory pending messages
- 再轮询 mailbox
- shutdown request 优先级高于普通消息
- abort signal 可以终止整个 teammate

这就是长期 actor 和一次性 subagent 的实际差别。

## 6. 初学者最容易犯的错

### 6.1 把 teammate 当成“有名字的 subagent”

如果生命周期还是：

```text
spawn -> work -> summary -> destroyed
```

那它本质上还是 subagent。

teammate 的生命周期应该更像：

```text
spawn -> work -> idle -> receive message -> work -> idle -> shutdown
```

### 6.2 所有人共用一份 `messages`

这会把 leader、alice、bob 的上下文混在一起。  
Day 16 的关键正是每个 teammate 都有自己的 `messages` 和 inbox。

### 6.3 把 mailbox 当成协议层本身

mailbox 只保证“消息能送达并被读取”。  
它不天然保证请求审批、超时、拒绝、恢复。

这些要到 Day 17 的 protocol request layer 才展开。

### 6.4 把 runtime task 当成 teammate 本身

runtime task 是正在跑的执行槽位。  
teammate 是长期身份。

一个 teammate 的当前执行槽位可以停止，但 team roster 里仍可能保留这个 member 的身份和历史。

### 6.5 忽略 leader 也有 inbox

团队通信不是单向的。  
worker 给 leader 发回结果、permission request、shutdown response，也应该通过 leader 的消息入口回到 leader。

## 7. 今天的输出物该怎么写

学习计划给 Day 16 的产出要求是：

> 画出 teammate 生命周期图，明确 team member、mailbox、runtime task、leader 之间的关系。

我建议把输出物压成两部分。

### 第一部分：一条生命周期链

```text
leader creates team
  ->
spawn teammate
  ->
team member recorded in roster
  ->
runtime loop starts
  ->
leader writes message to teammate mailbox
  ->
teammate drains mailbox before next turn
  ->
message enters teammate's own messages
  ->
teammate works and reports back
  ->
teammate becomes idle or receives shutdown
```

### 第二部分：一句硬边界

```text
team member 管“谁长期存在”
mailbox 管“消息怎样送达”
runtime task 管“当前谁正在跑”
leader 管“谁在协调和回收结果”
```

只要把这两部分讲顺，Day 16 就算学对了。

## 8. 读完 Day 16 应该能自己说清的话

如果这一天真的吃透了，你应该能不用看文档，自己说清下面这几句：

- teammate 不是一次性 subagent
- team roster 记录长期 actor 身份
- mailbox 是 per-recipient 的消息边界，不是共享上下文
- teammate 的新任务来自 inbox，而不是重新创建 teammate
- runtime task 表示当前执行槽位，不等于 teammate 身份
- Day 16 只建立团队和消息边界，Day 17 才把消息升级成协议请求

## 9. 最低验证

```sh
cd learn-claude-code
python agents/s15_agent_teams.py
```

这个脚本会启动交互式 agent loop，并依赖 `MODEL_ID` 和可用的 Anthropic API 配置。  
如果当前环境不适合真实调用模型，可以先做语法级验证：

```sh
python -m py_compile agents/s15_agent_teams.py
```

真实运行时重点观察三件事：

1. `spawn_teammate` 后 `.team/config.json` 是否出现 member
2. `send_message` 后对应 `.team/inbox/{name}.jsonl` 是否出现消息
3. teammate 读完 inbox 后是否进入自己的 loop，而不是污染 leader 的 `messages`

最后把这句话记住就够了：

> **Day 16 的关键词是“长期 actor + mailbox boundary”。**
