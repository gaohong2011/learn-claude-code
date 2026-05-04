# Day 18 输出物：自治协作图

Day 18 最该压成一页纸的，不是“让 agent 自己一直跑”，而是下面这条主线：

> **idle teammate 可以自己认领 ready task；但 worker 启动、权限请求、结果回收仍然必须接回统一控制面。**

## 1. 一页结论

如果把 Day 18 压成 5 句，最该记住的是这些：

- 自治从 idle 阶段开始，不是从无限循环开始
- 可认领任务必须满足 `pending + no owner + no blockedBy + role match`
- 认领必须原子化，避免多个 teammate 抢同一条 task
- worker 可以自治找活，但高风险 tool permission 仍要桥接给 leader
- worker 完成、失败、空闲后必须把结果或状态通知回 leader

所以 Day 18 真正新增的是：

> 团队从“leader 手动派活”升级到“teammate 在规则内主动接活”。

## 2. 自治协作图

```text
teammate finishes current work
  ->
enter idle
  ->
check inbox
  ->
new message?
  -> yes -> reinject message -> WORK
  -> no
       ->
       scan task board
       ->
       find claimable task?
       -> no -> keep idle or shutdown after timeout
       -> yes
            ->
            claim task atomically
            ->
            write claim event
            ->
            inject identity + task prompt
            ->
            start / resume worker runtime loop
            ->
            worker calls tools
            ->
            permission needed?
            -> no -> continue work
            -> yes
                 ->
                 send permission request to leader mailbox
                 ->
                 leader approve / reject
                 ->
                 worker continue / abort
            ->
            send result or idle notification to leader
            ->
            return to idle
```

## 3. 五段链路拆解

| 阶段 | 关键对象 | 它回答的问题 |
| --- | --- | --- |
| 任务认领 | `TaskRecord` + claim policy | 哪条 task 当前可以被谁接走 |
| worker 启动 | `InProcessTeammateTaskState` | 当前执行槽位怎样被追踪和中止 |
| 权限桥接 | permission request mailbox | worker 的风险操作谁来批准 |
| 结果回收 | idle / result notification | leader 怎样知道 worker 状态变化 |
| 恢复续行 | identity block + next prompt | teammate 怎样带着身份继续下一轮 |

一句话压缩：

```text
claim task -> run worker -> ask permission when needed -> report back -> idle and continue
```

## 4. 为什么“权限不自治”

自治只改变一件事：

> 谁来发现下一份可做的工作。

它不改变这些规则：

- 风险工具仍然要走 permission control plane
- worker 不能自己批准自己的高风险动作
- leader / user 仍然能 reject、abort、stop task
- permission request 要能取消，不能让 worker 永远挂起

所以 Day 18 的硬边界是：

> **自治认领 task，不自治审批权限。**

## 5. 最容易混的三层

| 层 | 是什么 | 不是 |
| --- | --- | --- |
| `task` | 工作目标 | 不是执行进程 |
| `teammate` | 长期 actor 身份 | 不是某一条 task |
| `runtime task` | 当前执行槽位 | 不是 durable work goal |

准确说法应该是：

> `alice` 作为 teammate 自动认领了 `task #7`，然后以一个 `in_process_teammate` runtime slot 推进它。

## 6. 一句记忆钩子

> **Day 17 让团队请求有编号；Day 18 让空闲队友按规则自己接下一份工作，但权限仍然回到 leader。**
