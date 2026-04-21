# Day 11 输出物：Durable Task Graph 状态机与任务边界总结

Day 11 最重要的不是把 `task` 这个词看熟，而是把下面这句话分清楚：

> durable task 解决“工作怎么持续推进”，runtime task 解决“现在谁在执行”。

## 1. Durable task 状态机

先给出一版最适合 Day 11 复述的状态机：

```text
创建
  |
  +-- 有前置依赖
  |      -> 阻塞
  |
  +-- 没有前置依赖
         -> 解锁 / ready
                |
                v
              认领
                |
                v
              执行中
              /   \
             v     v
           完成   失败
             |
             v
  从后继任务的 blockedBy 中移除自己
             |
             v
       解锁后续任务
```

这张图里最容易误解的一点是：

> “阻塞”和“解锁”不一定是代码里的显式 `status` 枚举，它们更像从依赖关系推导出来的图状态。

教学版 `s12_task_system.py` 的最小状态只有：

```text
pending -> in_progress -> completed
deleted
```

所以这份输出物里的六种状态或动作，是为了帮助你建立 durable graph 的完整心智，不是说教学版源码已经逐字实现了六个独立枚举值。

## 2. 六种关键状态或动作，最土的解释

| 状态或动作 | 最简单理解 | 在教学版里主要靠什么体现 |
|---|---|---|
| 创建 | 一条新任务正式写入工作图 | `TaskManager.create()` 把 JSON 写进 `.tasks/` |
| 阻塞 | 现在还不能开始，因为前置还没完成 | `blockedBy` 非空 |
| 解锁 | 前置已经完成，可以进入 ready | 前驱完成后从 `blockedBy` 中移除 |
| 认领 | 现在有人负责做这条任务 | `owner` 被设置，或进入 `in_progress` |
| 完成 | 这条工作已经结束，并且会解锁后继 | `status = completed` + `_clear_dependency()` |
| 失败 | 这条工作没做成，需要停在显式终态 | 教学版未单列，真实 durable task 设计应补成独立终态 |

## 3. 三段最关键代码

### 3.1 `TaskManager.create()`：任务第一次变成会话外状态

```python
def create(self, subject: str, description: str = "") -> str:
    task = {
        "id": self._next_id, "subject": subject, "description": description,
        "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
    }
    self._save(task)
    self._next_id += 1
    return json.dumps(task, indent=2)
```

这段代码最关键的意义是：

- 任务不再只活在 `messages` 里
- 任务正式落到磁盘
- durable task 从“会话内清单”升级成“会话外工作图节点”

所以 Day 11 的第一道门槛不是依赖算法，而是：

> 系统第一次真正拥有了可以跨轮次保留的工作状态。

### 3.2 `update()` 里的双向依赖：任务图第一次真正成图

```python
if add_blocks:
    task["blocks"] = list(set(task["blocks"] + add_blocks))
    for blocked_id in add_blocks:
        blocked = self._load(blocked_id)
        if task_id not in blocked["blockedBy"]:
            blocked["blockedBy"].append(task_id)
            self._save(blocked)
```

这段代码说明：

- A 的 `blocks` 记录“我会解锁谁”
- B 的 `blockedBy` 记录“我在等谁”

双向维护的价值是：

- 正向读得懂推进关系
- 反向读得懂阻塞原因

如果没有这一步，系统就只有任务列表，没有任务图。

### 3.3 `_clear_dependency()`：完成事件让工作图继续推进

```python
def _clear_dependency(self, completed_id: int):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

这段代码就是 Day 11 最核心的“自动推进器”：

- 某条任务完成
- 它从别人 `blockedBy` 里被拿掉
- 后继任务因此可能变成 ready

所以 durable task graph 的关键不是“把依赖记下来”，而是：

> 完成事件必须真的改变后继任务的可开工性。

## 4. 真实产品代码里，这三处分别代表什么

### 4.1 `src/Task.ts`：runtime task 的公共底座

```ts
export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

这段代码一看就更偏 runtime，而不是 durable graph：

- `outputFile` 在追踪执行输出
- `startTime` / `endTime` 在追踪生命周期
- `notified` 在追踪 UI/通知

它回答的是“运行中的执行槽位长什么样”，不是“工作依赖图长什么样”。

同一个片段里，`generateTaskId()` 和 `createTaskStateBase()` 也在强化这件事：

```ts
const TASK_ID_PREFIXES = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

export function createTaskStateBase(id, type, description, toolUseId?) {
  return {
    id,
    type,
    status: 'pending',
    description,
    toolUseId,
    startTime: Date.now(),
    outputFile: getTaskOutputPath(id),
    outputOffset: 0,
    notified: false,
  }
}
```

这说明：

- ID 前缀在区分执行类型
- 基础状态一创建就准备好了输出文件和通知字段
- 这套模型天然面向“执行中的任务对象”，不是“依赖图里的工作节点”

### 4.2 `src/tasks/types.ts`：后台任务筛选规则

```ts
export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

这段代码回答的是：

> 哪条 runtime task 要被算进后台任务列表？

它不是 Day 11 的 `ready rule`。

两者差别可以直接记成：

- `ready`：依赖是否解除
- `isBackgroundTask()`：UI 是否应显示为后台执行项

### 4.3 `src/commands/tasks/tasks.tsx`：`/tasks` 命令入口只是薄壳

```tsx
export async function call(onDone, context): Promise<React.ReactNode> {
  return <BackgroundTasksDialog toolUseContext={context} onDone={onDone} />;
}
```

这说明：

- `/tasks` 命令本身不负责 durable task 调度
- 它只负责把后台任务面板打开
- 真正的任务筛选发生在 runtime/UI 层

所以它在 Day 11 学习清单里是“跳读”，不是主角。

## 5. 三条关键边界

### 5.1 `TaskRecord` 和 `RuntimeTaskState` 不是一回事

最短记法：

```text
TaskRecord 管工作目标
RuntimeTaskState 管当前执行槽位
```

如果这条边界混掉，Day 12 的后台任务模型会非常难读。

### 5.2 `ready rule` 和后台任务筛选不是一回事

最短记法：

```text
ready = 现在能不能开始做
background task = 现在要不要显示在后台列表
```

这两个判断面向的是不同问题。

### 5.3 durable graph 和 `/tasks` 面板不是一回事

durable graph 管：

- 工作拆分
- 依赖关系
- 认领与推进

`/tasks` 面板管：

- 当前后台执行项
- 运行中任务详情
- 与 UI 的交互

## 6. 一句话收口

> Day 11 真正补上的，不是一个“任务列表功能”，而是“让工作目标脱离当前对话，变成可持久化、可依赖、可解锁、可认领的工作图”。

如果再往前多讲一句，就是：

> 从 Day 11 开始，系统第一次真正拥有了“会话外工作状态”；而 `Task.ts`、`tasks/types.ts`、`/tasks` 入口则提醒你，真实产品很快就会进入 runtime task 那一层。
