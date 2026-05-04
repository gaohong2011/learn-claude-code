# Claude Code 4 周冲刺学习清单

> 用法：每天按“桥接文档 -> 当前仓库章节 -> `agents/*.py` -> 参考源码”的顺序推进，并完成“阅读项 / 产出项 / 验证命令 / 输出物链接”。
>
> 运行约定：
> - `python agents/...` 默认需要已配置 `ANTHROPIC_API_KEY` 和 `MODEL_ID`。
> - 如果当天只做静态学习，优先跑 `python -m pytest tests/test_agents_smoke.py -q`。
> - 参考仓库静态校验默认用 `cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`。
> - 四周主线严格对齐当前教学仓库四阶段：`s01-s06 -> s07-s11 -> s12-s14 -> s15-s19`。

## 第 1 周：单 Agent 主骨架（s01-s06）

### Day 1 架构总览与最小 Loop
- [x] 阅读项：`README-zh.md`；`docs/zh/s00-architecture-overview.md`；`docs/zh/s00b-one-request-lifecycle.md`；`docs/zh/s01-the-agent-loop.md`；`agents/s01_agent_loop.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/01-agent-loop.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:225-328,391-500,631-792`；`/Users/hong.gao/python/src/claude-code-codex/src/query/deps.ts:1-40`
- [x] 产出项：画一张“用户输入 -> query state -> 模型响应 -> tool_result -> 下一轮”的最小闭环图，并补 5 句解释单 agent 为什么必须长出 query control plane。
- [x] 验证命令：`python agents/s01_agent_loop.py`
- [x] 输出物链接：[s01-day1-query-loop-output.md](./s01-day1-query-loop-output.md)

### Day 2 Tool Control Plane 与执行面
- [x] 阅读项：`docs/zh/s02-tool-use.md`；`docs/zh/s02a-tool-control-plane.md`；`docs/zh/s02b-tool-execution-runtime.md`；`agents/s02_tool_use.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/05-tool-skill-system.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-257,260-387`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/toolOrchestration.ts:8-188`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/StreamingToolExecutor.ts:53-145,265-521`
- [x] 产出项：写 1 页“tool pool -> permission gate -> execution runtime -> tool_result 回流”的调用链笔记，明确串行/并行分组条件。
- [x] 验证命令：`python agents/s02_tool_use.py`
- [x] 输出物链接：[s02-day2-tool-control-plane-output.md](./s02-day2-tool-control-plane-output.md)

### Day 3 会话规划与一次性委派
- [x] 阅读项：`docs/zh/s03-todo-write.md`；`docs/zh/s04-subagent.md`；`agents/s03_todo_write.py`；`agents/s04_subagent.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TodoWriteTool/TodoWriteTool.ts:31-115`；`/Users/hong.gao/python/src/claude-code-codex/docs/04-multi-agent-coordinator.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:110-196,239-356,567-670,736-846`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/runAgent.ts:103-225,256-420,930-1037`
- [x] 产出项：做一张 “Todo / Plan / Subagent” 对比表，维度至少包含状态存放位置、上下文边界、生命周期、返回结果方式。
- [x] 验证命令：`python agents/s04_subagent.py`
- [x] 输出物链接：[s03-day3-todo-plan-subagent-output.md](./s03-day3-todo-plan-subagent-output.md)

### Day 4 Skill 发现、按需加载与输入注入
- [x] 阅读项：`docs/zh/s05-skill-loading.md`；`docs/zh/s00e-reference-module-map.md`；`agents/s05_skill_loading.py`；`/Users/hong.gao/python/src/claude-code-codex/src/skills/loadSkillsDir.ts:78-270,407-638,861-1063`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:333-352,444-577`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts:41-123`
- [x] 产出项：画出 “发现技能 -> 生成描述 -> 按需展开正文 -> 注入输入管道” 的 4 步图，并写一段为什么 `s05` 必须早于 `s06`。
- [x] 验证命令：`python agents/s05_skill_loading.py`
- [x] 输出物链接：[s05-day4-skill-loading-output.md](./s05-day4-skill-loading-output.md)

### Day 5 Compact、Transition 与阶段 1 复盘
- [x] 阅读项：`docs/zh/s06-context-compact.md`；`docs/zh/s00c-query-transition-model.md`；`agents/s06_context_compact.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/02-context-management.md`；`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:138-164,226-305,422-446`；`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:33-93,160-351`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:137-232,421-498,536-924,960-1001`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/tokens.ts:46-79,226-261`
- [x] 产出项：整理 `skill loading / tool result budget / microcompact / autocompact / transition reason` 的关系图，并写 1 页“单 agent 主骨架已成立”的阶段复盘。
- [x] 验证命令：`python agents/s06_context_compact.py`
- [x] 输出物链接：[s06-day5-compact-transition-stage1-review-output.md](./s06-day5-compact-transition-stage1-review-output.md)、[s06-day5-skill-budget-compact-transition-map.md](./s06-day5-skill-budget-compact-transition-map.md)

## 第 2 周：控制面加固（s07-s11）

### Day 6 Permission Gate 与 Sandbox
- [x] 阅读项：`docs/zh/s07-permission-system.md`；`docs/zh/s00a-query-control-plane.md`；`agents/s07_permission_system.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/06-sandbox-system.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/permissions/permissions.ts:473-945,1071-1477`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/interactiveHandler.ts:57-430`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/BashTool/shouldUseSandbox.ts:21-153`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sandbox/sandbox-adapter.ts:99-404,422-479,532-927`
- [x] 产出项：画出“deny rules -> mode -> allow rules -> ask user -> sandbox”决策树，并单独列出 Bash 为什么要有独立安全通道。
- [x] 验证命令：`python agents/s07_permission_system.py`
- [x] 学习指南：[day6-permission-gate-and-sandbox-guide.md](./day6-permission-gate-and-sandbox-guide.md) 输出物链接：[s07-day6-permission-gate-sandbox-output.md](./s07-day6-permission-gate-sandbox-output.md)

### Day 7 Hook 事件与侧车扩展
- [x] 阅读项：`docs/zh/s08-hook-system.md`；`agents/s08_hook_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/hookEvents.ts:61-188`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/AsyncHookRegistry.ts:30-309`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/sessionHooks.ts:68-437`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/postSamplingHooks.ts:31-70`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/registerSkillHooks.ts:20-64`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/toolExecution.ts:492-760`
- [x] 产出项：写一页 Hook 事件矩阵，至少覆盖“触发时机 / 输入载荷 / 返回语义 / 是否影响主循环推进”四列。
- [x] 验证命令：`python agents/s08_hook_system.py`
- [x] 学习指南：[day7-hook-events-and-sidecar-guide.md](./day7-hook-events-and-sidecar-guide.md) 输出物链接：[s08-day7-hook-events-sidecar-output.md](./s08-day7-hook-events-sidecar-output.md)

### Day 8 Memory、Transcript 与 CLAUDE.md 链
- [x] 阅读项：`docs/zh/s09-memory-system.md`；`docs/zh/data-structures.md`；`agents/s09_memory_system.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/03-memory-persistence.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:976-1128,1408-1576,2069-2399,3306-3472`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/claudemd.ts:254-451,618-790,1153-1479`；`/Users/hong.gao/python/src/claude-code-codex/src/services/SessionMemory/sessionMemory.ts:134-357,387-495`；`/Users/hong.gao/python/src/claude-code-codex/src/services/extractMemories/extractMemories.ts:171-232,296-510,598-615`
- [x] 产出项：补一张“transcript / session memory / auto memory / CLAUDE.md”边界图，并写出哪些信息不该被放进 memory。
- [x] 验证命令：`python agents/s09_memory_system.py`
- [x] 学习指南：[day8-memory-transcript-claudemd-guide.md](./day8-memory-transcript-claudemd-guide.md) 输出物链接：[s09-day8-memory-transcript-claudemd-output.md](./s09-day8-memory-transcript-claudemd-output.md)

### Day 9 System Prompt Pipeline
- [x] 阅读项：`docs/zh/s10-system-prompt.md`；`docs/zh/s10a-message-prompt-pipeline.md`；`agents/s10_system_prompt.py`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:127-167,444-651,760-860`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/systemPromptSections.ts:20-65`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts:41-123`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/attachments.ts:744-1047,1711-1793,2197-2380,2548-2689`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/api.ts:437-479,566-718`
- [x] 产出项：写一份 system prompt 组装清单，至少列出“核心身份、工具说明、skills、memory、CLAUDE.md、动态环境”六类来源。
- [x] 验证命令：`python agents/s10_system_prompt.py`
- [x] 学习指南：[day9-system-prompt-pipeline-guide.md](./day9-system-prompt-pipeline-guide.md) 输出物链接：[s10-day9-system-prompt-pipeline-output.md](./s10-day9-system-prompt-pipeline-output.md)

### Day 10 Error Recovery 与阶段 2 复盘
- [x] 阅读项：`docs/zh/s11-error-recovery.md`；`docs/zh/s00c-query-transition-model.md`；`agents/s11_error_recovery.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/01-agent-loop.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:950-1034,1152-1458`；`/Users/hong.gao/python/src/claude-code-codex/src/query/tokenBudget.ts:13-93`；`/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:764-921,1358-1686,3087-3182,3402-3588`
- [x] 产出项：画出“continue / compact / backoff / fail”恢复状态机，并写 1 页“高完成度单 agent 的控制面已经成立”总结。
- [x] 验证命令：`python -m pytest tests/test_agents_smoke.py -q`
- [x] 学习指南：[day10-error-recovery-stage2-review-guide.md](./day10-error-recovery-stage2-review-guide.md) 输出物链接：[s11-day10-error-recovery-stage2-review-output.md](./s11-day10-error-recovery-stage2-review-output.md)

## 第 3 周：持久任务层与运行时层（s12-s14）

### Day 11 Durable Task Graph
- [x] 阅读项：`docs/zh/s12-task-system.md`；`docs/zh/data-structures.md`；`agents/s12_task_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/Task.ts:27-125`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`；`/Users/hong.gao/python/src/claude-code-codex/src/commands/tasks/tasks.tsx:1-7`
- [x] 产出项：整理 durable task 的状态机，至少包含创建、阻塞、解锁、认领、完成、失败六种状态。
- [x] 验证命令：`python agents/s12_task_system.py`
- [x] 学习指南：[day11-durable-task-graph-guide.md](./day11-durable-task-graph-guide.md) 输出物链接：[s12-day11-durable-task-graph-output.md](./s12-day11-durable-task-graph-output.md)

### Day 12 Runtime Task 与后台执行
- [x] 阅读项：`docs/zh/s13-background-tasks.md`；`docs/zh/s13a-runtime-task-model.md`；`agents/s13_background_tasks.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:39-180,180-259,293-515`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:50-149,197-270,412-657`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:124-225,386-808`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:78-121`
- [x] 产出项：做一张 runtime task 类型族地图，明确 `local_bash / local_agent / remote_agent / teammate` 各自属于什么执行槽位。
- [x] 验证命令：`python agents/s13_background_tasks.py`
- [x] 学习指南：[day12-runtime-task-background-execution-guide.md](./day12-runtime-task-background-execution-guide.md) 输出物链接：[s13-day12-runtime-task-family-map-output.md](./s13-day12-runtime-task-family-map-output.md)

### Day 13 Scheduler 与未来触发
- [x] 阅读项：`docs/zh/s14-cron-scheduler.md`；`agents/s14_cron_scheduler.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronCreateTool.ts:56-157`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronListTool.ts:37-97`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronDeleteTool.ts:35-95`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/useScheduledTasks.ts:40-139`
- [x] 产出项：画出“schedule record -> checker -> notification -> 主循环回注”的链路图，并写清 `schedule` 和 `runtime task` 为什么不是同一层。
- [x] 验证命令：`python agents/s14_cron_scheduler.py`
- [x] 学习指南：[day13-scheduler-future-trigger-guide.md](./day13-scheduler-future-trigger-guide.md) 输出物链接：[s14-day13-scheduler-future-trigger-output.md](./s14-day13-scheduler-future-trigger-output.md)

### Day 14 Task / Runtime / Notification 三层联读
- [x] 阅读项：`docs/zh/s00b-one-request-lifecycle.md`；`docs/zh/s13a-runtime-task-model.md`；`docs/zh/entity-map.md`；`agents/s_full.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalMainSessionTask.ts:94-224,338-479`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/stopTask.ts:10-100`；`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:1737-1938,2663-2863,4047-4120`
- [x] 产出项：写 1 页“durable task / runtime task / notification / transcript”四层边界说明，要求每层都给出 1 个真实例子。
- [ ] 验证命令：`python -m pytest tests/test_s_full_background.py -q`（1 passed in 0.01s）
- [x] 学习指南：[day14-task-runtime-notification-guide.md](./day14-task-runtime-notification-guide.md) 输出物链接：[s14a-day14-task-runtime-notification-transcript-output.md](./s14a-day14-task-runtime-notification-transcript-output.md)

### Day 15 阶段 3 复盘与小型原型
- [x] 阅读项：`docs/zh/s00d-chapter-order-rationale.md`；`docs/zh/s00e-reference-module-map.md`；`agents/s12_task_system.py`；`agents/s13_background_tasks.py`；`agents/s14_cron_scheduler.py`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:1478-1903`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`
- [ ] 产出项：实现一个最小 “task + runtime slot + schedule queue” 原型，或写一份等价设计文档，必须包含数据结构、状态机和触发链。
- [ ] 验证命令：`python -m pytest tests/test_agents_smoke.py -q`（23 passed）
- [x] 学习指南：[day15-stage3-review-mini-prototype-guide.md](./day15-stage3-review-mini-prototype-guide.md) 输出物链接：[s14b-day15-task-runtime-schedule-prototype-output.md](./s14b-day15-task-runtime-schedule-prototype-output.md)

## 第 4 周：多 Agent 平台与外部能力总线（s15-s19）

### Day 16 Agent Teams 与消息总线
- [ ] 阅读项：`docs/zh/s15-agent-teams.md`；`docs/zh/team-task-lane-model.md`；`agents/s15_agent_teams.py`；`/Users/hong.gao/python/src/claude-code-codex/docs/04-multi-agent-coordinator.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TeamCreateTool/TeamCreateTool.ts:74-220`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/shared/spawnMultiAgent.ts:72-294,760-1093`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:149-266,520-760`；`/Users/hong.gao/python/src/claude-code-codex/src/context/mailbox.tsx:8-37`
- [ ] 产出项：画出 teammate 生命周期图，明确 team member、mailbox、runtime task、leader 之间的关系。
- [ ] 验证命令：`python agents/s15_agent_teams.py`
- [ ] 学习指南：[day16-agent-teams-message-bus-guide.md](./day16-agent-teams-message-bus-guide.md) 输出物链接：[s15-day16-agent-teams-message-bus-output.md](./s15-day16-agent-teams-message-bus-output.md)

### Day 17 Team Protocols 与 Coordinator 纪律
- [ ] 阅读项：`docs/zh/s16-team-protocols.md`；`docs/zh/team-task-lane-model.md`；`agents/s16_team_protocols.py`；`/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:36-111,111-194`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:268-518,741-917`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TaskStopTool/TaskStopTool.ts:39-131`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/mailbox.ts:19-73`
- [ ] 产出项：写一页“自由文本消息 vs 协议消息”的对比说明，必须明确 `request_id` 为什么是团队协作的硬边界。
- [ ] 验证命令：`python agents/s16_team_protocols.py`
- [ ] 学习指南：[day17-team-protocols-coordinator-guide.md](./day17-team-protocols-coordinator-guide.md) 输出物链接：[s16-day17-team-protocols-coordinator-output.md](./s16-day17-team-protocols-coordinator-output.md)

### Day 18 Autonomous Agents 与权限分层
- [ ] 阅读项：`docs/zh/s17-autonomous-agents.md`；`docs/zh/team-task-lane-model.md`；`agents/s17_autonomous_agents.py`；`/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:80-194`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/coordinatorHandler.ts:26-65`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40-159`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:519-689,883-1040,1544-1552`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/spawnInProcess.ts:104-227`
- [ ] 产出项：画出 “任务认领 -> worker 启动 -> 权限桥接 -> 结果回收 -> 恢复续行” 的自治协作图。
- [ ] 验证命令：`python agents/s17_autonomous_agents.py`
- [ ] 学习指南：[day18-autonomous-agents-permission-guide.md](./day18-autonomous-agents-permission-guide.md) 输出物链接：[s17-day18-autonomous-agents-permission-output.md](./s17-day18-autonomous-agents-permission-output.md)

### Day 19 Worktree 执行车道
- [ ] 阅读项：`docs/zh/s18-worktree-task-isolation.md`；`docs/zh/team-task-lane-model.md`；`agents/s18_worktree_task_isolation.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts:52-127`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts:79-122,148-329`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/worktree.ts:158-235,702-813,902-1180`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/worktreeModeEnabled.ts:1-11`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:2889-2914,3472-3835`
- [ ] 产出项：整理 “task / runtime task / worktree lane” 三层关系，至少覆盖创建、进入、保留、移除、恢复五个生命周期节点。
- [ ] 验证命令：`python agents/s18_worktree_task_isolation.py`
- [ ] 学习指南：[day19-worktree-execution-lane-guide.md](./day19-worktree-execution-lane-guide.md) 输出物链接：[s18-day19-worktree-execution-lane-output.md](./s18-day19-worktree-execution-lane-output.md)

### Day 20 MCP、Plugin 与总复刻
- [ ] 阅读项：`docs/zh/s19-mcp-plugin.md`；`docs/zh/s19a-mcp-capability-layers.md`；`agents/s19_mcp_plugin.py`；`/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/mcp.ts:36-197`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/config.ts:62-223,625-888,1071-1297,1470-1553`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/client.ts:595-744,1743-2033,2226-2813,3029-3262`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/MCPConnectionManager.tsx:17-72`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-257,343-387`；`/Users/hong.gao/python/src/claude-code-codex/src/main.tsx:518-760,858-930`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/plugins/pluginLoader.ts:266-365,492-718,1147-1360,1888-2237,3009-3232`
- [ ] 产出项：实现一个 500-1000 行 mini Claude Code 原型，或写一份等价蓝图文档，最少覆盖 `loop / tools / permissions / compact / task / teams / worktree / MCP` 八块，并标出哪些能力来自当前教学仓库、哪些来自参考源码映射。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 学习指南：[day20-mcp-plugin-capability-bus-guide.md](./day20-mcp-plugin-capability-bus-guide.md) 输出物链接：[s19-day20-mcp-plugin-capability-bus-output.md](./s19-day20-mcp-plugin-capability-bus-output.md)

## 每周固定复盘模板

- [ ] 本周我真正跑过的脚本：
- [ ] 本周我能完整复述的调用链：
- [ ] 本周我已经能在两个仓库里对齐的模块簇：
- [ ] 本周最模糊的 3 个概念：
- [ ] 下周开始前需要补的源码段：
- [ ] 周总结链接：[]()
