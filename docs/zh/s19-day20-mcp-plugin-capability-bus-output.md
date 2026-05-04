# Day 20 输出物：mini Claude Code 八块总蓝图

Day 20 的产出不是再背一份 MCP 名词表，而是把前 20 天收束成一个可以继续实现的系统蓝图。

这份蓝图的硬边界是：

> **外部能力和本地能力必须进入同一条 tool pool、permission gate、tool_result 回路。**

## 1. 总体链路

```text
user request
  ->
agent loop
  ->
build runtime context
  - system prompt
  - memory
  - compact summary
  - task state
  - team mailbox state
  ->
assemble tool pool
  - native tools
  - MCP tools from connected servers
  ->
model turn
  ->
tool_use or final text
  ->
permission gate
  ->
tool router
  - native route
  - MCP route
  ->
normalized tool_result
  ->
event / transcript / task side effects
  ->
next loop turn or stop
```

最短系统判断法：

```text
loop 决定下一步
tools 执行动作
permissions 控制风险
compact 管上下文寿命
task 管长期工作
teams 管多人协作
worktree 管隔离执行目录
MCP 管外部能力接入
```

## 2. 八块模块表

| 模块 | 主要职责 | 当前教学仓库来源 | 参考源码映射 |
| --- | --- | --- | --- |
| `loop` | 接收请求、调用模型、处理 tool_use / tool_result、决定继续或停止 | `agents/s01_agent_loop.py`；`agents/s_full.py` | `src/query.ts`；`src/main.tsx` |
| `tools` | 声明工具、暴露 schema、执行 handler、返回结构化结果 | `agents/s02_tool_use.py`；`agents/s19_mcp_plugin.py` | `src/tools.ts`；`src/services/tools/*` |
| `permissions` | 对读写、高风险、外部能力做 allow / ask / deny | `agents/s07_permission_system.py`；`CapabilityPermissionGate` | `src/permissions/*`；`toolPermissionContext`；MCP tool check |
| `compact` | 当上下文过长时压缩历史，并保留继续工作所需状态 | `agents/s06_context_compact.py` | `src/query.ts` compact 段；`src/services/toolResultStorage/*` |
| `task` | 管 durable task、runtime slot、状态迁移、恢复续行 | `agents/s12_task_system.py`；`agents/s13_background_tasks.py`；`agents/s14_cron_scheduler.py` | `src/tasks/types.ts`；`LocalRuntimeTask`；`TaskStopTool` |
| `teams` | 管 team、member、mailbox、协议消息、request_id | `agents/s15_agent_teams.py`；`agents/s16_team_protocols.py`；`agents/s17_autonomous_agents.py` | `TeamCreateTool`；`SendMessageTool`；`coordinatorMode.ts`；`swarm/*` |
| `worktree` | 为并行任务绑定隔离目录，处理 create / enter / keep / remove / restore | `agents/s18_worktree_task_isolation.py` | `EnterWorktreeTool`；`ExitWorktreeTool`；`utils/worktree.ts`；`sessionStorage.ts` |
| `MCP` | 从 plugin manifest 发现外部 server，连接后把外部 tools 合并进工具池 | `agents/s19_mcp_plugin.py`；`docs/zh/s19a-mcp-capability-layers.md` | `entrypoints/mcp.ts`；`services/mcp/config.ts`；`services/mcp/client.ts`；`pluginLoader.ts` |

这张表里最重要的是来源分工：

```text
教学仓库 = 最小可运行形状
参考源码 = 真实系统边界和复杂度校准
```

## 3. Loop 蓝图

```text
start request
  ->
load conversation state
  ->
compose system prompt
  ->
attach available tools
  ->
call model
  ->
if text only:
    emit assistant message
    stop
if tool_use:
    execute tool path
    append tool_result
    continue loop
if context pressure:
    compact
    continue with summary
```

Loop 不直接知道每个工具的实现细节。

它只关心：

- 当前有哪些工具可用
- 模型选择了哪个工具
- 工具结果怎样回到下一轮消息
- 是否需要 compact、停止或恢复

## 4. Tools 蓝图

工具需要统一成同一种形状：

```text
ToolDefinition
  name
  description
  input_schema
  risk / permission metadata
  handler or router target
```

工具池由两部分合成：

```text
native tool definitions
  +
MCP tool definitions after prefixing
  ->
assembled tool pool
```

MCP 工具命名规则：

```text
mcp__{server_name}__{tool_name}
```

这个前缀解决三个问题：

- 避免和内置工具重名
- 保留工具来源
- 路由时能还原 server 和原始 tool name

## 5. Permissions 蓝图

权限层先把工具调用归一成 capability intent：

```text
tool_use
  ->
intent {
    source: native | mcp
    server: optional
    tool: actual_tool_name
    risk: read | write | high
  }
```

然后做裁决：

```text
read  -> allow by default
write -> ask unless trusted mode allows
high  -> ask or deny by policy
deny  -> do not route
```

硬边界：

> **MCP 工具虽然来自外部，但权限裁决必须和本地工具在同一层发生。**

## 6. Compact 蓝图

Compact 管的是上下文寿命，不是任务完成度。

最小状态应该保留：

- 用户目标
- 已完成动作
- 当前 task / subtask 状态
- 最近重要 tool_result
- open questions
- worktree lane / cwd 信息
- team mailbox 中未关闭的 request

压缩后的 loop 应该像这样继续：

```text
old transcript
  ->
summary + retained state
  ->
new model turn
  ->
same tool pool / same permission path
```

## 7. Task 蓝图

任务层至少分两类：

```text
Durable Task
  长期目标、状态、owner、产出、依赖

Runtime Task
  当前执行槽位、进程状态、取消控制、最后事件
```

典型状态：

```text
task: pending -> in_progress -> blocked | completed | failed
runtime: queued -> running -> done | killed | crashed
```

不要把 task completed 和 runtime done 混成同一个字段。

一个任务可以 completed，但相关 worktree 仍然 kept。

## 8. Teams 蓝图

Team 协作最小对象：

```text
team
  members[]
  leader
  mailbox
  tasks

member
  id
  role
  status
  runtime_task_id

message
  from
  to
  kind
  request_id
  payload
```

`request_id` 是硬边界。

它决定：

- 哪个请求正在等待回复
- 哪个结果能关闭等待状态
- 哪个 shutdown / plan approval / result 属于同一次协作

自由文本可以聊天，但不能可靠驱动调度。

协议消息必须能被机器判定状态。

## 9. Worktree 蓝图

Worktree 不是 task 本身，而是 task 的隔离执行车道。

```text
task_create
  ->
worktree_create(task_id)
  ->
bind task_id <-> worktree
  ->
enter lane
  ->
run command with cwd=worktree_path
  ->
closeout: keep or remove
  ->
persist / restore lane state
```

三层边界：

```text
task         = 做什么
runtime task = 当前谁在跑
worktree lane = 在哪做
```

删除前必须检查：

- 是否有未提交文件
- 是否有额外 commit
- 用户是否明确允许 discard

## 10. MCP / Plugin 蓝图

MCP 接入分三层：

```text
Plugin manifest
  发现 server 配置

MCP server
  运行中的外部能力提供者

MCP tool
  server 暴露的具体 callable capability
```

启动阶段：

```text
load plugin sources
  ->
validate manifest
  ->
collect mcpServers config
  ->
apply scope / policy / enable state
  ->
connect clients
  ->
list tools
  ->
prefix names
  ->
merge into tool pool
```

运行阶段：

```text
model emits mcp__server__tool
  ->
permission gate
  ->
router parses server/tool
  ->
MCP client calls actual tool
  ->
process MCP result
  ->
normalized tool_result
```

MCP 还可能有更完整的 capability layers：

- tools
- resources
- prompts
- elicitation
- auth
- connection state

但总蓝图的主线仍然是 tools-first。

## 11. 关键不变量

```text
1. 模型只看到 assembled tool pool，不直接看到 plugin manifest。
2. 本地工具和 MCP 工具都必须经过同一层 permission。
3. tool_result 要归一化，不能让外部 server 的原始响应污染主循环。
4. task、runtime task、worktree lane 是三层状态，不要合并。
5. team protocol 必须有 request_id，不能只靠自由文本。
6. compact 后必须能恢复继续执行所需状态。
7. plugin loader 负责发现和校验，不负责执行工具。
8. MCP client 负责连接、列工具、调用工具和处理外部结果。
```

## 12. 最小实现顺序

如果从零继续写 mini Claude Code，建议顺序是：

```text
1. loop + transcript
2. native tool definition + execution
3. permission gate
4. compact summary
5. durable task + runtime task
6. team mailbox + protocol message
7. worktree lane binding
8. plugin manifest loader
9. MCP client list_tools / call_tool
10. merge native + MCP tool pool
```

这样做的原因是：

- loop 是所有能力的承载面
- tools 和 permissions 是动作控制面
- compact / task / team / worktree 是长程执行面
- MCP 是能力扩展面，必须最后接回工具池

## 13. 一页版结论

```text
Claude Code-like system
  =
agent loop
  + tool control plane
  + permission gate
  + context lifecycle
  + durable task runtime
  + multi-agent mailbox protocol
  + worktree execution lanes
  + MCP/plugin capability bus
```

其中 Day 20 的关键补全是：

> **外部工具不是单独世界；它们经过 plugin discovery 和 MCP connection 后，必须变成普通 tool definition，进入同一条权限、路由、结果回路。**
