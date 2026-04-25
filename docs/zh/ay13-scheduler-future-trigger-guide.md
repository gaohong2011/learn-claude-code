# Day 13 学习指南：Scheduler 与未来触发

> Day 12 学的是“现在谁在跑”。  
> Day 13 学的是“未来什么时候开工”。
>
> 这一天最重要的不是背 cron 语法，而是先把下面这句话讲顺：
>
> **schedule record 只负责记住未来的开始条件；时间到了，再把 prompt 送回同一条主循环。**

## 0. 这一天到底在学什么

到 Day 12 为止，你已经有了后台执行能力。

也就是说，系统已经能处理：

- 命令已经启动了，但比较慢
- agent 已经跑起来了，但结果要稍后回来
- 远程任务已经存在了，本地需要继续跟踪

Day 13 再往前走一步，开始回答另一个问题：

> **如果一件事不是现在做，而是明天、下周一、30 分钟后再做，系统该怎么记住它？**

所以这一天真正要学的是这条链：

```text
用户 / 模型创建 schedule
  ->
schedule record 被保存
  ->
checker 定期看“现在是否匹配”
  ->
匹配后生成 scheduled notification
  ->
主循环下一轮把它当作新的输入重新注入
  ->
模型再决定要不要启动 runtime task / tool / agent
```

如果只记一句话，先记这个：

> **Scheduler 解决的是“未来何时开始”，不是“当前谁在执行”。**

## 1. 这次重点看哪些代码

学习计划里的 Day 13 已经把主线抓得很准，建议就按这组材料读。

### 必读

- `docs/zh/s14-cron-scheduler.md`
- `agents/s14_cron_scheduler.py`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronCreateTool.ts:56-157`
- `/Users/hong.gao/python/src/claude-code-codex/src/hooks/useScheduledTasks.ts:40-139`

### 选读

- `/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronListTool.ts:37-97`

### 跳读

- `/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronDeleteTool.ts:35-95`

为了把这些代码读顺，我建议在 `agents/s14_cron_scheduler.py` 里优先盯这几个入口：

- `CronLock`
- `cron_matches(...)`
- `CronScheduler.create(...)`
- `CronScheduler._check_tasks(...)`
- `CronScheduler.drain_notifications(...)`
- `agent_loop(...)`

原因很简单：

- `create(...)` 说明调度记录是怎么被建出来的
- `_check_tasks(...)` 说明时间到了之后到底谁负责触发
- `drain_notifications(...)` 和 `agent_loop(...)` 说明它怎么回到主线

## 2. 先建立最小心智模型

先别急着看 cron 语法细节。

先把 Day 13 想成下面这 4 个盒子：

```text
1. schedule record
2. checker
3. notification queue
4. main loop reinjection
```

它们的分工分别是：

- `schedule record`：记住未来任务的触发条件和要说的话
- `checker`：定期判断“现在是否该触发”
- `notification queue`：不要直接偷偷执行，先把触发结果排队
- `main loop reinjection`：在下一轮模型调用前，作为新的输入回到主循环

这里最值得先背下来的结论是：

> **scheduler 自己不是第二个 agent，它只是未来触发入口。**

也就是说，到了时间点之后，scheduler 不负责“把所有工作做完”，它只负责：

1. 确认应该触发了
2. 发出通知
3. 把决定权重新交还给主循环

## 3. Day 13 的主线到底怎么跑

### 3.1 先创建一条 schedule

在教学版 `agents/s14_cron_scheduler.py` 里，`CronScheduler.create(...)` 会把一条未来触发记录保存到 `self.tasks`。

最核心的字段有这些：

- `id`
- `cron`
- `prompt`
- `recurring`
- `durable`
- `createdAt`

如果是 recurring 任务，还可能补上 `jitter_offset`，避免所有任务都撞在整点触发。

在参考实现里，`CronCreateTool` 额外做了几件非常产品化的事：

- 校验 cron 表达式是否合法
- 校验未来一年内是否真的有匹配时间
- 控制最大任务数
- 禁止 teammate 创建 durable cron

这里有一个非常值得记住的边界：

> **create schedule 的结果不是“立刻开工”，而是“先把未来开工条件记下来”。**

### 3.2 checker 只负责“现在到点了吗”

教学版里真正负责触发的是两层：

- `_check_loop()`：后台线程，每秒醒一次
- `_check_tasks(now)`：但只按“分钟粒度”真正做一次匹配

`_check_tasks(...)` 主要做 4 件事：

1. 看 recurring 任务是否超过自动过期时间
2. 必要时应用 jitter 偏移
3. 用 `cron_matches(...)` 判断当前时间是否匹配
4. 匹配后发 notification，并处理 one-shot 删除

所以 Day 13 的 checker，本质上只是一个时间判定器：

> 它不负责执行业务，只负责把“未来条件已经满足”这件事说出来。

### 3.3 触发后先发通知，不要直接偷偷执行

教学版最关键的一行，是匹配后把内容塞进队列：

```python
self.queue.put(f"[Scheduled task {task['id']}]: {task['prompt']}")
```

这一步非常重要，因为它说明：

> **schedule 命中以后，系统先得到的是 notification，不是已经完成的结果。**

这和 Day 12 的后台任务很像，都是先进入一层通知通道，再由主循环接手。

### 3.4 主循环下一轮再接住它

`agent_loop(...)` 在每次模型调用前都会先 `drain_notifications()`。

然后把这些通知重新塞回 `messages`：

```python
messages.append({"role": "user", "content": note})
```

这就是 Day 13 最核心的教学动作：

> **把未来触发重新翻译成一条新的用户输入。**

到了这一步之后，系统又回到你熟悉的主线：

- 模型读到新输入
- 决定要不要调工具
- 决定要不要新建 runtime task
- 决定要不要启动 agent / shell / subagent

也就是说，scheduler 并不取代主循环，它只是多了一种“主循环收到新意图”的入口。

## 4. 和 Day 12 到底差在哪

这是 Day 13 最容易混掉的地方。

### 一张最短对比表

| 层 | 它回答的问题 | 典型例子 |
| --- | --- | --- |
| `schedule` | 未来什么时候开始 | “每周一 9 点提醒我生成周报” |
| `runtime task` | 现在谁正在跑 | `local_bash` 正在执行测试 |
| `durable task` | 整体还有什么工作目标 | “先写报告，再发邮件” |

所以你可以把三层这样记：

- Day 11 的 durable task 管“要做什么”
- Day 12 的 runtime task 管“现在谁在跑”
- Day 13 的 schedule 管“未来什么时候开工”

最短判断法：

如果某个对象还没开始执行，只是在等未来时间点，那它更像 `schedule`。  
如果某个对象已经有执行体在跑、还能被 kill、能看到输出文件和状态，那它更像 `runtime task`。

## 5. 参考实现里最值得看的产品化细节

如果你只读教学版，会得到最干净的主线。  
如果你再读参考实现，会看到主线外面包了一层更真实的产品约束。

### 5.1 `CronCreateTool.ts`

最值得看的是三类产品约束：

- 输入校验：不是所有 5 字段字符串都真的可用
- 上限控制：不能无限创建 schedule
- durable 限制：teammate 不应该留下跨会话孤儿 schedule

### 5.2 `useScheduledTasks.ts`

这部分最值得看的，不是 React 细节，而是“调度触发后怎么回到正确执行对象”：

- 普通 leader 任务：回到主消息队列
- teammate 任务：按 `agentId` 路由回对应 teammate
- teammate 已消失：清理 orphaned cron，避免继续空打

这说明产品实现比教学版多了一层问题：

> **不是所有 schedule 都回到同一个入口，有些要回到 leader，有些要回到 teammate。**

### 5.3 `CronListTool` 和 `CronDeleteTool`

这两块读一遍就够，但你要记住它们回答的是两个很现实的问题：

- 系统里现在都记住了哪些未来触发
- 某条未来触发怎样被显式取消

这会让 schedule 从“能创建”变成“可管理”。

## 6. 初学者最容易犯的错

### 6.1 一上来沉迷 cron 语法

语法当然重要，但 Day 13 的主线不是“背表达式”，而是：

**schedule record -> checker -> notification -> 主循环回注**

### 6.2 把 scheduler 想成后台执行器

后台执行器更像 Day 12 的 runtime task 世界。  
scheduler 不是拿来直接跑工作本体的，它更像一个“未来唤醒器”。

### 6.3 以为 schedule 命中就等于 runtime task 已经存在

不是。

命中之后最先出现的是 notification。  
至于后面是否真的会长出 shell / agent / teammate runtime task，还是由主循环和模型决定。

### 6.4 把 recurring / one-shot 和 durable / session-only 混成一回事

这是两组不同维度：

- `recurring / one-shot`：说的是触发次数
- `durable / session-only`：说的是是否跨重启保存

### 6.5 忘了多会话去重

如果两个会话同时在检查同一批任务，就可能重复触发。

所以教学版里的 `CronLock` 很值得看，它说明：

> 时间命中本身不难，难的是“谁有资格发这次触发”。

## 7. 今天的输出物该怎么写

学习计划给 Day 13 的产出要求非常好：

> 画出“schedule record -> checker -> notification -> 主循环回注”的链路图，并写清 `schedule` 和 `runtime task` 为什么不是同一层。

我建议你把输出物压成两部分：

### 第一部分：一条主链

```text
schedule_create(...)
  ->
record saved
  ->
time checker matches now
  ->
scheduled notification queued
  ->
agent_loop drains queue
  ->
messages.append(user-like prompt)
  ->
model decides next action
```

### 第二部分：一句硬边界

```text
schedule 管未来开始条件
runtime task 管当前执行槽位
```

只要你把这两部分讲顺，这一天就算学对了。

## 8. 读完 Day 13 应该能自己说清的话

如果这一天真的吃透了，你应该能不用看文档，自己说清下面这几句：

- scheduler 不是第二套主循环
- cron 命中后先进入 notification，不是直接偷偷执行
- schedule 只表示“未来何时开工”，不表示“现在谁在跑”
- runtime task 是执行槽位，schedule 是未来触发条件
- Day 12 解决“等结果”，Day 13 解决“等开始”

## 9. 最低验证

```sh
cd learn-claude-code
python agents/s14_cron_scheduler.py
```

可以重点观察三件事：

1. 新建 recurring / one-shot 任务后，列表里怎么显示
2. 到点时是否先出现 notification，再由主循环接住
3. durable 任务在重启后是否还能被重新加载

最后把这句话记住就够了：

> **Day 12 的关键词是“后台继续跑”，Day 13 的关键词是“未来再开工”。**
