# Day 15 输出物：最小 `task + runtime slot + schedule queue` 原型设计

Day 15 的产出项可以实现原型，也可以写等价设计文档。这里采用设计文档路径，把最小系统压成三类状态和一条回注链：

> **task 保存目标，runtime slot 保存执行现场，schedule queue 保存未来入口，agent loop 负责 drain 和续行。**

## 1. 一页结论

最小原型不需要一开始就支持多 agent、worktree、MCP 或完整 UI。

它只需要能讲顺这条链：

```text
schedule queue fires
  ->
scheduled prompt enters agent loop
  ->
model starts runtime slot
  ->
runtime slot finishes and emits notification
  ->
agent loop injects notification
  ->
model reads output and updates durable task
```

所以最小系统由 4 个模块组成：

| 模块 | 责任 |
| --- | --- |
| `TaskStore` | 管 durable work graph：目标、依赖、owner、完成状态 |
| `RuntimeSlotManager` | 管当前执行槽位：running / completed / failed / killed、输出文件 |
| `ScheduleQueue` | 管未来触发：cron/at、prompt、recurring/one-shot |
| `AgentLoop` | 管回注：drain schedule/runtime notification，追加 messages，进入下一轮 |

## 2. 数据结构

### 2.1 Durable Task Record

```json
{
  "id": 12,
  "subject": "Fix login failure",
  "description": "Investigate failing auth tests and patch implementation.",
  "status": "pending",
  "blockedBy": [],
  "blocks": [13],
  "owner": null,
  "createdAt": 1710000000.0,
  "updatedAt": 1710000000.0
}
```

字段边界：

- `subject / description` 描述工作目标
- `blockedBy / blocks` 描述工作图依赖
- `owner` 描述谁认领了这条工作
- 不放 `output_file`
- 不放 shell process / abort controller
- 不放 notification 消费状态

### 2.2 Runtime Slot Record

```json
{
  "id": "r8k2m1qz",
  "type": "local_bash",
  "status": "running",
  "description": "pytest tests/test_auth.py -q",
  "taskId": 12,
  "startedAt": 1710000100.0,
  "finishedAt": null,
  "outputFile": ".runtime-tasks/r8k2m1qz.log",
  "resultPreview": "",
  "notified": false
}
```

字段边界：

- `taskId` 可以关联 durable task，但 runtime slot 不是 durable task 本身
- `status` 描述执行现场
- `outputFile` 保存完整输出入口
- `notified` 是防重复通知的幂等阀门
- 不放 `blockedBy / blocks`

### 2.3 Schedule Record

```json
{
  "id": "c7p9a2bd",
  "cron": "0 9 * * 1",
  "prompt": "Start weekly report task",
  "recurring": true,
  "durable": true,
  "createdAt": 1710000200.0,
  "lastFiredAt": null
}
```

字段边界：

- `cron` 或 `at` 表示未来触发条件
- `prompt` 是触发后送回主循环的内容
- `recurring` 决定是否重复
- `durable` 决定是否跨会话保存
- 不放当前执行状态

### 2.4 Notification Record

```json
{
  "id": "n5q4d8ya",
  "kind": "runtime_completed",
  "priority": "next",
  "targetAgentId": null,
  "payload": {
    "runtimeSlotId": "r8k2m1qz",
    "taskId": 12,
    "status": "completed",
    "outputFile": ".runtime-tasks/r8k2m1qz.log",
    "summary": "pytest finished"
  },
  "createdAt": 1710000300.0,
  "consumedAt": null
}
```

字段边界：

- notification 是回注信封，不是完整输出
- `priority` 决定何时 drain
- `targetAgentId` 决定谁能消费
- `consumedAt` 防重复注入

## 3. 状态机

### 3.1 Durable Task 状态机

```text
pending
  -> blocked        blockedBy 非空时逻辑上不可启动
  -> in_progress   被 owner 认领或 runtime slot 启动
  -> completed     模型确认目标已完成
  -> failed        模型确认目标无法完成或需要人工处理
  -> deleted       显式删除
```

关键规则：

- `blocked` 可以不作为独立字段，而由 `status=pending && blockedBy.length > 0` 推导
- 完成 task 时，要从后续 task 的 `blockedBy` 中移除自己
- runtime slot 完成不自动等于 durable task 完成

### 3.2 Runtime Slot 状态机

```text
running
  -> completed
  -> failed
  -> killed
  -> timeout
```

关键规则：

- 创建 slot 时立即写入 `outputFile`
- 终态必须设置 `finishedAt`
- 终态后最多发一次 notification
- `killed` 可以不发完整 XML/上下文通知，但仍要关闭 SDK 或外部生命周期事件

### 3.3 Schedule 状态机

```text
waiting
  -> fired

fired + recurring
  -> waiting

fired + one-shot
  -> deleted

waiting + expired
  -> deleted
```

关键规则：

- schedule 命中只 enqueue prompt
- schedule 不直接创建 runtime slot
- durable schedule 可以在启动时检查 missed fire

### 3.4 Notification 状态机

```text
queued
  -> injected
  -> consumed
```

关键规则：

- drain 时按 `priority` 和 `targetAgentId` 过滤
- 注入为 message / attachment 后标记 consumed
- 同一 runtime slot 的 completion notification 要能折叠或幂等

## 4. 触发链

### 4.1 从未来触发到启动执行

```text
ScheduleQueue.tick(now)
  ->
cron_matches(schedule.cron, now)
  ->
NotificationQueue.push(kind="scheduled_prompt", payload={prompt})
  ->
AgentLoop.before_model_call()
  ->
drain scheduled_prompt
  ->
messages.append({"role": "user", "content": "[Scheduled task ...] prompt"})
  ->
model decides to call background_run(...)
  ->
RuntimeSlotManager.spawn(...)
```

这条链说明：

> schedule 是未来入口，不是执行器。

### 4.2 从执行完成到回注上下文

```text
RuntimeSlotManager worker finishes
  ->
write outputFile
  ->
runtime.status = completed / failed
  ->
if not runtime.notified:
    runtime.notified = true
    NotificationQueue.push(kind="runtime_completed", payload={outputFile, status, summary})
  ->
AgentLoop.before_model_call()
  ->
drain runtime_completed
  ->
messages.append({"role": "user", "content": "<task-notification>...</task-notification>"})
  ->
model reads outputFile if needed
```

这条链说明：

> notification 是取结果的入口，不是完整结果本体。

### 4.3 从执行结果到更新 durable task

```text
model sees runtime_completed notification
  ->
model reads outputFile
  ->
model decides:
    tests pass -> task_update(status="completed")
    tests fail -> edit code / spawn another runtime slot
    blocked -> task_update(add_blocked_by=[...])
```

这条链说明：

> durable task 是否完成，要由主循环在读完执行结果后判断。

## 5. 最小伪代码

```python
class TaskStore:
    def create(self, subject, description=""): ...
    def ready(self):
        return [t for t in self.tasks if t["status"] == "pending" and not t["blockedBy"]]
    def complete(self, task_id):
        self.tasks[task_id]["status"] = "completed"
        for task in self.tasks.values():
            if task_id in task["blockedBy"]:
                task["blockedBy"].remove(task_id)
```

```python
class RuntimeSlotManager:
    def spawn(self, command, task_id=None):
        slot_id = new_id()
        self.slots[slot_id] = {
            "id": slot_id,
            "type": "local_bash",
            "status": "running",
            "taskId": task_id,
            "outputFile": f".runtime-tasks/{slot_id}.log",
            "notified": False,
        }
        start_thread(lambda: self._run(slot_id, command))
        return slot_id

    def _run(self, slot_id, command):
        output, status = run_command(command)
        write_file(self.slots[slot_id]["outputFile"], output)
        self.slots[slot_id]["status"] = status
        if not self.slots[slot_id]["notified"]:
            self.slots[slot_id]["notified"] = True
            notifications.push({
                "kind": "runtime_completed",
                "priority": "next",
                "payload": {
                    "runtimeSlotId": slot_id,
                    "outputFile": self.slots[slot_id]["outputFile"],
                    "status": status,
                },
            })
```

```python
class ScheduleQueue:
    def tick(self, now):
        for schedule in self.schedules:
            if cron_matches(schedule["cron"], now):
                notifications.push({
                    "kind": "scheduled_prompt",
                    "priority": "later",
                    "payload": {"prompt": schedule["prompt"]},
                })
                if not schedule["recurring"]:
                    self.delete(schedule["id"])
```

```python
def agent_loop(messages):
    while True:
        for note in notifications.drain(max_priority="next", target_agent_id=None):
            messages.append(notification_to_message(note))

        response = model(messages=messages, tools=TOOLS)
        messages.append(response)

        if not response.tool_use:
            return

        tool_results = run_tools(response.tool_use)
        messages.extend(tool_results)
```

## 6. 验收标准

一个最小原型或等价实现，只要满足下面 6 条就够：

1. 能创建 durable task，并表达 `blockedBy / blocks`
2. 能启动 runtime slot，并立刻返回 slot id
3. runtime slot 结束后写 output file，而不是把完整输出塞进上下文
4. runtime slot 结束后最多发一次 notification
5. schedule 命中后只 enqueue prompt，不直接执行任务
6. agent loop 会 drain notification，并把它变成下一轮模型可见输入

## 7. 一句记忆钩子

> **task 是目标，slot 是现场，schedule 是闹钟，notification 是回到主循环的信封。**
