# Day 19 输出物：`task / runtime task / worktree lane` 三层关系

Day 19 最该压成一页纸的，不是 `git worktree` 命令表，而是下面这条边界：

> **task 是工作目标，runtime task 是当前执行槽位，worktree lane 是隔离执行目录。**

## 1. 三层关系表

| 层 | 回答的问题 | 典型状态 | 典型字段 |
| --- | --- | --- | --- |
| `task` | 要做什么、谁负责、是否完成 | `pending / in_progress / completed` | `id / subject / owner / status` |
| `runtime task` | 当前谁正在跑、能不能停止 | `running / done / killed / idle` | `taskId / type / abortController` |
| `worktree lane` | 在哪做、目录是否保留 | `active / kept / removed / restored` | `path / branch / task_id / closeout` |

最短判断法：

```text
task         = work goal
runtime task = live execution slot
worktree lane = isolated directory lane
```

## 2. 五个生命周期节点

```text
CREATE
  ->
task_create
worktree_create
bind task_id <-> worktree
worktree_state=active

ENTER
  ->
switch cwd to worktree path
save current worktree session
clear cwd-dependent prompt / memory caches

KEEP
  ->
exit to original cwd
preserve worktree directory and branch
worktree_state=kept
record closeout reason

REMOVE
  ->
verify no changes or require discard confirmation
remove worktree directory
worktree_state=removed
optionally complete task

RESTORE
  ->
load worktree-state from transcript / session storage
restore current worktree session
continue in correct execution lane
```

## 3. 为什么三层不能混

一个真实状态可能是：

```text
task #12: completed
runtime task: finished
worktree lane: kept
```

这不是矛盾。

它表示：

- 工作目标已经完成
- 当前执行进程已经结束
- 隔离目录仍然保留，方便 review 或后续排查

如果把三层混成一个状态，就无法表达这种情况。

## 4. 最小链路图

```text
task created
  ->
teammate claims task
  ->
worktree lane created for task
  ->
session enters lane
  ->
commands run with cwd=worktree_path
  ->
task progresses
  ->
closeout decision
  ->
keep for review or remove after safe checks
  ->
session can restore lane metadata later
```

## 5. 硬边界

```text
task 管“做什么”
runtime task 管“现在谁在跑”
worktree lane 管“在哪做且不互相踩目录”
```

删除 worktree 前还要守住安全边界：

> **无法确认没有未提交改动或额外 commit 时，不要删除。**

## 6. 一句记忆钩子

> **Day 18 让 teammate 自己接活；Day 19 给每份并行工作分配不会互踩的目录车道。**
