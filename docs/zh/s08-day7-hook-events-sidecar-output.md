# Day 7 教学讲义：Hook 事件与侧车扩展

> 这份讲义面向第一次读真实 Hook 源码的人。  
> 目标不是一上来吃掉全部事件种类，而是先建立一条稳定主线：
>
> **主循环只暴露时机，Hook 在这些时机旁路执行，并用结构化结果回影响或补充主流程。**

## 0. 这一天到底在学什么

到了 Day 7，系统已经不只是“能调用工具”和“能判断权限”了。

真正的问题开始变成：

- 不改主循环代码，怎么增加审计、通知、额外检查、上下文补充？
- 这些扩展逻辑从哪里注册，生命周期归谁管？
- Hook 执行时的进度、异步回调、失败结果，怎么安全地回流到系统？
- Skill、session、内部 sidecar 功能，为什么都能长在 Hook 体系旁边？

所以，这一天不是在学“更多 if/else”，而是在学：

```text
主循环 / 工具执行 = 主状态推进
Hook / post-sampling / async sidecar = 周边扩展控制面
```

如果只记一句话，可以记成：

> Hook 不是第二套主循环，而是主循环在固定生命周期节点上对外发出的旁路调用。

## 1. 推荐阅读顺序

第一次读，不要直接扎进 `src/utils/hooks.ts` 这类大文件。

建议按下面顺序读：

1. `docs/zh/s08-hook-system.md`
   先建立教学版 3 事件心智：`SessionStart / PreToolUse / PostToolUse`
2. `agents/s08_hook_system.py`
   先理解最小产品原型里，Hook 是怎么挂到 loop 上的
3. `src/utils/hooks/hookEvents.ts`
   再区分“业务 Hook”与“Hook 执行观测事件”
4. `src/utils/hooks/AsyncHookRegistry.ts`
   理解异步 Hook 为什么需要 registry 和二次回收
5. `src/utils/hooks/sessionHooks.ts`
   理解 session 内动态注入 Hook 的存储结构
6. `src/utils/hooks/registerSkillHooks.ts`
   理解 skill frontmatter 如何变成 session-scoped hooks
7. `src/utils/hooks/postSamplingHooks.ts`
   理解另一条“模型采样后 sidecar”通道
8. `src/services/tools/toolExecution.ts:492-760`
   最后回来看工具执行前的输入整形、Hook 前置准备和接线点

真实产品里，Hook 真正执行总线主要在 `src/utils/hooks.ts` 和
`src/services/tools/toolHooks.ts`。这两份虽然不在 Day 7 指定阅读项里，
但为了把 `toolExecution.ts` 看懂，下面会在必要时补它们的关键桥接代码。

## 2. 先建立最小心智模型

教学版在 [s08 文档](./s08-hook-system.md) 和
`agents/s08_hook_system.py` 里，只保留 3 个最适合入门的事件：

- `SessionStart`
- `PreToolUse`
- `PostToolUse`

可以先记住这一条最小流程：

```text
model emits tool_use
  ->
run PreToolUse hooks
  ->
permission / execute tool
  ->
run PostToolUse hooks
  ->
append tool_result and continue
```

教学版代码里，这条主线非常直白：

```python
pre_result = hooks.run_hooks("PreToolUse", ctx)
if pre_result.get("blocked"):
    ...
output = handler(**tool_input)
post_result = hooks.run_hooks("PostToolUse", ctx)
```

`agents/s08_hook_system.py` 的重要价值，不是功能完整，而是把边界讲清：

- loop 负责推进主状态
- hook runner 负责在时机点调用扩展
- hook result 决定是继续、拦截、还是补充说明

到了真实产品版，这条主线没变，但有三个重要升级：

1. 事件面扩展成几十种，不再只有 3 个事件。
2. Hook 结果不再只靠退出码，而是靠结构化 JSON。
3. Hook 来源不再只有配置文件，还包括 session 注册、skill frontmatter、内部 sidecar、function callback。

## 3. 先看事件面：产品版到底有哪些 Hook 事件

完整事件清单在 `src/entrypoints/sdk/coreTypes.ts`：

```ts
export const HOOK_EVENTS = [
  'PreToolUse',
  'PostToolUse',
  'PostToolUseFailure',
  'Notification',
  'UserPromptSubmit',
  'SessionStart',
  'SessionEnd',
  'Stop',
  'StopFailure',
  'SubagentStart',
  'SubagentStop',
  'PreCompact',
  'PostCompact',
  'PermissionRequest',
  'PermissionDenied',
  'Setup',
  'TeammateIdle',
  'TaskCreated',
  'TaskCompleted',
  'Elicitation',
  'ElicitationResult',
  'ConfigChange',
  'WorktreeCreate',
  'WorktreeRemove',
  'InstructionsLoaded',
  'CwdChanged',
  'FileChanged',
] as const
```

这里先不要被事件数量吓到。第一遍只要抓住四类：

- 生命周期：`SessionStart / SessionEnd / Setup`
- 工具执行：`PreToolUse / PostToolUse / PostToolUseFailure`
- 会话控制：`Stop / Compact / PermissionRequest / PermissionDenied`
- 多 agent / 运行时：`SubagentStart / SubagentStop / TaskCreated / TaskCompleted / TeammateIdle`

Day 7 真正要学会的，不是背全事件名，而是理解：

> 事件名定义了“什么时候给旁路扩展一个机会”，而不是“主循环到底怎么推进”。

## 4. `hookEvents.ts`：Hook 执行事件总线，不是业务 Hook 本体

源码：`src/utils/hooks/hookEvents.ts:61-188`

这个文件非常容易被误会。它不是跑 Hook 的地方，而是广播
“某个 Hook 开始了、进行中、结束了”的执行事件总线。

先看核心类型：

```ts
export type HookStartedEvent = {
  type: 'started'
  hookId: string
  hookName: string
  hookEvent: string
}

export type HookProgressEvent = {
  type: 'progress'
  hookId: string
  hookName: string
  hookEvent: string
  stdout: string
  stderr: string
  output: string
}

export type HookResponseEvent = {
  type: 'response'
  hookId: string
  hookName: string
  hookEvent: string
  output: string
  stdout: string
  stderr: string
  exitCode?: number
  outcome: 'success' | 'error' | 'cancelled'
}
```

这层回答的不是：

- Hook 允许不允许工具继续执行？

而是：

- 这个 Hook 开始了吗？
- 中间输出了什么？
- 最后是成功、失败还是取消？

### 4.1 延迟注册与事件缓存

先看 `registerHookEventHandler()` 和 `pendingEvents`：

```ts
const pendingEvents: HookExecutionEvent[] = []
let eventHandler: HookEventHandler | null = null

export function registerHookEventHandler(handler: HookEventHandler | null): void {
  eventHandler = handler
  if (handler && pendingEvents.length > 0) {
    for (const event of pendingEvents.splice(0)) {
      handler(event)
    }
  }
}
```

这段说明：

- 即使 handler 还没装好，Hook 事件也不会直接丢失
- 事件会先进入 `pendingEvents`
- 等 handler 注册后再一次性回放

这就是一个典型 sidecar 设计：

> 业务执行先发生，观测消费方可以稍后接入。

### 4.2 为什么不是所有事件都默认对外发

再看 `shouldEmit()`：

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

这段的意义很大：

- `SessionStart` 和 `Setup` 属于低噪声生命周期事件，默认就发
- 高频 Hook 事件默认不对外广播，防止 SDK/CLI 侧噪声过大
- 当 `includeHookEvents` 或 remote mode 开启后，才会全量广播

也就是说，产品设计明确区分了两层：

1. Hook 是否执行
2. Hook 执行过程是否对外暴露

这两件事不是同一个开关。

### 4.3 progress 不是实时流，而是轮询快照

`startHookProgressInterval()` 很值得专门记住：

```ts
export function startHookProgressInterval(params: {
  hookId: string
  hookName: string
  hookEvent: string
  getOutput: () => Promise<{ stdout: string; stderr: string; output: string }>
  intervalMs?: number
}): () => void {
  let lastEmittedOutput = ''
  const interval = setInterval(() => {
    void params.getOutput().then(({ stdout, stderr, output }) => {
      if (output === lastEmittedOutput) return
      lastEmittedOutput = output
      emitHookProgress({ ... })
    })
  }, params.intervalMs ?? 1000)
  interval.unref()
  return () => clearInterval(interval)
}
```

这里不是在监听流式 stdout 事件，而是在定时读取当前任务输出快照。

这个设计的好处是：

- 对消费端来说，progress 事件频率稳定
- 只在输出变化时发送，避免刷屏
- 不要求所有 Hook 执行器都暴露统一的流式接口

## 5. `AsyncHookRegistry.ts`：异步 Hook 的后台登记册

源码：`src/utils/hooks/AsyncHookRegistry.ts:30-309`

这个文件不是“执行 Hook”，而是解决一个更具体的问题：

> 某个 Hook 已经后台化运行了，系统稍后怎么知道它完成了、结果是什么、要不要把结果重新附着回主会话？

### 5.1 注册异步 Hook

核心入口在 `registerPendingAsyncHook()`：

```ts
const pendingHooks = new Map<string, PendingAsyncHook>()

export function registerPendingAsyncHook({
  processId,
  hookId,
  asyncResponse,
  hookName,
  hookEvent,
  command,
  shellCommand,
  toolName,
  pluginId,
}: ...) {
  const timeout = asyncResponse.asyncTimeout || 15000
  const stopProgressInterval = startHookProgressInterval({
    hookId,
    hookName,
    hookEvent,
    getOutput: async () => {
      const taskOutput = pendingHooks.get(processId)?.shellCommand?.taskOutput
      ...
      return { stdout, stderr, output: stdout + stderr }
    },
  })
  pendingHooks.set(processId, {
    processId,
    hookId,
    hookName,
    hookEvent,
    toolName,
    pluginId,
    command,
    startTime: Date.now(),
    timeout,
    responseAttachmentSent: false,
    shellCommand,
    stopProgressInterval,
  })
}
```

这段做了 4 件事：

1. 保存后台 Hook 的进程与元信息
2. 记录超时时间
3. 启动 progress 轮询
4. 把“是否已经把结果回送给系统”记在 `responseAttachmentSent`

`responseAttachmentSent` 这个字段非常关键，因为它决定：

- 一个异步 Hook 的结果是否只会被投递一次
- attachment 生成后能否安全清理 registry

### 5.2 回收异步 Hook 响应

真正重要的是 `checkForAsyncHookResponses()`：

```ts
if (hook.shellCommand.status !== 'completed') {
  return { type: 'skip' as const }
}

if (hook.responseAttachmentSent || !stdout.trim()) {
  hook.stopProgressInterval()
  return { type: 'remove' as const, processId: hook.processId }
}

const lines = stdout.split('\n')
let response: SyncHookJSONOutput = {}
for (const line of lines) {
  if (line.trim().startsWith('{')) {
    try {
      const parsed = jsonParse(line.trim())
      if (!('async' in parsed)) {
        response = parsed
        break
      }
    } catch {
      ...
    }
  }
}

hook.responseAttachmentSent = true
await finalizeHook(hook, exitCode, exitCode === 0 ? 'success' : 'error')
```

这里的心智很重要：

- Hook 可以先在 stdout 打很多日志
- 只要最后某一行能解析成同步 JSON 响应，就能当作真正的结构化结果
- 一旦标记 `responseAttachmentSent = true`，后续轮询就不会重复投递

所以产品版异步 Hook 的协议不是“stdout 全部必须是 JSON”，而是：

> stdout 里可以有普通日志，但要有一行结构化 JSON 作为最终响应。

### 5.3 为什么 `SessionStart` 完成后要失效环境缓存

别漏掉这段：

```ts
if (sessionStartCompleted) {
  invalidateSessionEnvCache()
}
```

这说明 `SessionStart` Hook 在产品里不只是“打个欢迎消息”。

它还可能：

- 写入 hook env 文件
- 改变当前 session 的环境脚本
- 影响后续 shell / tool 执行环境

这就是 sidecar 真正变“产品化”的地方：它不仅能观察，还能间接影响环境层。

## 6. `sessionHooks.ts`：会话内临时 Hook 的存储模型

源码：`src/utils/hooks/sessionHooks.ts:68-437`

这个文件解决的是另一个关键问题：

> 当前 session 临时装进去的 Hook 存哪里，怎么按 session 隔离，怎么支持运行时 callback？

### 6.1 为什么这里用 `Map` 而不是普通对象

作者在注释里把原因写得非常明白：

```ts
export type SessionHooksState = Map<string, SessionStore>
```

上面的长注释说明了两件事：

1. `.set/.delete` 不会改变容器身份，可以避免 store listener 大量触发
2. 在高并发 parallel workflow 下，`Map` 能避免不断 spread copy 带来的 O(N²) 成本

这不是风格问题，而是运行时成本问题。

### 6.2 session hook 有两类

这个文件里有两个核心入口：

```ts
export function addSessionHook(...)
export function addFunctionHook(...)
```

它们分别对应：

- `HookCommand`
  来自配置 / skill frontmatter / 命令式注册的常规 hook
- `FunctionHook`
  直接在内存里挂一个 TypeScript callback 的 runtime-only hook

`FunctionHook` 的定义也很直白：

```ts
export type FunctionHook = {
  type: 'function'
  id?: string
  timeout?: number
  callback: FunctionHookCallback
  errorMessage: string
  statusMessage?: string
}
```

这类 Hook 的要点是：

- 不可持久化到 settings
- 只能在当前 session 存活
- 适合内部 runtime 逻辑，而不是用户静态配置

### 6.3 matcher + skillRoot 分桶，避免来源混淆

真正存储时，`addHookToSession()` 不是简单 append：

```ts
const existingMatcherIndex = eventMatchers.findIndex(
  m => m.matcher === matcher && m.skillRoot === skillRoot,
)
```

这说明 session hook 的组织维度至少有三层：

- event
- matcher
- skillRoot

结果就是：

- 同一事件下可以有多组 matcher
- 同名 matcher 但来自不同 skillRoot，不会被混成一组
- skill-scoped hook 在 session 中仍然保留来源边界

### 6.4 为什么 `getSessionHooks()` 和 `getSessionFunctionHooks()` 分开

这两个函数的拆分也很重要：

- `getSessionHooks()`
  只返回可转成常规 HookMatcher 的配置型 hooks
- `getSessionFunctionHooks()`
  单独返回 function hooks

这说明产品有一个非常清楚的边界：

> 可持久化的 Hook 配置格式，与只存在于内存里的 callback hook，不强行混成一种序列化结构。

## 7. `registerSkillHooks.ts`：Skill frontmatter 到 session hook 的桥

源码：`src/utils/hooks/registerSkillHooks.ts:20-64`

这是 Day 7 里非常典型的“侧车扩展入口”。

Skill 自带的 hooks，不是写回全局 settings，而是注册到当前 session：

```ts
export function registerSkillHooks(
  setAppState,
  sessionId,
  hooks,
  skillName,
  skillRoot?,
): void {
  for (const eventName of HOOK_EVENTS) {
    const matchers = hooks[eventName]
    if (!matchers) continue

    for (const matcher of matchers) {
      for (const hook of matcher.hooks) {
        const onHookSuccess = hook.once
          ? () => {
              removeSessionHook(setAppState, sessionId, eventName, hook)
            }
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
      }
    }
  }
}
```

这里最值得记的是 `once: true` 语义。

它不是“尝试一次就删”，而是：

> 挂一个 `onHookSuccess`，只有执行成功后才移除这个 session hook。

所以 one-shot hook 的真实语义是“成功消费一次”，不是“无论成败只试一次”。

Skill Hook 因此具备三个特征：

- 生命周期是 session-scoped
- 来源边界保留在 `skillRoot`
- 可以做一次性 sidecar 行为，而不用污染全局配置

## 8. `postSamplingHooks.ts`：另一条纯 sidecar 通道

源码：`src/utils/hooks/postSamplingHooks.ts:31-70`

这个文件要特别和前面的 `HOOK_EVENTS` 体系区分开。

它不是用户在 `settings.json` 里可配置的 Hook，而是内部程序化注册的 sidecar：

```ts
export type REPLHookContext = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  querySource?: QuerySource
}

const postSamplingHooks: PostSamplingHook[] = []

export function registerPostSamplingHook(hook: PostSamplingHook): void {
  postSamplingHooks.push(hook)
}

export async function executePostSamplingHooks(...) {
  const context: REPLHookContext = { ... }
  for (const hook of postSamplingHooks) {
    try {
      await hook(context)
    } catch (error) {
      logError(toError(error))
    }
  }
}
```

这里要抓住 3 个边界：

1. 它的输入是完整 REPL 上下文，不是某个单一生命周期事件 payload
2. 它没有结构化权限决策返回值，返回类型只有 `void`
3. 它的异常只记录日志，不会把主流程打断

调用点在 `src/query.ts`：

```ts
if (assistantMessages.length > 0) {
  void executePostSamplingHooks(
    [...messagesForQuery, ...assistantMessages],
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    querySource,
  )
}
```

所以它的真实语义是：

> 模型采样结束后，异步起一批纯 sidecar 后处理任务，不阻塞本轮 query 主状态机。

这就是为什么 session memory、magic docs、skill improvement 都能挂在这里：

- `src/services/SessionMemory/sessionMemory.ts`
- `src/services/MagicDocs/magicDocs.ts`
- `src/utils/hooks/skillImprovement.ts`

## 9. `toolExecution.ts`：Hook 接入主执行链前的准备带

源码：`src/services/tools/toolExecution.ts:492-760`

你给的这段范围，严格说还没完全进入 Hook 执行本体，
但它对理解 Hook 非常重要，因为它决定了：

> Hook 看到的输入，到底是模型原始输入，还是系统整形后的可观察输入。

### 9.1 `streamedCheckPermissionsAndCallTool()` 只是流式包装

先看入口：

```ts
function streamedCheckPermissionsAndCallTool(...) {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(..., progress => {
    stream.enqueue({
      message: createProgressMessage({
        toolUseID: progress.toolUseID,
        parentToolUseID: toolUseID,
        data: progress.data,
      }),
    })
  })
    .then(results => {
      for (const result of results) {
        stream.enqueue(result)
      }
    })
    .finally(() => {
      stream.done()
    })
  return stream
}
```

这个包装层只是在做一件事：

- 把“工具进度消息”和“最终 tool result”统一塞回一个 async iterable

真正的逻辑还是在 `checkPermissionsAndCallTool()`。

### 9.2 Hook 前一定先做输入校验

`checkPermissionsAndCallTool()` 一开始先做的是 `zod` 校验和工具级输入验证：

```ts
const parsedInput = tool.inputSchema.safeParse(input)
if (!parsedInput.success) {
  ...
  return [tool_use_error]
}

const isValidCall = await tool.validateInput?.(
  parsedInput.data,
  toolUseContext,
)
if (isValidCall?.result === false) {
  ...
  return [tool_use_error]
}
```

也就是说，Hook 并不是“最先碰到模型原始 tool input 的模块”。

在产品里，至少先有两层前置边界：

1. schema 层：参数类型是否合法
2. tool 自己的 validateInput：参数值是否合法

### 9.3 Hook 看到的不是完全原始输入

接着看这一段：

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

这里是 Day 7 很容易忽略、但非常关键的一段。

它说明系统明确区分了两份输入：

- `callInput`
  最终传给 `tool.call()` 的实际调用输入
- `processedInput`
  给 Hook、permission、可观测性等外围控制面看的输入

为什么要这样做？

- 某些工具需要给外部观察者补齐派生字段
- 但这些派生/兼容字段不应该反向污染真正的 `tool.call()` 语义
- 否则会影响 transcript 序列化、fixture 哈希、工具结果文本等稳定性

所以这里的设计非常成熟：

> 产品版明确区分“调用语义”与“观察语义”。

### 9.4 `PreToolUse` 真正进入点在 760 行之后

虽然你的指定区间停在 760，但要真正看懂 Day 7，必须接着看下面几行：

```ts
for await (const result of runPreToolUseHooks(
  toolUseContext,
  tool,
  processedInput,
  toolUseID,
  assistantMessage.message.id,
  requestId,
  mcpServerType,
  mcpServerBaseUrl,
)) {
  ...
}
```

主循环并没有直接解释每个 Hook 的细节，而是只消费一种“桥接后结果”：

- `message`
- `hookPermissionResult`
- `hookUpdatedInput`
- `preventContinuation`
- `stopReason`
- `additionalContext`
- `stop`

这就是 Hook 接入主循环时最核心的产品边界：

> `toolExecution.ts` 只认识桥接后的统一结果，不认识底层每类 Hook 的执行细节。

## 10. 为什么还必须补看 `toolHooks.ts`

虽然 Day 7 阅读项没点到它，但不看这层就会误以为：

- Hook 可以直接替代 permission
- Hook 的结果会原样落到 loop

实际上它们之间隔着一个桥接层：`src/services/tools/toolHooks.ts`

### 10.1 `PreToolUse` 的桥接结果不是原始 HookResult

看 `runPreToolUseHooks()`：

```ts
for await (const result of executePreToolHooks(...)) {
  if (result.blockingError) {
    yield {
      type: 'hookPermissionResult',
      hookPermissionResult: {
        behavior: 'deny',
        message: denialMessage,
        decisionReason: { type: 'hook', ... },
      },
    }
  }

  if (result.permissionBehavior === 'allow') {
    yield { type: 'hookPermissionResult', ... }
  }

  if (result.updatedInput && result.permissionBehavior === undefined) {
    yield { type: 'hookUpdatedInput', updatedInput: result.updatedInput }
  }
}
```

这段说明：

- 底层 HookResult 会被翻译成工具执行器能吃的离散信号
- `blockingError` 会被上升成一条权限拒绝结果
- `updatedInput` 如果只是 passthrough 修改，会单独回传

### 10.2 Hook 的 allow 不能越权绕过规则系统

最关键的是 `resolveHookPermissionDecision()`：

```ts
if (hookPermissionResult?.behavior === 'allow') {
  const hookInput = hookPermissionResult.updatedInput ?? input
  const ruleCheck = await checkRuleBasedPermissions(
    tool,
    hookInput,
    toolUseContext,
  )
  if (ruleCheck === null) {
    return { decision: hookPermissionResult, input: hookInput }
  }
  if (ruleCheck.behavior === 'deny') {
    return { decision: ruleCheck, input: hookInput }
  }
  return {
    decision: await canUseTool(...),
    input: hookInput,
  }
}
```

这是 Day 7 必须真正吃透的一条产品语义：

> `PreToolUse hook` 的 `allow` 不是最高优先级。  
> 它可以绕过交互式提示，但不能绕过 settings 里的 deny/ask 规则。

也就是说，Hook 可以参与权限决策，但不能篡改权限系统的硬边界。

## 11. 事件矩阵：先抓 Day 7 真正关心的几类

| 事件/通道 | 触发时机 | 输入载荷 | 返回语义 | 是否影响主循环推进 |
|---|---|---|---|---|
| `SessionStart` | 会话启动 / 恢复 / clear / compact | `session_id / cwd / transcript_path / source / model / agent_type` | `additionalContext`、`initialUserMessage`、`watchPaths` | 会影响初始化上下文，但不替代 loop |
| `PreToolUse` | 工具执行前 | `tool_name / tool_input / tool_use_id` | `allow/ask/deny/passthrough`、`updatedInput`、`additionalContext` | 会影响，甚至阻止工具执行 |
| `PostToolUse` | 工具成功后 | `tool_name / tool_input / tool_response / tool_use_id` | `additionalContext`、`updatedMCPToolOutput`、`preventContinuation` | 不回滚当前工具，但可阻止继续推进 |
| `PostToolUseFailure` | 工具失败后 | `tool_name / tool_input / error / is_interrupt` | 追加错误反馈、blocking/non-blocking 信息 | 不改变失败事实，但影响后续反馈 |
| `hookEvents` | Hook 执行期间 | `hookId / hookName / hookEvent / stdout / stderr / output` | `started/progress/response` | 不影响，只提供可观测性 |
| `postSamplingHooks` | 模型采样完成后 | 完整 REPL 上下文 | 无结构化返回，错误仅记录 | 默认不影响，是纯 sidecar |

## 12. 教学版与产品版：最值得刻意对照的差异

### 12.1 事件数量

- 教学版：只讲 3 个事件，优先建立心智
- 产品版：几十种事件，覆盖生命周期、工具、任务、多 agent、环境变化

### 12.2 返回协议

- 教学版：统一退出码 `0 / 1 / 2`
- 产品版：结构化 JSON + 附加 attachment + permission bridge

### 12.3 执行方式

- 教学版：顺序跑 Hook，谁先返回谁影响后续
- 产品版：并行执行 Hook，再聚合结果

产品版并行的核心在 `src/utils/hooks.ts`：

```ts
// Run all hooks in parallel with individual timeouts
const hookPromises = matchingHooks.map(async function* (...) { ... })
```

这一点很重要，因为它会影响：

- timing 统计看的是 wall-clock，不是单个 Hook duration 相加
- 多个 Hook 的阻塞、附加上下文、updatedInput 要走聚合逻辑

### 12.4 Hook 来源

- 教学版：`.hooks.json`
- 产品版：
  - settings snapshot
  - registered hooks
  - session hooks
  - function hooks
  - skill frontmatter hooks
  - 内部 callback / post-sampling sidecar

## 13. 第一遍读源码时最容易犯的错

### 13.1 把 `hookEvents.ts` 当成 Hook 执行器

不是。它只是执行事件总线。

### 13.2 把 `postSamplingHooks` 当成 settings hook 的一种

不是。它是内部程序化 sidecar，不走 `HOOK_EVENTS` 配置系统。

### 13.3 以为 Hook 的 `allow` 能覆盖权限系统

不行。`resolveHookPermissionDecision()` 明确保留规则系统硬边界。

### 13.4 以为 Hook 总是串行执行

教学版是，产品版不是。产品版执行和结果聚合明显更复杂。

### 13.5 以为 Hook 修改 input 就一定会改到真实 tool call

不一定。产品版明确区分 `processedInput` 与 `callInput`。

## 14. 一句话总结

Day 7 可以压成一句话：

> `hookEvents.ts` 提供执行观测面，`AsyncHookRegistry.ts` 提供异步回收面，`sessionHooks.ts` 与 `registerSkillHooks.ts` 提供运行时注入面，`postSamplingHooks.ts` 提供采样后 sidecar 面，而 `toolExecution.ts` / `toolHooks.ts` 负责把这些旁路扩展安全地接回主工具执行链。

## 15. 推荐自测题

读完后，你至少应该能回答下面 6 个问题：

1. `hookEvents.ts` 里的 event，和业务 Hook event，是不是同一层？
2. 为什么异步 Hook 需要 registry，而不是直接在原地 await 完？
3. `sessionHooks.ts` 为什么要专门支持 `FunctionHook`？
4. `registerSkillHooks.ts` 里 `once: true` 的真实语义是什么？
5. `postSamplingHooks.ts` 为什么不应该和 `PreToolUse` 混成一类？
6. 为什么产品版要把 `processedInput` 和 `callInput` 分开？
