# Day 2 输出物：`tool pool -> permission gate -> execution runtime -> tool_result` 回流

这条链路的关键不是“按名字找到 handler”这么简单，而是先把所有可见能力组装成同一个 `tool pool`，再让每一次 `tool_use` 经过统一的 `permission gate`，只有被放行的调用才进入 `execution runtime`，最后再统一包装成 `tool_result` 回流到主循环。

## 0.一些理解
- LLM API一次调用可能会触发返回多个工具，这些工具的执行顺序是要根据工具的一些属性（是否可以并发）做编排的，按编排顺序执行，同时上下文修改也是有一定规则的，保证上下文结果的稳定性。
- batch 内的 contextModifier 不会立刻乱改共享上下文，而是等整批结束后，再按原始工具顺序应用，这块可以保证上下文稳定。 实现方式：并发 batch 里，先把各工具的 contextModifier 暂存到 queuedContextModifiers，整批跑完后，再按原始 blocks 顺序应用。
- 工具进度message是不会进入下一次LLM的上下文的，工具负责产出“进度数据”，框架把这些数据包装成progress消息。

## 1. 一张总图

```text
getAllBaseTools()
  -> getTools(permissionContext)
  -> assembleToolPool(permissionContext, mcpTools)
  -> model sees one tool pool

assistant emits tool_use blocks
  -> runToolUse()
  -> input parse / validate
  -> PreToolUse hooks
  -> resolveHookPermissionDecision()
  -> allow | ask | deny

allow
  -> toolOrchestration / StreamingToolExecutor
  -> serial batch or concurrent batch
  -> tool.call(...)
  -> progress / context modifier / result shaping

deny or ask-reject
  -> synthesize error tool_result

all paths
  -> create user message with tool_result
  -> append back to messages
  -> next model turn
```

## 2. `tool pool` 怎么形成

- 入口不是单个工具，而是 `getAllBaseTools() -> getTools() -> assembleToolPool()` 三级组装。
- `getTools()` 先处理 mode 过滤，再用 deny rules 先删掉不该暴露给模型的工具，然后处理 REPL mode 和 `isEnabled()`。
- `assembleToolPool()` 再把 built-in tools 和 MCP tools 合到一起，并再次按 deny rules 过滤 MCP 工具。
- 合并时会先分别排序，再按 `name` 去重，内置工具优先；所以“模型看见什么工具”本身就是控制面的一部分，不只是执行时再拦。

对应参考实现：`src/tools.ts:191-365`。

## 3. `permission gate` 插在 runtime 前面

- `runToolUse()` 先在当前可见工具池里 `findToolByName()`；找不到才回退到 base tools 的 alias。
- 找到工具后，不会立刻 `tool.call()`，而是进入 `checkPermissionsAndCallTool()`。
- 这里的顺序很关键：
  1. `inputSchema.safeParse / parse` 做输入类型校验。
  2. `validateInput` 做工具自定义值校验。
  3. `runPreToolUseHooks()` 先跑前置 hook，可改输入、给附加上下文、甚至直接 stop。
  4. `resolveHookPermissionDecision()` 再汇总 hook 决策和权限系统。
  5. 底层权限主线是 `hasPermissionsToUseToolInner()`：
     `deny rule -> ask rule -> tool.checkPermissions -> tool-specific deny/ask/safety check -> bypass mode allow -> always-allow rule -> passthrough 转 ask`
- 只要最终 `permissionDecision.behavior !== 'allow'`，就不会进入执行面，而是直接构造 `is_error: true` 的 `tool_result` 回写。
- 所以 `ask` 和 `deny` 也是结果，但不是执行；它们终止在 gate，不会流入 `tool.call()`。

对应参考实现：`src/services/tools/toolExecution.ts:337-490,599-1319`，`src/utils/permissions/permissions.ts:1071-1319`。

## 4. `execution runtime` 怎么决定串行还是并行

批量执行主线在 `runTools()`，先 `partitionToolCalls()` 再调度。

### 串行/并行分组条件

- 先判断单个工具是否 `concurrency safe`：
  必须同时满足
  1. 能在当前 `tool pool` 里找到工具。
  2. `tool.inputSchema.safeParse(toolUse.input)` 成功。
  3. `tool.isConcurrencySafe(parsedInput.data)` 返回 `true`。
- 只要 parse 失败，或 `isConcurrencySafe()` 抛异常，系统就保守地当成 `false`。
- 分批规则不是“所有 safe 工具合成一组”，而是“连续出现的 safe 工具合成一组”。
- 任何 non-safe 工具都会单独开一个 serial batch。
- 所以批处理的真实分组规则是：

```text
[safe, safe, unsafe, safe, unsafe, unsafe]
-> [safe+safe] [unsafe] [safe] [unsafe] [unsafe]
```

### 批量 runtime 的执行规则

- concurrent batch 走 `runToolsConcurrently()`，底层并发上限由 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 控制，默认 `10`。
- serial batch 走 `runToolsSerially()`，每个工具执行完后才进入下一个。
- serial batch 里 `contextModifier` 会立刻改 `currentContext`。
- concurrent batch 里不会谁先跑完谁先改共享上下文，而是把 `contextModifier` 先按 `toolUseID` 暂存，等整批结束后再按原始 block 顺序统一合并。


## 5. 流式 runtime 的额外约束

- `StreamingToolExecutor.addTool()` 在模型还在流式输出时就给每个 `tool_use` 打上 `isConcurrencySafe`。
- `canExecuteTool()` 的条件更像“当前队列能不能继续开工”：
  1. 当前没有 executing tools，可以启动任何 queued tool。
  2. 当前已经有 executing tools 时，只有“新工具 safe 且所有正在执行的工具也 safe”才能继续并发。
- `processQueue()` 一旦遇到一个“现在不能启动的 non-safe 工具”就会停住，后面的 queued 工具也先不越过它，以保序。
- `getCompletedResults()` 会立刻吐 progress，但正式结果仍按 `tools[]` 的顺序回流；如果前面有一个 non-safe 工具仍在执行，会在它那里停住，不让后面的结果抢跑到前面。
- 并发批次里如果某个 Bash tool 报错，会触发 sibling abort，取消同批其他工具；这是因为 Bash 常有隐式依赖链，读类工具则不会互相连坐。
- 代码还明确写了一个当前边界：streaming runtime 目前不支持 concurrent tools 的 `contextModifier` 落地，只有 non-safe 工具的 modifier 会直接更新上下文。

对应参考实现：`src/services/tools/StreamingToolExecutor.ts:76-145,265-518`。

## 6. `tool_result` 怎么回流

- 真正放行后，执行面才会调用 `tool.call(...)`。
- 工具返回的不是裸字符串，而是 `ToolResult`，里面可以有 `data / newMessages / contextModifier / mcpMeta`。
- 结果不会直接原样塞回消息流，而是先经过 `processToolResultBlock()`：
  1. 规范化成协议里的 `tool_result` block。
  2. 超大结果时做持久化和预览替换。
  3. 保留 `is_error / structuredContent / _meta` 这类语义。
- 最终统一包装成一条 `user` 消息回写到 `messages`，下一轮模型再把它当作上一次工具调用的结果继续推理。
- 这也是为什么教学版虽然只写了 `results.append({"type": "tool_result", ...})`，但真实系统必须把 result shaping 单独做成一层。

对应参考实现：`src/services/tools/toolExecution.ts:1206-1416`。

## 7. 一句话结论

真正的主链不是 `tool_name -> handler`，而是：

```text
tool pool 决定模型能看见什么
-> permission gate 决定这次调用能不能进执行面
-> execution runtime 决定这批调用怎么串行/并行与怎样合并上下文
-> tool_result 统一回流到下一轮 loop
```

其中最容易写错的点只有两个：

- 不要把“能并发执行”和“结果按什么顺序回流”混成一件事。
- 不要让并发工具谁先跑完谁先乱改共享上下文。
