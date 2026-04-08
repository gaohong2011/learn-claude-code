# Claude Code 4 周冲刺学习清单（高优先级版）

> 详细版：`claude-code-4-week-sprint-checklist.md`
>
> 用法：
> - 第一遍只做“必读 + 当天产出 + 最低验证”。
> - 第二遍再补“选读”。
> - “跳读”默认只在你自己实现对应机制、排障、或准备写长文档时再翻。
>
> 压缩原则：
> - `必读`：建立机制主心智所需的最小集合。
> - `选读`：补边界、补产品化实现、补跨模块连接。
> - `跳读`：第一遍不影响建模，但第二遍有价值。

## 第 1 周：单 Agent 主骨架（s01-s06）

### Day 1 架构总览与最小 Loop
- [ ] 必读：`docs/zh/s00b-one-request-lifecycle.md`；`docs/zh/s01-the-agent-loop.md`；`agents/s01_agent_loop.py`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:225-328,391-500`
- [ ] 选读：`README-zh.md`；`docs/zh/s00-architecture-overview.md`；`/Users/hong.gao/python/src/claude-code-codex/docs/01-agent-loop.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:631-792`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/query/deps.ts:1-40`
- [ ] 当天产出：画出“用户输入 -> query state -> 模型响应 -> tool_result -> 下一轮”的最小闭环图。
- [ ] 最低验证：`python agents/s01_agent_loop.py`

### Day 2 Tool Control Plane 与执行面
- [ ] 必读：`docs/zh/s02-tool-use.md`；`docs/zh/s02a-tool-control-plane.md`；`agents/s02_tool_use.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-257,260-387`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/toolOrchestration.ts:8-188`
- [ ] 选读：`docs/zh/s02b-tool-execution-runtime.md`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/StreamingToolExecutor.ts:53-145`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/StreamingToolExecutor.ts:265-521`
- [ ] 当天产出：写 1 页“tool pool -> permission gate -> execution runtime -> tool_result 回流”的调用链笔记。
- [ ] 最低验证：`python agents/s02_tool_use.py`

### Day 3 会话规划与一次性委派
- [ ] 必读：`docs/zh/s03-todo-write.md`；`docs/zh/s04-subagent.md`；`agents/s03_todo_write.py`；`agents/s04_subagent.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TodoWriteTool/TodoWriteTool.ts:31-115`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:110-196,239-356`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/runAgent.ts:103-225`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/docs/04-multi-agent-coordinator.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/runAgent.ts:256-420`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/AgentTool.tsx:567-670,736-846`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/runAgent.ts:930-1037`
- [ ] 当天产出：做一张 “Todo / Plan / Subagent” 对比表。
- [ ] 最低验证：`python agents/s04_subagent.py`

### Day 4 Skill 发现、按需加载与输入注入
- [ ] 必读：`docs/zh/s05-skill-loading.md`；`agents/s05_skill_loading.py`；`/Users/hong.gao/python/src/claude-code-codex/src/skills/loadSkillsDir.ts:78-270,407-638`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:444-577`
- [ ] 选读：`docs/zh/s00e-reference-module-map.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts:41-123`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/skills/loadSkillsDir.ts:861-1063`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:333-352`
- [ ] 当天产出：画出 “发现技能 -> 生成描述 -> 按需展开正文 -> 注入输入管道” 的 4 步图。
- [ ] 最低验证：`python agents/s05_skill_loading.py`

### Day 5 Compact、Transition 与阶段 1 复盘
- [ ] 必读：`docs/zh/s06-context-compact.md`；`agents/s06_context_compact.py`；`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:226-305`；`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/autoCompact.ts:160-351`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:536-924`
- [ ] 选读：`docs/zh/s00c-query-transition-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:138-164`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:137-232,421-498`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/tokens.ts:46-79,226-261`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/services/compact/microCompact.ts:422-446`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/toolResultStorage.ts:960-1001`
- [ ] 当天产出：整理 `skill loading / tool result budget / microcompact / autocompact / transition reason` 的关系图。
- [ ] 最低验证：`python agents/s06_context_compact.py`

## 第 2 周：控制面加固（s07-s11）

### Day 6 Permission Gate 与 Sandbox
- [ ] 必读：`docs/zh/s07-permission-system.md`；`agents/s07_permission_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/permissions/permissions.ts:473-945,1071-1477`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/interactiveHandler.ts:57-430`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/BashTool/shouldUseSandbox.ts:21-153`
- [ ] 选读：`docs/zh/s00a-query-control-plane.md`；`/Users/hong.gao/python/src/claude-code-codex/docs/06-sandbox-system.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sandbox/sandbox-adapter.ts:99-404,422-479`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/sandbox/sandbox-adapter.ts:532-927`
- [ ] 当天产出：画出“deny rules -> mode -> allow rules -> ask user -> sandbox”决策树。
- [ ] 最低验证：`python agents/s07_permission_system.py`

### Day 7 Hook 事件与侧车扩展
- [ ] 必读：`docs/zh/s08-hook-system.md`；`agents/s08_hook_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/hookEvents.ts:61-188`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/AsyncHookRegistry.ts:30-309`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/sessionHooks.ts:68-437`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/postSamplingHooks.ts:31-70`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/hooks/registerSkillHooks.ts:20-64`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/toolExecution.ts:492-760`
- [ ] 当天产出：写一页 Hook 事件矩阵。
- [ ] 最低验证：`python agents/s08_hook_system.py`

### Day 8 Memory、Transcript 与 CLAUDE.md 链
- [ ] 必读：`docs/zh/s09-memory-system.md`；`agents/s09_memory_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:976-1128,2069-2399`；`/Users/hong.gao/python/src/claude-code-codex/src/services/SessionMemory/sessionMemory.ts:134-357,387-495`
- [ ] 选读：`docs/zh/data-structures.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/claudemd.ts:254-451,618-790`；`/Users/hong.gao/python/src/claude-code-codex/src/services/extractMemories/extractMemories.ts:296-510`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:1408-1576,3306-3472`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/claudemd.ts:1153-1479`；`/Users/hong.gao/python/src/claude-code-codex/src/services/extractMemories/extractMemories.ts:171-232,598-615`
- [ ] 当天产出：补一张“transcript / session memory / auto memory / CLAUDE.md”边界图。
- [ ] 最低验证：`python agents/s09_memory_system.py`

### Day 9 System Prompt Pipeline
- [ ] 必读：`docs/zh/s10-system-prompt.md`；`docs/zh/s10a-message-prompt-pipeline.md`；`agents/s10_system_prompt.py`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:444-651`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/systemPromptSections.ts:20-65`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts:41-123`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/attachments.ts:744-1047,1711-1793`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/api.ts:566-718`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:127-167,760-860`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/attachments.ts:2197-2380,2548-2689`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/api.ts:437-479`
- [ ] 当天产出：写一份 system prompt 组装清单。
- [ ] 最低验证：`python agents/s10_system_prompt.py`

### Day 10 Error Recovery 与阶段 2 复盘
- [ ] 必读：`docs/zh/s11-error-recovery.md`；`agents/s11_error_recovery.py`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:950-1034,1152-1458`；`/Users/hong.gao/python/src/claude-code-codex/src/query/tokenBudget.ts:13-93`
- [ ] 选读：`docs/zh/s00c-query-transition-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:764-921,1358-1686`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:3087-3182,3402-3588`
- [ ] 当天产出：画出“continue / compact / backoff / fail”恢复状态机。
- [ ] 最低验证：`python -m pytest tests/test_agents_smoke.py -q`

## 第 3 周：持久任务层与运行时层（s12-s14）

### Day 11 Durable Task Graph
- [ ] 必读：`docs/zh/s12-task-system.md`；`agents/s12_task_system.py`；`/Users/hong.gao/python/src/claude-code-codex/src/Task.ts:27-125`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`
- [ ] 选读：`docs/zh/data-structures.md`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/commands/tasks/tasks.tsx:1-7`
- [ ] 当天产出：整理 durable task 的状态机。
- [ ] 最低验证：`python agents/s12_task_system.py`

### Day 12 Runtime Task 与后台执行
- [ ] 必读：`docs/zh/s13-background-tasks.md`；`docs/zh/s13a-runtime-task-model.md`；`agents/s13_background_tasks.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:39-180,293-515`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:50-149,412-657`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:124-225,386-808`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalShellTask/LocalShellTask.tsx:180-259`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalAgentTask/LocalAgentTask.tsx:197-270`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/InProcessTeammateTask/types.ts:78-121`
- [ ] 当天产出：做一张 runtime task 类型族地图。
- [ ] 最低验证：`python agents/s13_background_tasks.py`

### Day 13 Scheduler 与未来触发
- [ ] 必读：`docs/zh/s14-cron-scheduler.md`；`agents/s14_cron_scheduler.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronCreateTool.ts:56-157`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/useScheduledTasks.ts:40-139`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronListTool.ts:37-97`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/tools/ScheduleCronTool/CronDeleteTool.ts:35-95`
- [ ] 当天产出：画出“schedule record -> checker -> notification -> 主循环回注”的链路图。
- [ ] 最低验证：`python agents/s14_cron_scheduler.py`

### Day 14 Task / Runtime / Notification 三层联读
- [ ] 必读：`docs/zh/entity-map.md`；`docs/zh/s13a-runtime-task-model.md`；`agents/s_full.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/LocalMainSessionTask.ts:94-224,338-479`；`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:1737-1938,2663-2863`
- [ ] 选读：`docs/zh/s00b-one-request-lifecycle.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/stopTask.ts:10-100`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:4047-4120`
- [ ] 当天产出：写 1 页“durable task / runtime task / notification / transcript”四层边界说明。
- [ ] 最低验证：`python -m pytest tests/test_s_full_background.py -q`

### Day 15 阶段 3 复盘与小型原型
- [ ] 必读：`docs/zh/s00d-chapter-order-rationale.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:1478-1903`
- [ ] 选读：`docs/zh/s00e-reference-module-map.md`；`agents/s12_task_system.py`；`agents/s13_background_tasks.py`
- [ ] 跳读：`agents/s14_cron_scheduler.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tasks/types.ts:1-46`
- [ ] 当天产出：实现一个最小 “task + runtime slot + schedule queue” 原型，或写一份等价设计文档。
- [ ] 最低验证：`python -m pytest tests/test_agents_smoke.py -q`

## 第 4 周：多 Agent 平台与外部能力总线（s15-s19）

### Day 16 Agent Teams 与消息总线
- [ ] 必读：`docs/zh/s15-agent-teams.md`；`agents/s15_agent_teams.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TeamCreateTool/TeamCreateTool.ts:74-220`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/shared/spawnMultiAgent.ts:72-294`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:149-266`
- [ ] 选读：`docs/zh/team-task-lane-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/shared/spawnMultiAgent.ts:760-1093`；`/Users/hong.gao/python/src/claude-code-codex/src/context/mailbox.tsx:8-37`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:520-760`
- [ ] 当天产出：画出 teammate 生命周期图。
- [ ] 最低验证：`python agents/s15_agent_teams.py`

### Day 17 Team Protocols 与 Coordinator 纪律
- [ ] 必读：`docs/zh/s16-team-protocols.md`；`agents/s16_team_protocols.py`；`/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:36-111,111-194`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:268-518`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/mailbox.ts:19-73`
- [ ] 选读：`docs/zh/team-task-lane-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/TaskStopTool/TaskStopTool.ts:39-131`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:741-917`
- [ ] 当天产出：写一页“自由文本消息 vs 协议消息”的对比说明。
- [ ] 最低验证：`python agents/s16_team_protocols.py`

### Day 18 Autonomous Agents 与权限分层
- [ ] 必读：`docs/zh/s17-autonomous-agents.md`；`agents/s17_autonomous_agents.py`；`/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:80-194`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40-159`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/spawnInProcess.ts:104-227`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:519-689`
- [ ] 选读：`docs/zh/team-task-lane-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/coordinatorHandler.ts:26-65`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:883-1040`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/swarm/inProcessRunner.ts:1544-1552`
- [ ] 当天产出：画出 “任务认领 -> worker 启动 -> 权限桥接 -> 结果回收 -> 恢复续行” 的自治协作图。
- [ ] 最低验证：`python agents/s17_autonomous_agents.py`

### Day 19 Worktree 执行车道
- [ ] 必读：`docs/zh/s18-worktree-task-isolation.md`；`agents/s18_worktree_task_isolation.py`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts:52-127`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts:148-329`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/worktree.ts:158-235,902-1180`
- [ ] 选读：`docs/zh/team-task-lane-model.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts:79-122`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/worktree.ts:702-813`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:3472-3835`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/utils/worktreeModeEnabled.ts:1-11`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:2889-2914`
- [ ] 当天产出：整理 “task / runtime task / worktree lane” 三层关系。
- [ ] 最低验证：`python agents/s18_worktree_task_isolation.py`

### Day 20 MCP、Plugin 与总复刻
- [ ] 必读：`docs/zh/s19-mcp-plugin.md`；`docs/zh/s19a-mcp-capability-layers.md`；`agents/s19_mcp_plugin.py`；`/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/mcp.ts:36-197`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/config.ts:62-223,625-888`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/client.ts:2226-2813`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-257,343-387`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/plugins/pluginLoader.ts:492-718,1888-2237`
- [ ] 选读：`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/client.ts:595-744,1743-2033,3029-3262`；`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/MCPConnectionManager.tsx:17-72`；`/Users/hong.gao/python/src/claude-code-codex/src/main.tsx:518-760,858-930`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/plugins/pluginLoader.ts:266-365,1147-1360,3009-3232`
- [ ] 跳读：`/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/config.ts:1071-1297,1470-1553`
- [ ] 当天产出：实现一个 500-1000 行 mini Claude Code 原型，或写一份等价蓝图文档。
- [ ] 最低验证：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`

## 第一遍建议完成线

- [ ] 已完成 Day 1-10 的全部“必读 + 最低验证”。
- [ ] 已完成 Day 11-12 的“必读”，能复述 `task / runtime task / schedule` 三层边界。
- [ ] 已完成 Day 16、Day 18、Day 19、Day 20 的“必读”，能复述 `teams / autonomy / worktree / MCP` 四块怎样接回统一控制面。
- [ ] 已把所有“跳读”暂时收住，没有在第一遍里被大文件细节拖慢。

## 第二遍补强建议

- [ ] 先补所有“选读”，再回详细版挑具体源码段做深挖。
- [ ] 第二遍优先补 Day 6、Day 8、Day 10、Day 18、Day 20，这几天最容易在真实实现里踩边界。
- [ ] 只有在你要自己复刻 `sandbox / memory / worktree / MCP client` 时，再系统补“跳读”。
