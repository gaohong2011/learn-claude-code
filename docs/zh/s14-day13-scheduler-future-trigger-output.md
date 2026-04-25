# Day 13 输出物：`schedule record -> checker -> notification -> 主循环回注`

Day 13 最该压成一页纸的，不是 cron 五字段表，而是下面这条主线：

> **schedule 负责记住未来开始条件；时间到了，checker 只发 notification；真正要不要开工，仍然交给主循环决定。**

## 1. 一页结论

如果把 Day 13 压成 5 句，最该记住的是这些：

- `schedule record` 不是任务执行体，它只是未来触发条件记录
- `checker` 只负责判断“现在是否该触发”
- 命中后先进入 `notification queue`，而不是直接偷偷执行
- `agent_loop` 会把触发结果重新当成输入注入主循环
- 是否长出新的 `runtime task`，要看主循环后续怎么处理

所以 Day 13 真正新增的不是“又多了一种 task”，而是：

> 系统第一次拥有了“未来时间也能成为新意图入口”的能力。

## 2. 链路图

```text
cron_create(...)
  ->
schedule record saved
  ->
checker scans current time
  ->
cron_matches(...) == true
  ->
enqueue scheduled notification
  ->
agent_loop drains queue
  ->
inject as a new user-like message
  ->
model decides what to do next
  ->
maybe spawn runtime task / tool / agent
```

把这条链拆开看，就是：

| 阶段 | 它手里拿的是什么 | 它回答的问题 |
| --- | --- | --- |
| `schedule record` | `id / cron / prompt / recurring / durable / createdAt` | 未来什么时候触发、触发后要说什么 |
| `checker` | 当前时间 + 规则匹配 | 现在到点了吗 |
| `notification` | `scheduled prompt` | 到点这件事怎样回到系统 |
| `main loop reinjection` | 新的 user-like message | 接下来要不要真的开工 |
| `runtime task` | shell / agent / teammate 执行槽位 | 现在到底是谁在跑 |

## 3. 为什么 `schedule` 和 `runtime task` 不是同一层

最容易混掉的地方，是把“未来触发记录”和“当前执行槽位”都叫 task。

最短区分法：

| 层 | 它回答的问题 | 例子 |
| --- | --- | --- |
| `schedule` | 未来什么时候开始 | “每周一 9 点生成周报” |
| `runtime task` | 现在谁正在跑 | `local_bash` 正在执行 `pytest` |

所以：

- `schedule` 还没真的开工
- `runtime task` 已经有执行体在跑

两者可能有关联，但不是同一个对象：

```text
schedule: every Monday 09:00 -> "Run weekly report"
  ->
time arrives
  ->
notification enters main loop
  ->
model chooses to call a tool
  ->
runtime task appears
```

这就是 Day 13 最该守住的边界：

> `schedule` 管未来开始条件，`runtime task` 管当前执行槽位。

## 4. 一个最真实的例子

如果用户说：

> “每周一上午 9 点帮我生成周报。”

系统内部更像这样：

1. 创建一条 `schedule record`
2. 周一 9 点时 `checker` 发现命中
3. 系统发出一条 scheduled notification
4. 主循环把这条通知当作新的输入读到
5. 模型决定调用工具、读文件、写报告，必要时才长出新的 runtime task

也就是说，周报不是 scheduler 自己写出来的。  
scheduler 只是负责在正确时间把这件事重新提起来。

## 5. 一句记忆钩子

> **Day 12 是“等结果回来”，Day 13 是“等开始时刻到来”。**
