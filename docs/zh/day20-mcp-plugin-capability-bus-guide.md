# Day 20 学习指南：MCP、Plugin 与外部能力总线

> Day 19 学的是“并行任务在哪条隔离目录车道里执行”。
> Day 20 学的是“外部工具怎样接进同一条 agent 控制面，并把前 19 天复刻成一张系统蓝图”。
>
> 这一天最重要的不是 MCP 协议细节，而是先守住下面这句话：
>
> **外部能力进入系统的方式可以不同，但进入后必须回到同一条 tool / permission / result 总线。**

## 0. 这一天到底在学什么

前 19 天已经把一个类 Claude Code 系统拆成了很多块：

- agent loop
- tool use
- todo / plan
- subagent
- skill loading
- context compact
- permission
- hook
- memory
- system prompt
- error recovery
- task runtime
- scheduler
- team
- coordinator protocol
- autonomous worker
- worktree lane

Day 20 要做两件事：

1. 学会 MCP / plugin 怎样把外部能力接进 agent。
2. 把前面所有模块压成一份“mini Claude Code 总蓝图”。

如果只记一句话，先记这个：

> **MCP 不是另一套 agent loop，而是外部 capability 进入现有工具池的接入层。**

## 1. 这次重点看哪些代码

### 必读

- `docs/zh/s19-mcp-plugin.md`
- `docs/zh/s19a-mcp-capability-layers.md`
- `agents/s19_mcp_plugin.py`
- `/Users/hong.gao/python/src/claude-code-codex/src/entrypoints/mcp.ts:36-197`
- `/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/config.ts:62-223,625-888,1071-1297,1470-1553`
- `/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/client.ts:595-744,1743-2033,2226-2813,3029-3262`
- `/Users/hong.gao/python/src/claude-code-codex/src/services/mcp/MCPConnectionManager.tsx:17-72`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools.ts:191-257,343-387`
- `/Users/hong.gao/python/src/claude-code-codex/src/main.tsx:518-760,858-930`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/plugins/pluginLoader.ts:266-365,492-718,1147-1360,1888-2237,3009-3232`

在教学版 `agents/s19_mcp_plugin.py` 里，优先盯这几个入口：

- `CapabilityPermissionGate`
- `MCPClient.connect(...)`
- `MCPClient.list_tools(...)`
- `MCPClient.get_agent_tools(...)`
- `PluginLoader.load(...)`
- `MCPToolRouter.route(...)`
- `build_tool_pool(...)`

原因很直接：

- `CapabilityPermissionGate` 说明 MCP 不能绕过权限面
- `MCPClient` 说明外部 server 怎样暴露工具
- `PluginLoader` 说明 plugin manifest 只是发现 server 配置
- `MCPToolRouter` 说明本地工具和外部工具最终要走同一个调用结果格式

## 2. 先建立最小心智模型

Day 20 可以先画成下面这条线：

```text
Plugin manifest
  ->
MCP server config
  ->
MCP client connection
  ->
list tools
  ->
normalize tool names
  ->
merge into tool pool
  ->
permission check
  ->
native route or MCP route
  ->
normalized tool_result
  ->
agent loop continues
```

这里有一个关键边界：

> **Plugin 负责发现配置，MCP server 负责暴露能力，MCP tool 才是真正被模型调用的能力。**

不要把这三层混成一层。

## 3. Plugin、MCP Server、MCP Tool 的边界

| 层 | 它是什么 | 它不是什么 |
| --- | --- | --- |
| `plugin manifest` | 一份声明文件，告诉系统有哪些 server 配置 | 不是运行中的 server |
| `MCP server` | 一个外部进程或远端连接，暴露一组 capability | 不是单个工具 |
| `MCP tool` | server 暴露的具体 callable capability | 不是独立插件 |

教学版可以用一个最小 manifest 表达：

```json
{
  "name": "my-db-tools",
  "version": "1.0.0",
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"]
    }
  }
}
```

这份配置只回答一个问题：

> 如果要接入 `postgres` 这个外部 server，应该怎样启动它？

真正的工具列表要等 server 连接后通过 `tools/list` 取回来。

## 4. MCP 接入主循环的正确位置

MCP 不应该直接出现在模型旁边。

它应该先进入工具池：

```text
native tools
  + MCP tools after normalization
  ->
assembled tool pool
  ->
model sees available tools
```

运行时则是：

```text
model emits tool_use
  ->
tool permission context
  ->
tool executor / router
  ->
if native: call local implementation
if mcp: call MCP client
  ->
process result
  ->
tool_result message
```

参考源码里的 `assembleToolPool(...)` 很值得看：

- 内置工具先进入基础工具池
- MCP 工具再按权限上下文过滤
- 名称冲突时内置工具优先
- deny 规则可以让 MCP 工具不进入模型可见工具池

这说明 MCP 不是“模型想调什么就调什么”。

它先要变成系统认可的 tool definition，才能被模型看到。

## 5. 真实系统比教学版多出来的能力层

教学版先坚持 tools-first，这是对的。

但参考源码会多出这些层：

| 能力层 | 负责什么 | 教学版是否必须实现 |
| --- | --- | --- |
| Config Layer | server 配置、scope、policy、enable/disable | 必须理解，最小实现即可 |
| Transport Layer | stdio / http / sse / websocket | 先实现 stdio 即可 |
| Connection State Layer | connected / pending / failed / needs-auth / disabled | 必须理解，可简化 |
| Capability Layer | tools / resources / prompts / elicitation | 主线先做 tools |
| Auth Layer | OAuth / XAA / session expiry | 知道位置即可 |
| Router Integration Layer | 权限、路由、通知、结果归一化 | 必须实现或清楚设计 |

Day 20 不要求把 auth 和 elicitation 都手搓出来。

但必须知道它们不属于 agent loop 本身，而是外部 capability 平台的高级接入层。

## 6. 参考源码里的几条关键线

### 6.1 `entrypoints/mcp.ts`

这条线说明 Claude Code 本身也可以作为 MCP server 暴露能力。

最重要的是：

- `ListToolsRequestSchema` 把内部工具暴露给 MCP client
- `CallToolRequestSchema` 把外部请求转回内部工具执行
- 调用仍然带着 `toolPermissionContext`
- 输入校验失败、权限拒绝、工具执行异常都会转成 MCP response

这说明“服务别人调用”时也不能绕开工具权限。

### 6.2 `services/mcp/config.ts`

这条线说明 MCP server 配置不是一个平铺 JSON。

它至少涉及：

- enterprise / user / project / local / dynamic / plugin / claudeai 等来源
- scope 优先级和覆盖规则
- policy allow / deny
- enable / disable
- plugin server 去重

所以最小教学版可以简化配置，但设计文档必须写清楚：

> server config 有来源和优先级，不是随便合并。

### 6.3 `services/mcp/client.ts`

这条线说明 MCP client 不只是 `call_tool`。

参考实现还处理：

- 多 transport 连接
- tool schema 转换
- tool name 前缀
- tool call timeout
- progress notification
- structured output
- large / blob result
- auth failure / session expired
- server 状态分组
- resources / prompts / commands

教学版可以先做最小路径，但最终蓝图要知道这些能力应该挂在 client 层和 result processing 层。

### 6.4 `utils/plugins/pluginLoader.ts`

这条线说明 plugin loader 的职责不是执行工具。

它负责：

- 从 session / marketplace / builtin 多来源加载 plugin
- 校验 manifest
- 校验 commands / agents / hooks 等组件路径
- 处理 cache-only 和 full-load
- 处理 marketplace policy
- 合并 plugin 来源并解决优先级
- 缓存 plugin settings 供启动阶段读取

所以 plugin loader 的边界是：

> 发现、校验、装配插件元数据；不直接跑 agent loop。

## 7. Day 20 的总复刻蓝图

产出物不一定要写 500-1000 行代码。

如果写设计蓝图，至少要覆盖下面八块：

| 块 | 必须回答的问题 |
| --- | --- |
| `loop` | 一轮用户请求怎样进入模型、工具、结果、下一轮 |
| `tools` | 工具怎样声明、选择、执行、返回 |
| `permissions` | 哪些行为要 allow / ask / deny，谁做最终裁决 |
| `compact` | 长上下文怎样压缩，压缩后怎样继续 |
| `task` | durable task 和 runtime slot 怎样分层 |
| `teams` | teammate、mailbox、protocol、request_id 怎样协作 |
| `worktree` | 并行任务怎样绑定隔离执行目录 |
| `MCP` | 外部能力怎样通过 plugin / server / tool 接回工具池 |

这八块不是八个孤立模块。

它们应该连成一条执行链：

```text
request
  ->
loop
  ->
system prompt / memory / compact state
  ->
tool pool = native tools + MCP tools
  ->
permission gate
  ->
tool execution
  ->
task / team / worktree side effects
  ->
tool_result
  ->
loop continues or compact / stop / resume
```

## 8. 源码归属怎么标

输出物里必须标出两类来源：

### 当前教学仓库

主要用于“最小可运行心智模型”：

- `agents/s01_agent_loop.py`
- `agents/s02_tool_use.py`
- `agents/s06_context_compact.py`
- `agents/s07_permission_system.py`
- `agents/s12_task_system.py`
- `agents/s15_agent_teams.py`
- `agents/s16_team_protocols.py`
- `agents/s17_autonomous_agents.py`
- `agents/s18_worktree_task_isolation.py`
- `agents/s19_mcp_plugin.py`

### 参考源码映射

主要用于“接近真实系统的模块边界”：

- `src/query.ts`
- `src/tools.ts`
- `src/services/mcp/config.ts`
- `src/services/mcp/client.ts`
- `src/utils/plugins/pluginLoader.ts`
- `src/tools/SendMessageTool/SendMessageTool.ts`
- `src/utils/swarm/inProcessRunner.ts`
- `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
- `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`
- `src/utils/worktree.ts`
- `src/utils/sessionStorage.ts`

判断标准是：

> 教学仓库负责复刻核心形状；参考源码负责校准真实边界。

## 9. 常见错误

### 9.1 让 MCP 工具绕过 permission

这是最严重的错误。

外部能力一旦绕过权限面，就等于在系统侧面开了一个不受控执行入口。

### 9.2 把 plugin manifest 当成工具

manifest 只是配置声明。

真正的工具来自 MCP server 连接后的 `tools/list`。

### 9.3 把 transport 细节放到主线最前面

初学阶段先做 stdio。

transport、auth、elicitation 可以在平台层补充，但不要打断 tool-first 主线。

### 9.4 总复刻只列模块，不画调用链

Day 20 的蓝图不能只是目录表。

它必须能讲清：

```text
用户请求怎样进来
模型怎样看到工具
工具怎样被权限裁决
本地和 MCP 工具怎样统一路由
任务、团队、worktree 怎样成为 side effect
结果怎样回到下一轮 loop
```

## 10. 验证方式

最低验证：

```bash
python -m py_compile agents/s19_mcp_plugin.py
```

如果要验证参考仓库：

```bash
cd /Users/hong.gao/python/src/claude-code-codex
bun run typecheck
```

注意：

- `agents/s19_mcp_plugin.py` 的完整运行依赖 `MODEL_ID` 和实际 MCP server 配置。
- `bun run typecheck` 验证的是参考源码，不是当前教学仓库生成的 markdown。

## 11. 一句记忆钩子

> **Day 20 的 MCP/plugin 不是外挂章节，而是把“外部能力”接回同一条工具控制面；总复刻则要证明你已经能把 20 天的模块连成一条系统链。**
