# Day 7 输出物：Hook 事件矩阵与侧车扩展导图

Day 7 最重要的不是“系统又多了一层扩展”，而是把主状态推进与旁路扩展彻底拆开：

- `query / tool loop` 负责推进主流程
- `hooks / post-sampling / async sidecar` 负责在固定时机观察、拦截、补充、异步回流

所以主线不是“在主循环里继续堆 if/else”，而是：

```text
tool_use / lifecycle event
  ->
hook matcher + registry
  ->
execute hooks in parallel
  ->
aggregate result
  ->
permission / tool result / continuation control / async attachment 回流
```

## 1. Hook 事件矩阵

| 事件/通道 | 触发时机 | 输入载荷 | 返回语义 | 是否影响主循环推进 |
|---|---|---|---|---|
| `SessionStart` | 会话启动、恢复、clear、compact | `session_id / transcript_path / cwd / source / model / agent_type` | `additionalContext`、`initialUserMessage`、`watchPaths` | 影响初始化上下文，但不替代主循环 |
| `PreToolUse` | 工具执行前 | `tool_name / tool_input / tool_use_id` | `allow / ask / deny / passthrough`、`updatedInput`、`additionalContext` | 会影响，甚至阻止工具执行 |
| `PostToolUse` | 工具成功后 | `tool_name / tool_input / tool_response / tool_use_id` | `additionalContext`、`updatedMCPToolOutput`、`preventContinuation` | 不回滚当前工具，但可阻止继续续行 |
| `PostToolUseFailure` | 工具失败后 | `tool_name / tool_input / error / is_interrupt` | 附加错误反馈、阻塞/非阻塞 Hook 信息 | 不改变失败事实，但影响失败后的反馈 |
| `hookEvents` 执行事件 | Hook 执行期间 | `hookId / hookName / hookEvent / stdout / stderr / output` | `started / progress / response` | 不影响主循环，只提供可观测性 |
| `postSamplingHooks` | 模型采样完成后 | 完整 `messages / systemPrompt / userContext / systemContext / toolUseContext` | 无结构化返回，错误仅记录 | 默认不影响，是纯 sidecar |

## 2. 调用链主图

```text
assistant emits tool_use
  ->
toolExecution.ts
  -> zod parse
  -> tool.validateInput
  -> preprocess observable input
  ->
runPreToolUseHooks()
  -> executePreToolHooks()
  -> utils/hooks.ts executeHooks()
  -> session/settings/skill hooks merged
  -> hooks run in parallel
  -> aggregated result
  ->
resolveHookPermissionDecision()
  -> hook allow/ask/deny 与规则系统收口
  ->
tool.call()
  ->
runPostToolUseHooks()
  -> additional context / stop continuation / MCP output rewrite
  ->
append tool_result and continue
```

失败分支则是：

```text
tool.call() throws / aborts
  ->
runPostToolUseFailureHooks()
  ->
hook attachments / additional context
  ->
tool_result error 回流
```

## 3. 最关键的源码片段

### 3.1 `hookEvents.ts`：不是执行业务 Hook，而是执行事件总线

```ts
const ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup'] as const

function shouldEmit(hookEvent: string): boolean {
  if ((ALWAYS_EMITTED_HOOK_EVENTS as readonly string[]).includes(hookEvent)) {
    return true
  }
  return (
    allHookEventsEnabled &&
    (HOOK_EVENTS as readonly string[]).includes(hookEvent)
  )
}
```

这段的意义是：`SessionStart` 和 `Setup` 默认可观测，其它高频事件要显式开启。

### 3.2 `AsyncHookRegistry.ts`：异步 Hook 先登记，再二次回收

```ts
pendingHooks.set(processId, {
  processId,
  hookId,
  hookName,
  hookEvent,
  timeout,
  responseAttachmentSent: false,
  shellCommand,
  stopProgressInterval,
})
```

```ts
if (hook.shellCommand.status !== 'completed') {
  return { type: 'skip' as const }
}

hook.responseAttachmentSent = true
await finalizeHook(hook, exitCode, exitCode === 0 ? 'success' : 'error')
```

这说明异步 Hook 的关键不是后台跑，而是后台跑完以后怎么只回送一次结果。

### 3.3 `sessionHooks.ts`：session 内动态 Hook 不和全局配置混在一起

```ts
export type SessionHooksState = Map<string, SessionStore>

export function addFunctionHook(...) {
  const hook: FunctionHook = {
    type: 'function',
    id,
    timeout: options?.timeout || 5000,
    callback,
    errorMessage,
  }
  addHookToSession(setAppState, sessionId, event, matcher, hook)
}
```

这说明运行时 callback hook 是 session-scoped、runtime-only 的，不是 settings 持久化项。

### 3.4 `registerSkillHooks.ts`：skill frontmatter 通过 session hook 注入

```ts
const onHookSuccess = hook.once
  ? () => removeSessionHook(setAppState, sessionId, eventName, hook)
  : undefined

addSessionHook(
  setAppState,
  sessionId,
  eventName,
  matcher.matcher || '',
  hook,
  onHookSuccess,
  skillRoot,
)
```

`once: true` 的真实语义是“成功执行一次后移除”，不是“尝试一次就移除”。

### 3.5 `toolExecution.ts`：Hook 看到的是整形后的观察输入

```ts
let callInput = processedInput
const backfilledClone =
  tool.backfillObservableInput &&
  typeof processedInput === 'object' &&
  processedInput !== null
    ? ({ ...processedInput } as typeof processedInput)
    : null
if (backfilledClone) {
  tool.backfillObservableInput!(backfilledClone as Record<string, unknown>)
  processedInput = backfilledClone
}
```

这里区分了：

- `callInput`：真正传给工具的调用输入
- `processedInput`：外围 Hook / permission / telemetry 看到的可观察输入

## 4. 三条必须记住的结论

1. Hook 不是第二套主循环，而是主循环暴露给 sidecar 的固定扩展点。
2. 产品版 Hook 不是串行 `if/else`，而是“多来源匹配 + 并行执行 + 结构化聚合 + 按需异步回流”。
3. `PreToolUse hook` 可以参与权限决策，但不能绕过规则系统硬边界；Hook 的 `allow` 不是绝对最高优先级。

## 5. 参考位置

- 教学主线：`docs/zh/s08-hook-system.md`，`docs/zh/day7-hook-events-and-sidecar-guide.md`
- 教学实现：`agents/s08_hook_system.py`
- 真实源码：
  - `src/utils/hooks/hookEvents.ts`
  - `src/utils/hooks/AsyncHookRegistry.ts`
  - `src/utils/hooks/sessionHooks.ts`
  - `src/utils/hooks/postSamplingHooks.ts`
  - `src/utils/hooks/registerSkillHooks.ts`
  - `src/services/tools/toolExecution.ts`
  - `src/services/tools/toolHooks.ts`
