# Claude Code 4 周冲刺学习清单

> 用法：每天按顺序完成“阅读项 / 产出项 / 验证命令 / 输出物链接”。`python agents/...` 默认需要已配置 `ANTHROPIC_API_KEY` 和 `MODEL_ID`；如果当天只做静态阅读，`learn-claude-code` 可改跑 `python -m pytest tests/test_agents_smoke.py -q`，`claude-code-codex` 可改跑 `cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`。

## 第 1 周：Agent Loop 与 Tool 基础

### Day 1 最小闭环
- [ ] 阅读项：`docs/en/s01-the-agent-loop.md`；`agents/s01_agent_loop.py:1-120, 81-101`；`/Users/hong.gao/python/src/claude-code-codex/docs/01-agent-loop.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:225-365, 631-729`
- [ ] 产出项：画出“用户输入 -> 模型响应 -> tool_result -> 下一轮”的首轮时序图，并写 5 句解释。
- [ ] 验证命令：`python agents/s01_agent_loop.py`
- [ ] 输出物链接：[]()

### Day 2 工具注册与调度
- [ ] 阅读项：`docs/en/s02-tool-use.md`；`agents/s02_tool_use.py:41-131, 94-111, 114-131`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-249, 269-325, 343-365`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/toolOrchestration.ts:19-188`
- [ ] 产出项：写一页“工具如何被注册、合并、分派、串并行执行”的调用链笔记。
- [ ] 验证命令：`python agents/s02_tool_use.py`
- [ ] 输出物链接：[]()

### Day 3 状态化工具与 Todo
- [ ] 阅读项：`docs/en/s03-todo-write.md`；`agents/s03_todo_write.py:51-192, 141-160, 163-192`；`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:682-831`
- [ ] 产出项：总结“本地状态如何进入工具调用上下文”，补一张 Todo 状态流转图。
- [ ] 验证命令：`python agents/s03_todo_write.py`
- [ ] 输出物链接：[]()

### Day 4 流式模型调用
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:734-1035`；`/Users/hong.gao/python/src/claude-code-codex/src/services/api/claude.ts:764-792, 830-920`；`/Users/hong.gao/python/src/claude-code-codex/src/services/tools/StreamingToolExecutor.ts:40-205, 265-320`
- [ ] 产出项：画出“流式 token、tool call、tool result、异常恢复”四条并发时间线。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 5 第一次闭环复盘
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/cli.tsx:34-260`；`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:2663-2805`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:247-365`
- [ ] 产出项：写一份“CLI -> REPL -> query -> tool executor”的 1 页路径说明，要求只保留关键函数名。
- [ ] 验证命令：`python -m pytest tests/test_agents_smoke.py -q`
- [ ] 输出物链接：[]()

## 第 2 周：Prompt、Context、Compact、Memory

### Day 6 System Prompt 组装
- [ ] 阅读项：`docs/en/s05-skill-loading.md`；`agents/s05_skill_loading.py:58-115, 166-208`；`/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts:444-577, 606-710, 758-760`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts:41-123`
- [ ] 产出项：写一份 system prompt 拼装清单，列出“固定提示、环境信息、技能内容、运行态注入”四类来源。
- [ ] 验证命令：`python agents/s05_skill_loading.py`
- [ ] 输出物链接：[]()

### Day 7 Compact 机制
- [ ] 阅读项：`docs/en/s06-context-compact.md`；`agents/s06_context_compact.py:68-127, 201-237`；`/Users/hong.gao/python/src/claude-code-codex/docs/02-context-management.md`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:391-605`
- [ ] 产出项：整理 `microcompact`、`autocompact`、tool result budget 三者关系，并写出触发条件。
- [ ] 验证命令：`python agents/s06_context_compact.py`
- [ ] 输出物链接：[]()

### Day 8 Context 来源
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/src/context.ts:116-188`；`/Users/hong.gao/python/src/claude-code-codex/src/screens/REPL.tsx:2528-2576`
- [ ] 产出项：画出“system context / user context / session state / cwd 信息”的来源图。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 9 Transcript 与记忆
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/docs/03-memory-persistence.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:128-225, 231-303, 320-360`
- [ ] 产出项：写一份 transcript JSONL 字段表，至少覆盖消息、tool use、压缩边界、subagent 元数据。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 10 第二次闭环复盘
- [ ] 阅读项：复读 `agents/s06_context_compact.py:68-127, 201-237`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:247-605`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:198-303`
- [ ] 产出项：写一份“压缩后为什么还能恢复会话”的说明，必须包含 `parentUuid`、compact boundary、重建顺序。
- [ ] 验证命令：`python -m pytest tests/test_agents_smoke.py -q`
- [ ] 输出物链接：[]()

## 第 3 周：Subagent、Task、Background、Coordinator

### Day 11 Subagent 最小版
- [ ] 阅读项：`docs/en/s04-subagent.md`；`agents/s04_subagent.py:104-168, 118-136, 139-168`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/AgentTool/runAgent.ts:103-225, 256-420`
- [ ] 产出项：画出父 agent、子 agent、上下文隔离、结果回收四个边界。
- [ ] 验证命令：`python agents/s04_subagent.py`
- [ ] 输出物链接：[]()

### Day 12 Task System
- [ ] 阅读项：`docs/en/s07-task-system.md`；`agents/s07_task_system.py:47-224, 173-200, 204-224`
- [ ] 产出项：整理任务状态机，至少列出创建、分派、执行、完成、失败五种状态。
- [ ] 验证命令：`python agents/s07_task_system.py`
- [ ] 输出物链接：[]()

### Day 13 Background Task
- [ ] 阅读项：`docs/en/s08-background-tasks.md`；`agents/s08_background_tasks.py:50-215, 163-185, 188-215`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/BashTool/BashTool.tsx:624-819, 826-940`
- [ ] 产出项：写一份“前台 turn 驱动 vs 后台持续执行”的差异表。
- [ ] 验证命令：`python agents/s08_background_tasks.py`
- [ ] 输出物链接：[]()

### Day 14 Coordinator 与多 Agent
- [ ] 阅读项：`docs/en/s09-agent-teams.md`；`docs/en/s10-team-protocols.md`；`/Users/hong.gao/python/src/claude-code-codex/docs/04-multi-agent-coordinator.md`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/shared/spawnMultiAgent.ts:72-100, 208-294, 305-420`
- [ ] 产出项：做一张“subagent / teammate / coordinator”对比表，维度至少包含上下文、权限、生命周期、通信方式。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 15 自治协作与 Worktree 隔离
- [ ] 阅读项：`docs/en/s11-autonomous-agents.md`；`docs/en/s12-worktree-task-isolation.md`；`agents/s11_autonomous_agents.py:126-377, 438-553`；`agents/s12_worktree_task_isolation.py:121-474, 536-553, 729-759`
- [ ] 产出项：写一份“自治协作为什么必须结合隔离执行环境”的说明，至少覆盖任务认领、事件总线、worktree 生命周期。
- [ ] 验证命令：`python agents/s12_worktree_task_isolation.py`
- [ ] 输出物链接：[]()

## 第 4 周：Permission、Sandbox、MCP、总复刻

### Day 16 权限决策
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/docs/05-tool-skill-system.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/permissions/permissions.ts:1071-1265`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/interactiveHandler.ts:57-260`
- [ ] 产出项：画出权限决策树，区分 rule-based、interactive、sandbox 自动放行三条路径。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 17 Sandbox 核心
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/docs/06-sandbox-system.md`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sandbox/sandbox-adapter.ts:83-340`；`/Users/hong.gao/python/src/claude-code-codex/src/tools/BashTool/shouldUseSandbox.ts:18-153`
- [ ] 产出项：做一张“命令类型 -> 是否进 sandbox -> 原因”的映射表。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 18 Coordinator 权限分层
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/coordinatorHandler.ts:16-64`；`/Users/hong.gao/python/src/claude-code-codex/src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:26-159`
- [ ] 产出项：写一页多 agent 权限分层说明，明确 coordinator 与 worker 的授权差异。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 19 MCP 与入口层
- [ ] 阅读项：`/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/mcp.ts:36-197`；`/Users/hong.gao/python/src/claude-code-codex/src/main.tsx:586-760`；`/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/cli.tsx:34-260`
- [ ] 产出项：画出 CLI 入口、MCP 入口、主循环初始化三者关系图。
- [ ] 验证命令：`cd /Users/hong.gao/python/src/claude-code-codex && bun run typecheck`
- [ ] 输出物链接：[]()

### Day 20 Mini Claude Code 复刻
- [ ] 阅读项：`agents/s01_agent_loop.py:81-101`；`agents/s04_subagent.py:118-136`；`agents/s06_context_compact.py:68-127`；`/Users/hong.gao/python/src/claude-code-codex/src/query.ts:247-605`；`/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-365`；`/Users/hong.gao/python/src/claude-code-codex/src/utils/sessionStorage.ts:128-303`
- [ ] 产出项：实现一个 500-1000 行的 mini 原型，或写一份等价设计文档，最少包含 loop、tool pool、compact、subagent、transcript 五块。
- [ ] 验证命令：`python -m pytest tests/test_agents_smoke.py -q`
- [ ] 输出物链接：[]()

## 每周固定复盘模板

- [ ] 本周我真正跑过的脚本：
- [ ] 本周我能完整复述的调用链：
- [ ] 本周最模糊的 3 个概念：
- [ ] 下周开始前需要补的源码段：
- [ ] 周总结链接：[]()
