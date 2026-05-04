# Day 16 输出物：`teammate lifecycle -> mailbox -> runtime loop`

Day 16 最该压成一页纸的，不是 tmux / iTerm2 / in-process 后端差异，而是下面这条主线：

> **teammate 是长期存在的 actor；mailbox 是 actor 间消息总线的最小边界；runtime task 只表示当前这个 actor 的执行槽位。**

## 1. 一页结论

如果把 Day 16 压成 5 句，最该记住的是这些：

- `teammate` 不是一次性 subagent，而是有名字、角色、inbox 和生命周期的长期成员
- `team roster` 记录 team 里有谁，不记录“当前这件工作本身”
- `mailbox` 负责把消息投递到指定 recipient，不应该共享所有人的 `messages[]`
- `runtime task` 负责追踪当前执行槽位，例如 teammate 是否 running、idle、shutdown
- `leader` 负责创建 team、分派消息、接收回报，但不应该成为所有 teammate 的上下文容器

所以 Day 16 真正新增的不是“并行多调用”，而是：

> 系统第一次拥有了“长期协作成员可以被点名、被唤醒、被继续投递消息”的能力。

## 2. Teammate 生命周期图

```text
leader creates team
  ->
team roster records leader
  ->
leader spawns alice
  ->
team roster records alice as member
  ->
alice runtime loop starts
  ->
leader writes message to alice mailbox
  ->
alice drains mailbox before next turn
  ->
mailbox message becomes alice's own user-like input
  ->
alice runs tools / sends reply / reports progress
  ->
alice becomes idle
  ->
leader sends another message or shutdown request
  ->
alice works again or shuts down
```

把这条链拆开看，就是：

| 阶段 | 它手里拿的是什么 | 它回答的问题 |
| --- | --- | --- |
| `leader` | team context、team file、teammate list | 谁来协调、谁来分派、谁来回收结果 |
| `team member` | `name / role / status / agentId` | 这个长期 actor 是谁 |
| `mailbox` | message envelope、recipient inbox | 新消息怎样送到指定 actor |
| `runtime task` | execution slot、abort controller、idle state | 当前这个 actor 是否正在跑 |
| `teammate messages` | teammate 自己的对话历史 | 收到消息后，下一轮模型调用看见什么 |

## 3. 四个对象的硬边界

最容易混掉的是这四个对象：

| 对象 | 它是什么 | 不是 |
| --- | --- | --- |
| `leader` | team 的协调者和消息分派者 | 不是所有 teammate 的共享上下文 |
| `team member` | 长期 actor 身份 | 不是 durable task，也不是 runtime slot |
| `mailbox` | per-recipient message bus | 不是协议状态机，也不是共享 `messages[]` |
| `runtime task` | 当前执行槽位 | 不是 teammate 的长期身份本身 |

最短区分法：

```text
team member  = 谁长期存在
mailbox      = 话怎样送到谁那里
runtime task = 当前谁正在跑
leader       = 谁在协调和回收结果
```

## 4. 一个最真实的例子

如果 leader 说：

> “alice，你以后负责测试。先帮我跑一遍登录模块测试。”

系统内部更像这样：

1. leader 创建或复用一个 team
2. `alice` 被写进 team roster，拥有长期身份
3. 系统启动一个 teammate runtime loop
4. 初始任务或后续消息被写入 `alice` 的 inbox
5. `alice` 在自己的下一轮先读 inbox
6. 这条消息进入 `alice` 自己的 `messages`
7. `alice` 调工具、跑测试、把结果写回 leader inbox
8. `alice` 进入 idle，等待下一条消息或 shutdown

也就是说，`alice` 不是“这次测试任务”。  
`alice` 是长期队友；测试任务是她收到的一段工作输入；runtime task 是她当前活着的执行槽位。

## 5. 和 Day 13 的对应关系

Day 13 的链是：

```text
schedule record -> checker -> notification -> main loop reinjection
```

Day 16 的链是：

```text
team roster -> mailbox -> teammate inbox drain -> teammate loop reinjection
```

两者相似的地方是：

- 都不是直接偷偷执行
- 都先把外部事件变成下一轮输入
- 都把最终决策交还给某条 agent loop

不同点是：

- Day 13 的入口来自时间
- Day 16 的入口来自另一个 actor 的消息
- Day 13 回到 leader 主循环
- Day 16 可以回到某个 teammate 自己的循环

## 6. 一句记忆钩子

> **Day 13 是“未来时间把事情重新提起来”，Day 16 是“长期队友通过 mailbox 被再次叫醒”。**
