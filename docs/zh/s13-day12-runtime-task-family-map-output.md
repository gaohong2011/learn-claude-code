# Day 12 输出物：Runtime Task 类型族地图与执行槽位说明

Day 12 最重要的不是记住 `LocalShellTask`、`LocalAgentTask` 这些名字，而是把下面这句话讲顺：

> runtime task 解决“现在谁在跑、跑到哪了、结果怎么回系统”，它描述的是执行槽位层，不是 Day 11 的 durable work graph 层。

## 0. runtime-task异步结果怎么放入上下文，发起LLM调用？
- 忙的时候：下一次安全点顺带塞进当前上下文
- 闲的时候：队列变更事件主动唤起下一轮 loop


## 1. 一页结论

如果把 Day 12 压成一页纸，最该记住的是这 4 句：

- `local_bash` 是本地 shell 进程槽位
- `local_agent` 是本地 agent 生命周期槽位
- `remote_agent` 是远程 session 的本地镜像槽位
- `teammate` 更准确地说是 `in_process_teammate`，它是长寿命协作 actor 槽位

这 4 类对象虽然执行体不同，但都共享同一层 runtime task 外壳：

- 有统一的 `id / type / status / outputFile / notified`
- 能被挂进 `AppState.tasks`
- 能被 UI、通知系统、恢复逻辑和 kill 流程统一看见

所以 Day 12 真正新增的不是“又多了几种任务名字”，而是：

> 系统第一次把不同执行体统一抽象成“执行槽位层”。

## 2. Runtime Task 类型族地图

| runtime task 类型 | 它代表的执行槽位 | 真正执行者 | 运行位置 | 本地 task 外壳主要负责什么 | 结果如何回到主循环 |
| --- | --- | --- | --- | --- | --- |
| `local_bash` | 本地 shell 进程槽位 | `ShellCommand` / 本地进程 | 本地 | 注册 task、前后台切换、stall 检测、输出文件、通知、回收 | 终态后发 `task-notification`，模型通常再读 output file |
| `local_agent` | 本地 agent 生命周期槽位 | `runAgent` 生命周期 | 本地 | 记录 progress、summary、retain、pendingMessages、panel 生命周期 | 由 `AgentTool` 高层通知回注，或把 pending user messages 重新注入主循环 |
| `remote_agent` | 远程 session 的本地镜像槽位 | 远程 CCR session | 远程执行，本地只镜像/poll | poll、restore、本地状态镜像、output file、通知、远程归档 | poll 到终态后发 `task-notification`；remote review 可直接内联 findings |
| `in_process_teammate` | 长寿命 teammate actor 槽位 | teammate runner / mailbox 协作流 | 本地同进程 | 身份可见性、消息镜像、UI cap、pendingUserMessages、idle/active 状态 | 通过 mailbox / pendingUserMessages / 状态变化回到 leader，而不是一次性命令完成通知 |

把这张表换成一句更土的话就是：

- `local_bash` 管一个命令
- `local_agent` 管一个活着的 agent 会话
- `remote_agent` 管一个远端 session 的本地影子
- `in_process_teammate` 管一个长期存在的协作端口

## 3. 为什么这 4 类都属于 runtime task

它们的共同点不是“都能后台跑”，而是都回答同一组运行时问题：

1. 现在有没有一个执行单元活着
2. 它属于哪种执行类型
3. 它的当前状态是什么
4. 它的输出去哪了
5. 它结束后怎样通知系统

也就是说，runtime task 层统一的是：

- 状态登记
- 生命周期可见性
- 输出与通知
- kill / restore / GC 接口

它并不要求真正执行工作的东西一定长得一样：

- shell task 背后是本地进程
- local agent 背后是本地 async agent 生命周期
- remote agent 背后是远端 session
- teammate 背后是 mailbox 驱动的长期 actor

## 4. 它和 Day 11 durable task 的硬边界

最容易混掉的地方，是把 Day 11 的工作目标和 Day 12 的执行槽位看成同一个“task”。

最短区分法：

| 层 | 它回答的问题 | 典型例子 |
| --- | --- | --- |
| durable task / work graph | 还有哪些工作目标、谁依赖谁、谁被解锁 | “先写 parser，再写 tests” |
| runtime task / execution slot | 现在到底是谁在执行、输出写到哪、何时回注主循环 | `pytest` 后台跑、subagent 继续执行、remote review 在云端 poll |

所以：

- Day 11 的 `task` 更像工作图节点
- Day 12 的 `runtime task` 更像执行槽位对象

两层可以关联，但不能混成一个词：

```text
work graph task #12: 实现 parser
  |
  +-- spawns runtime task A: local_bash (pytest)
  +-- spawns runtime task B: local_agent (coder worker)
```

这就是 Day 12 最该守住的边界：

> durable task 管“要做什么”，runtime task 管“现在谁在跑”。
