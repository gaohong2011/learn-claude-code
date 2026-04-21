# Day 11 学习指南：Durable Task Graph 与任务边界

> 这份讲义面向第一次读任务系统的人。
> Day 11 最重要的不是背几个字段，而是先把下面这句话真正讲顺：
>
> **todo 解决“别忘了做什么”，durable task graph 解决“谁先做、谁在等谁、谁现在可以开始”。**

## 0. 这一天到底在学什么

到 Day 10 为止，你已经看过单 agent 的主循环、上下文控制和错误恢复。

但系统还缺一块非常关键的能力：

> **有些工作不应该只活在当前对话里，而应该变成一张能跨轮次、跨压缩、甚至跨会话存在的工作图。**

所以 Day 11 真正学的是这条链：

```text
复杂目标
  ->
拆成多个 task
  ->
把依赖关系写清楚
  ->
把 task 持久化到磁盘
  ->
根据 blockedBy / blocks 判断谁被卡住、谁 ready
  ->
任务被认领、推进、完成
  ->
完成事件自动解锁后续任务
```

如果只记一句话，可以先记这个：

> Durable task graph 不是“更大的 todo”，而是“能持续推进的工作依赖图”。

## 1. 推荐阅读顺序

建议按下面顺序读，不要一上来就把真实产品代码混成一团：

1. `docs/zh/s12-task-system.md`
   先建立最小任务图的心智模型：`TaskRecord`、`blockedBy`、`blocks`、`ready`
2. `docs/zh/data-structures.md`
   专门看 `TaskRecord` 和 `RuntimeTaskState` 的边界，防止把 Day 11 和 Day 12 混掉
3. `agents/s12_task_system.py`
   看教学版最小实现：任务落盘、依赖双写、完成后自动解锁
4. `/Users/hong.gao/python/src/claude-code-codex/src/Task.ts:27-125`
   看真实代码里的任务公共底座，但要意识到它更偏 runtime task
5. `/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`
   看“后台任务筛选”这类 runtime/UI 判断
6. `/Users/hong.gao/python/src/claude-code-codex/src/commands/tasks/tasks.tsx:1-7`
   最后跳读命令入口，确认 `/tasks` 只是 UI 壳

原因很简单：

- `s12-task-system.md` 告诉你 durable task 到底在解决什么
- `s12_task_system.py` 告诉你这套心智怎样落成最小代码
- `Task.ts` 等真实代码告诉你产品里哪些已经进入 runtime 执行层

## 2. 先背一张总图

先把 Day 11 的主线图背下来：

```text
用户给出复杂目标
  |
  v
模型拆出多个 durable task
  |
  v
task_create / task_update
  |
  v
.tasks/task_N.json
  |
  +-- blockedBy 非空
  |      -> 任务暂时被卡住
  |
  +-- blockedBy 为空且 status=pending
  |      -> task ready
  |
  +-- owner 被设置 / status 进入进行中
  |      -> 任务被认领
  |
  +-- status=completed
         -> 从后续任务的 blockedBy 中移除自己
         -> 解锁后续任务
```

这张图里最关键的是：

- durable task 关心的是“工作图”
- runtime task 关心的是“当前执行槽位”
- `/tasks` 命令只是把 runtime task 列给用户看

## 3. 先分清两个最容易混的东西

### 3.1 `TaskRecord` 是工作目标

`docs/zh/data-structures.md` 对 `TaskRecord` 的定义非常直接：

```python
task = {
    "id": 12,
    "subject": "Implement auth module",
    "description": "",
    "status": "pending",
    "blockedBy": [],
    "blocks": [],
    "owner": "",
    "worktree": "",
}
```

它回答的是：

- 这条工作是什么
- 现在做到哪一步
- 还在等谁
- 它完成后会解锁谁
- 现在是谁在做

所以 `TaskRecord` 的关键词是：

- `blockedBy`
- `blocks`
- `owner`

### 3.2 `RuntimeTaskState` 是执行槽位

同一份总表里，`RuntimeTaskState` 的最小形状更像这样：

```python
runtime_task = {
    "id": "b8k2m1qz",
    "type": "local_bash",
    "status": "running",
    "description": "Run pytest",
    "start_time": 1710000000.0,
    "end_time": None,
    "output_file": ".task_outputs/b8k2m1qz.txt",
    "notified": False,
}
```

它回答的是：

- 当前系统里哪一个执行单元在跑
- 它属于哪种执行类型
- 输出写到哪里
- 用户是否已经收到通知

所以这里的关键词变成：

- `type`
- `outputFile`
- `startTime`
- `notified`

最短结论：

> `TaskRecord` 管工作目标，`RuntimeTaskState` 管执行现场。

## 4. 教学版 `s12_task_system.py` 应该怎么看

### 4.1 先看 `TaskManager`，别先看 CLI

按 `docs/zh/s00f-code-reading-order.md` 的建议，读这一章代码时先看管理器类。

最关键的代码是：

```python
class TaskManager:
    def create(self, subject: str, description: str = "") -> str:
        task = {
            "id": self._next_id, "subject": subject, "description": description,
            "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
        }
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

这段代码最重要的不是“新建了一条 JSON”，而是：

> 从 Day 11 开始，任务第一次脱离 `messages`，真正变成会话外状态。

### 4.2 `ready rule` 是本章最关键的一条规则

文档里的最小判断规则是：

```python
def is_ready(task: dict) -> bool:
    return task["status"] == "pending" and not task["blockedBy"]
```

翻成人话就是：

- 这条任务还没开始
- 没有人挡着它

只要这两条同时满足，它就 ready。

这一步非常关键，因为它说明：

> 任务系统的核心不是“把任务存起来”，而是“稳定判断谁现在可以开工”。

### 4.3 为什么依赖关系要双向写

教学版在 `update()` 里把依赖做成双向：

```python
if add_blocks:
    task["blocks"] = list(set(task["blocks"] + add_blocks))
    for blocked_id in add_blocks:
        blocked = self._load(blocked_id)
        if task_id not in blocked["blockedBy"]:
            blocked["blockedBy"].append(task_id)
            self._save(blocked)
```

这样做的目的不是“字段多写一份”，而是让系统能从两个方向都读懂图：

- 从前往后看：我完成后会解锁谁
- 从后往前看：我现在还在等谁

### 4.4 为什么“完成”必须触发“解锁”

教学版里最值得反复看的代码是：

```python
def _clear_dependency(self, completed_id: int):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

这段代码说明一件非常重要的事：

> durable task graph 不是静态任务表，而是会随着完成事件自动推进的图。

如果没有这一步，任务系统就只是在记账，不是在推进工作。

### 4.5 `owner` 为什么重要

教学版里 `owner` 只是一个简单字段，但它代表了一个更大的问题：

> 这条任务现在到底是谁在做。

在 Day 11 里，你只要先记住：

- `owner` 为空：还没人认领
- `owner` 被写入：任务已被某个执行者认领

到后面的多 agent 章节，这个字段会变得更重要。

## 5. 怎样把学习状态机和代码状态分开看

Day 11 的学习清单要求你整理至少六种状态或动作：

- 创建
- 阻塞
- 解锁
- 认领
- 完成
- 失败

这里最容易踩坑的是：

> 教学版 `s12_task_system.py` 的 `status` 枚举并没有把这六个词全部做成独立字段。

教学版只有：

```text
pending -> in_progress -> completed
deleted
```

所以你在输出物里整理的“六种状态”，更准确地说是：

- 一部分是显式状态值
- 一部分是图上的阶段性现象
- 一部分是为了贴近真实系统而补上的学习状态

最实用的拆法是：

- 创建：任务刚写入磁盘
- 阻塞：`status=pending` 且 `blockedBy` 非空
- 解锁：前置完成后，`blockedBy` 被清空
- 认领：`owner` 写入，或进入 `in_progress`
- 完成：`status=completed`
- 失败：教学版未单列，但真实 durable graph 设计里应视为独立终态

这样就不会把“学习状态机”和“代码里的最小枚举”硬绑成一回事。

## 6. 真实源码该怎样读，不会读偏

### 6.1 `Task.ts`：这是 runtime task 公共底座，不是 Day 11 任务图本体

你指定的片段里，这段最关键：

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

看到这些字段就要立刻意识到：

- `outputFile`、`outputOffset` 在管输出流
- `startTime`、`endTime` 在管执行生命周期
- `notified` 在管通知/UI

所以它更像 Day 12 的 `RuntimeTaskState`，而不是 Day 11 的 `TaskRecord`。

再看任务 ID 和基础状态初始化：

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

这里也能看出它的关注点是 runtime：

- 任务 ID 前缀在区分执行槽位类别，不是在表达依赖关系
- 初始化时立刻准备 `outputFile`，说明它天然带着执行输出语义
- `pending` 在这里更像“等待开始执行”，不是 Day 11 那种“等待依赖解锁”的图状态

再看状态：

```ts
export type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
```

这也是 runtime 视角，因为执行系统必须区分：

- 自然完成
- 执行失败
- 被外部停止

### 6.2 `tasks/types.ts`：这里的判断是“后台任务筛选”，不是“ready rule”

最关键的函数是：

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

它回答的问题是：

> 这条 runtime task 要不要显示在后台任务指示器里？

它不是在回答：

> 这条 durable task 是否已经 ready？

这两个判断不要混：

- `ready rule` 关心依赖是否解除
- `isBackgroundTask()` 关心 UI 上是不是后台执行项

### 6.3 `commands/tasks/tasks.tsx`：这是一个很薄的命令入口

这份文件几乎只有一件事：

```tsx
export async function call(onDone, context): Promise<React.ReactNode> {
  return <BackgroundTasksDialog toolUseContext={context} onDone={onDone} />;
}
```

也就是说：

- `/tasks` 命令本身没有 durable graph 调度逻辑
- 它只是把后台任务面板挂出来
- 真正的筛选和详情逻辑在 `BackgroundTasksDialog`

所以学习清单才把它标成跳读。

## 7. 一张你可以直接复述的 Day 11 状态机

建议把 Day 11 的 durable task graph 先背成下面这版：

```text
创建
  ->
如果有前置依赖 -> 阻塞
如果没有前置依赖 -> ready / 已解锁
  ->
认领
  ->
执行中
  ->
完成
  | \
  |  \-> 失败
  |
  \-> 完成后移除后继任务中的 blockedBy
         -> 解锁后继任务
```

这里有两个细节一定要记住：

1. “阻塞”和“已解锁”未必都是独立 status 值。
   它们更像由 `blockedBy` 派生出来的图状态。
2. “失败”在教学版里没展开，但在真正的 durable task 设计里应该被当成独立终态看。

## 8. 读完 Day 11，最少要能自己讲出这几句

1. todo 更像当前会话清单，durable task graph 更像跨轮次的工作图。
2. `TaskRecord` 关心的是工作目标、依赖和认领，不是执行输出文件。
3. `ready` 的核心判断不是“看起来像能做”，而是 `pending` 且 `blockedBy` 为空。
4. 完成任务不只是改 `status`，还必须解锁后续任务。
5. `Task.ts`、`tasks/types.ts`、`/tasks` 命令已经明显站在 runtime/UI 这一层，不是 Day 11 教学版 durable graph 的纯实现。

## 9. 最低验证怎么做

先跑学习清单里的最低验证：

```bash
python agents/s12_task_system.py
```

你不需要一开始就做复杂实验。

只要能自己手动演一遍下面这条链，就已经说明你读懂了：

```text
create task A
create task B
让 A blocks B
完成 A
观察 B 的 blockedBy 被清空
```

只要这条链你能复述，Day 11 的主心智就立住了。
