# Day 19 学习指南：Worktree 执行车道

> Day 18 学的是“空闲 teammate 怎样自己接下一份工作”。
> Day 19 学的是“并行任务应该在哪条隔离目录车道里执行”。
>
> 这一天最重要的不是背 `git worktree` 命令，而是先把下面这句话讲顺：
>
> **task 管做什么，runtime task 管现在谁在跑，worktree lane 管在哪做且不互相踩目录。**

## 0. 这一天到底在学什么

到 Day 18 为止，系统已经能让多个 teammate 自治认领任务。

但如果大家都在同一个工作目录里改文件，就会出现：

- 并行任务互相覆盖未提交改动
- 很难单独查看某个任务的 diff
- 删除、保留、恢复某条任务车道时没有清晰状态
- agent 续行后不知道自己之前在哪个目录工作

Day 19 要回答的是：

> **当多个任务并行推进时，系统怎样把“工作目标”和“执行目录”明确绑定？**

所以这一天真正要学的是这条链：

```text
create task
  ->
create worktree lane for task
  ->
bind task_id <-> worktree record
  ->
enter lane and run commands with cwd=worktree_path
  ->
record command / status / events
  ->
closeout: keep or remove
  ->
persist / restore session worktree state when resuming
```

如果只记一句话，先记这个：

> **Worktree 不是 task 本身，而是 task 的隔离执行车道。**

## 1. 这次重点看哪些代码

### 必读

- `docs/zh/s18-worktree-task-isolation.md`
- `docs/zh/team-task-lane-model.md`
- `agents/s18_worktree_task_isolation.py`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts:52-127`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts:79-122,148-329`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/worktree.ts:158-235,702-813,902-1180`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/worktreeModeEnabled.ts:1-11`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:2889-2914,3472-3835`

在教学版 `agents/s18_worktree_task_isolation.py` 里，优先盯这几个入口：

- `EventBus`
- `TaskManager.bind_worktree(...)`
- `TaskManager.record_closeout(...)`
- `WorktreeManager.create(...)`
- `WorktreeManager.enter(...)`
- `WorktreeManager.run(...)`
- `WorktreeManager.closeout(...)`
- `worktree_events`

原因很直接：

- `TaskManager` 说明任务记录怎样绑定执行车道
- `WorktreeManager` 说明车道怎样创建、进入、执行和收尾
- `EventBus` 说明车道生命周期必须可观察
- `closeout(...)` 说明保留和移除是同一个收尾决策的两个分支

## 2. 先建立最小心智模型

先把 Day 19 想成下面这 3 层：

```text
Task Layer
  做什么、谁负责、工作状态如何

Runtime Layer
  当前有什么执行槽位正在跑

Worktree Lane Layer
  在哪个隔离目录里推进这份工作
```

它们回答的问题完全不同：

| 层 | 回答的问题 | 典型字段 |
| --- | --- | --- |
| `task` | 要做什么 | `id / subject / status / owner` |
| `runtime task` | 现在谁正在跑 | `taskId / type / status / abortController` |
| `worktree lane` | 在哪做 | `name / path / branch / task_id / status` |

这里最值得先背下来的结论是：

> **task 是目标，runtime task 是执行槽位，worktree 是目录车道。**

把这三层混起来，后面的 keep / remove / resume 都会乱。

## 3. Day 19 的主线到底怎么跑

### 3.1 创建：先有任务，再有车道

教学版推荐的顺序是：

```text
task_create(...)
  ->
worktree_create(name, task_id)
  ->
TaskRecord.worktree = name
  ->
WorktreeRecord.task_id = task_id
```

不要先开一堆目录，再回头猜它们属于哪个 task。

最小绑定结果应该同时出现在两边：

```text
.tasks/task_12.json
  worktree = "auth-refactor"
  worktree_state = "active"
  last_worktree = "auth-refactor"

.worktrees/index.json
  name = "auth-refactor"
  path = ".worktrees/auth-refactor"
  branch = "wt/auth-refactor"
  task_id = 12
  status = "active"
```

这就是最小的双向可查关系。

### 3.2 进入：分配车道不等于正在车道里工作

教学版把创建和进入分开：

```text
worktree_create(...)
  ->
worktree_enter(...)
```

参考实现里的 `EnterWorktreeTool` 会做更多 session-level mutation：

- 检查当前 session 是否已经在 worktree 里
- 找到 canonical git root
- 创建或恢复 worktree session
- `process.chdir(worktreePath)`
- 更新 `cwd / originalCwd`
- 保存 worktree state
- 清掉依赖 cwd 的 prompt / memory cache

所以“进入车道”不是 UI 文案，而是系统状态改变：

> 当前这条 session 的后续命令默认在 worktree path 里执行。

### 3.3 执行：命令必须以 worktree path 作为 cwd

教学版的关键动作在 `WorktreeManager.run(...)`：

```python
subprocess.run(command, cwd=path, ...)
```

这行才是隔离执行的核心。

如果 task 已经绑定了 worktree，但命令仍在主仓库目录跑，那么 worktree lane 只是摆设。

执行时还要记录：

- `last_entered_at`
- `last_command_at`
- `last_command_preview`
- `worktree.run.before / after / timeout` event

这让车道不只是目录，而是可观察的执行 lane。

### 3.4 保留：任务可以结束，但车道仍然 kept

`ExitWorktreeTool` 的 `action="keep"` 会：

- 保留 worktree path
- session 回到 original cwd
- 清掉 current worktree session
- 记录 kept analytics / message

教学版的 `worktree_closeout(action="keep")` 会：

- `WorktreeRecord.status = "kept"`
- `TaskRecord.worktree_state = "kept"`
- 写 closeout reason
- 可以选择是否把 task 标记 completed

这里要守住一个边界：

> **task completed 和 worktree kept 可以同时成立。**

任务状态和车道状态不是同一层。

### 3.5 移除：删除前必须检查改动

参考实现里的 `ExitWorktreeTool.validateInput(...)` 很值得看。

当 `action="remove"` 且没有 `discard_changes` 时，它会先统计：

- uncommitted files
- commits since original head

如果有改动，会拒绝删除，并要求明确确认。

这说明 remove 是危险收尾动作，不是普通清理：

> 删除 worktree 可能永久丢掉未合并工作，所以必须 fail-closed。

教学版里也应该至少建立这个原则：

- remove 前先检查状态
- remove 后更新 task closeout
- remove 后更新 worktree status
- remove failed 要写 event

### 3.6 恢复：worktree state 要进 transcript / session storage

参考实现不是只在内存里记 current worktree。

`saveWorktreeState(...)` 会把 worktree session 写入当前项目 / transcript metadata：

```text
originalCwd
worktreePath
worktreeName
worktreeBranch
originalBranch
originalHeadCommit
sessionId
tmuxSessionName
hookBased
```

`loadTranscriptFile(...)` 会读取 `worktree-state` entry，恢复出 session 对应的 worktree 状态。

这说明恢复不是“重新猜目录在哪里”，而是：

> session transcript 里要保存这条 worktree lane 的状态，resume 时才能回到正确执行边界。

## 4. 五个生命周期节点

Day 19 产出项要求至少覆盖：创建、进入、保留、移除、恢复。

可以这样画：

```text
CREATE
  task_create
  worktree_create
  bind task_id <-> worktree
  status=active

ENTER
  switch cwd to worktree path
  save current worktree session
  clear cwd-dependent caches

KEEP
  exit to original cwd
  preserve worktree path and branch
  worktree_state=kept
  task may remain completed or in_progress

REMOVE
  verify no uncommitted files / extra commits, or require discard confirmation
  remove worktree path
  worktree_state=removed
  optionally complete task

RESTORE
  load worktree-state from transcript/session storage
  restore current worktree session
  continue with correct cwd and lane metadata
```

一句话压缩：

> **worktree lane 的生命周期不是“建目录 / 删目录”，而是 create / enter / keep / remove / restore。**

## 5. task / runtime task / worktree lane 三层关系

| 对象 | 它是什么 | 生命周期 | Day 19 中的关键字段 |
| --- | --- | --- | --- |
| `task` | 工作目标 | pending -> in_progress -> completed | `id / status / owner / worktree_state` |
| `runtime task` | 当前执行槽位 | running -> done / killed / idle | `taskId / status / abortController` |
| `worktree lane` | 隔离执行目录 | active -> kept / removed / restored | `path / branch / task_id / closeout` |

一个常见组合是：

```text
task #12: completed
runtime task: finished
worktree lane: kept
```

这不是矛盾。
它表示工作目标完成了，执行进程结束了，但目录还保留用于 review 或后续排查。

## 6. 参考实现里最值得看的产品化细节

### 6.1 `EnterWorktreeTool.ts`

最值得看的是 session 切换：

- 不能重复进入当前 session 创建的 worktree
- 从 worktree 内调用时会先回 canonical root
- 创建 worktree 后直接 `chdir`
- 保存 worktree state
- 清除 system prompt / memory / plans directory 缓存

这说明进入 worktree 会影响后续 prompt 和工具行为。

### 6.2 `ExitWorktreeTool.ts`

最值得看的是 remove 前的安全检查：

- 无法验证状态时拒绝 remove
- 有 uncommitted files 或额外 commits 时拒绝 remove
- 需要 `discard_changes: true` 才允许继续
- keep/remove 都会恢复 original cwd

这说明 closeout 不是普通删除，而是有安全门的收尾决策。

### 6.3 `utils/worktree.ts`

这里最值得看的是三类能力：

- session worktree：`createWorktreeForSession / keepWorktree / cleanupWorktree`
- agent worktree：`createAgentWorktree / removeAgentWorktree`
- stale cleanup：只清理匹配临时模式且安全检查通过的旧 worktree

这说明真实系统里不只有用户显式 EnterWorktree，还有 agent / workflow 临时 worktree。

### 6.4 `sessionStorage.ts`

`saveWorktreeState(...)` 和 `loadTranscriptFile(...)` 说明：

- worktree state 是 session metadata
- resume 时要能从 transcript 找回
- 这不是单纯的当前进程全局变量

## 7. 初学者最容易犯的错

### 7.1 把 worktree 当 task

worktree 是目录车道，不是工作目标。

### 7.2 task 绑定了 worktree，但命令还在主目录执行

隔离执行的关键是 `cwd=worktree_path`。

### 7.3 删除前不检查改动

这会丢工作。
remove 应该 fail-closed。

### 7.4 只保存 path，不保存 session state

resume 需要知道 original cwd、branch、head commit、session id 等信息。

### 7.5 没有 closeout reason

保留或移除都应该说清原因。
否则后续排查只能猜。

## 8. 今天的输出物该怎么写

学习计划给 Day 19 的产出要求是：

> 整理 “task / runtime task / worktree lane” 三层关系，至少覆盖创建、进入、保留、移除、恢复五个生命周期节点。

建议输出物压成两部分：

### 第一部分：三层关系表

```text
task = work goal
runtime task = currently running execution slot
worktree lane = isolated directory lane
```

### 第二部分：五节点生命周期图

```text
create -> enter -> keep/remove -> restore
```

## 9. 读完 Day 19 应该能自己说清的话

如果这一天真的吃透了，你应该能不用看文档，自己说清下面这几句：

- task 管做什么，worktree 管在哪做
- runtime task 和 worktree lane 不是同一层
- 创建 worktree 后要同时更新 task record 和 worktree index
- 进入 worktree 会改变 cwd 和 session state
- keep 和 remove 是 closeout 的两个分支
- resume 要靠持久化 worktree state，而不是猜路径

## 10. 最低验证

```sh
cd learn-claude-code
python agents/s18_worktree_task_isolation.py
```

这个脚本会启动交互式 agent loop，并依赖 `MODEL_ID` 和可用的 Anthropic API 配置。
如果当前环境不适合真实调用模型，可以先做语法级验证：

```sh
python -m py_compile agents/s18_worktree_task_isolation.py
```

真实运行时重点观察三件事：

1. `task_create` 后 `worktree_create` 是否同时更新 task record 和 `.worktrees/index.json`
2. `worktree_run` 是否在 worktree path 里执行命令
3. `worktree_closeout` 的 keep/remove 是否写入 closeout、worktree_state 和 event log

最后把这句话记住就够了：

> **Day 19 的关键词是“隔离目录车道”。**
