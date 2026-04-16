# Day 6 教学讲义：Permission Gate 与 Sandbox

> 这份讲义面向第一次读真实源码的人。  
> 目标不是一次吃掉所有产品分支，而是先建立一条稳定主线：
>
> **tool call 先过 Permission Gate，再决定是否进入 Sandbox，最后才真正执行。**

## 0. 这一天到底在学什么

到了 Day 6，agent 已经不只是“会调工具”了。

真正的问题开始变成：

- 模型提了一个动作，系统凭什么就执行它？
- 哪些动作必须问人？
- 哪些 Bash 命令可以直接跑，但必须放进 OS 级沙箱？
- 哪些风险不能只靠规则匹配，而要靠更硬的执行隔离？

所以，这一天不是在学更多工具，而是在学两层安全边界：

```text
第一层：Permission Gate
  决定 这次动作能不能做

第二层：Sandbox
  决定 就算能做，最多能碰到哪里
```

对应源码：

- `src/utils/permissions/permissions.ts`
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`
- `src/tools/BashTool/shouldUseSandbox.ts`
- `src/utils/sandbox/sandbox-adapter.ts`

## 1. 推荐阅读顺序

第一次读，不要按文件树乱跳。  
建议按下面顺序读：

1. `permissions.ts:1071-1319`
   先看最核心的权限判定骨架。
2. `permissions.ts:473-945`
   再看外层包装，理解 `dontAsk`、`auto`、classifier、headless 分支。
3. `interactiveHandler.ts:57-430`
   理解“要 ask 时，系统怎么收口成一个最终答案”。
4. `shouldUseSandbox.ts:21-153`
   理解“为什么 Bash 还要单独判断是否进沙箱”。
5. `sandbox-adapter.ts:99-404,422-479,532-927`
   最后看沙箱适配层，理解设置是怎么翻译成真实 OS 级限制的。

如果只记一条主线，可以先记这张图：

```text
assistant emits tool_use
  ->
permissions.ts
  -> deny | ask | allow
  ->
if ask:
  interactiveHandler.ts
  -> local / bridge / channel / hook 谁先决定谁赢
  ->
if tool is Bash and will run:
  shouldUseSandbox.ts
  -> 要不要进沙箱
  ->
sandbox-adapter.ts
  -> 把 settings 翻译成真正的文件系统/网络限制
  ->
execute
```

## 2. 先建立最小心智模型

第一次读这些源码，最容易混掉三件事：

1. “权限判断” 和 “真正执行” 不是一回事。
2. “ask 用户” 不是一个按钮，而是一套多路竞态协调。
3. “沙箱” 不是再做一次字符串规则判断，而是 OS 级隔离。

所以可以先把 Day 6 理解成四层：

```text
tool_use 到来
  ->
Permission Gate
  - deny?
  - ask?
  - allow?
  ->
Ask Orchestration
  - 谁来批准
  ->
Sandbox Decision (Bash only)
  - 要不要进沙箱
  ->
Sandbox Runtime
  - 能读哪儿
  - 能写哪儿
  - 能访问哪些域名
```

下面按这个顺序读源码。

## 3. `permissions.ts`：真正的权限闸门

### 3.1 第一段核心骨架：`checkRuleBasedPermissions()`

源码位置：

- `src/utils/permissions/permissions.ts:1071-1155`

可以先只看这个删减后的骨架：

```ts
export async function checkRuleBasedPermissions(tool, input, context) {
  const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
  if (denyRule) {
    return { behavior: 'deny' }
  }

  const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
  if (askRule) {
    const canSandboxAutoAllow =
      tool.name === BASH_TOOL_NAME &&
      SandboxManager.isSandboxingEnabled() &&
      SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
      shouldUseSandbox(input)

    if (!canSandboxAutoAllow) {
      return { behavior: 'ask' }
    }
  }

  const parsedInput = tool.inputSchema.parse(input)
  const toolPermissionResult = await tool.checkPermissions(parsedInput, context)

  if (toolPermissionResult?.behavior === 'deny') {
    return toolPermissionResult
  }

  if (
    toolPermissionResult?.behavior === 'ask' &&
    toolPermissionResult.decisionReason?.type === 'rule'
  ) {
    return toolPermissionResult
  }

  if (
    toolPermissionResult?.behavior === 'ask' &&
    toolPermissionResult.decisionReason?.type === 'safetyCheck'
  ) {
    return toolPermissionResult
  }

  return null
}
```

这段代码的意义不是复杂，而是顺序很硬：

1. 先看整工具 deny rule。
2. 再看整工具 ask rule。
3. 再交给工具自己做更细的权限判断。
4. 工具自己给出的 `deny`、内容级 `ask`、`safetyCheck` 都是高优先级结果。

最值得注意的是这个例外：

```ts
const canSandboxAutoAllow =
  tool.name === BASH_TOOL_NAME &&
  SandboxManager.isSandboxingEnabled() &&
  SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
  shouldUseSandbox(input)
```

人话解释：

> 如果这是一条 Bash，而且它会进入沙箱，并且策略允许“沙箱内 Bash 自动放行”，那就不要急着因为整工具 ask rule 而拦下来。

这就是 Permission Gate 和 Sandbox 的第一个连接点。

### 3.2 第二段核心骨架：`hasPermissionsToUseToolInner()`

源码位置：

- `src/utils/permissions/permissions.ts:1158-1319`

删减后的主线是：

```ts
async function hasPermissionsToUseToolInner(tool, input, context) {
  const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
  if (denyRule) return { behavior: 'deny' }

  const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
  if (askRule && !canSandboxAutoAllow) {
    return { behavior: 'ask' }
  }

  const parsedInput = tool.inputSchema.parse(input)
  const toolPermissionResult = await tool.checkPermissions(parsedInput, context)

  if (toolPermissionResult?.behavior === 'deny') {
    return toolPermissionResult
  }

  if (tool.requiresUserInteraction?.() &&
      toolPermissionResult?.behavior === 'ask') {
    return toolPermissionResult
  }

  if (toolPermissionResult?.decisionReason?.type === 'rule') {
    return toolPermissionResult
  }

  if (toolPermissionResult?.decisionReason?.type === 'safetyCheck') {
    return toolPermissionResult
  }

  const shouldBypassPermissions =
    appState.toolPermissionContext.mode === 'bypassPermissions' ||
    (appState.toolPermissionContext.mode === 'plan' &&
      appState.toolPermissionContext.isBypassPermissionsModeAvailable)

  if (shouldBypassPermissions) {
    return { behavior: 'allow' }
  }

  const alwaysAllowedRule = toolAlwaysAllowedRule(...)
  if (alwaysAllowedRule) {
    return { behavior: 'allow' }
  }

  return toolPermissionResult.behavior === 'passthrough'
    ? { behavior: 'ask' }
    : toolPermissionResult
}
```

第一次读时，只要看懂下面这张图就够了：

```text
deny rule
  ->
ask rule
  ->
tool.checkPermissions()
  ->
tool-specific deny / ask / safetyCheck
  ->
mode allow?
  ->
always-allow rule?
  ->
剩下的变成 ask
```

这里最容易读错的地方有两个。

#### 第一，不是 mode 最大，而是某些安全结果比 mode 更大

这段代码明确在 mode 之前就拦住了两类 ask：

- 内容级 ask rule
- `safetyCheck`

也就是说，真实系统不是：

```text
模式说 allow
=> 一切都 allow
```

而是：

```text
如果工具自己已经判断“这里有安全边界”
=> mode 也不能把它直接冲掉
```

这就是为什么文档里会反复强调：

> Permission Gate 是独立 gate，不是某个 hook 顺手做一下。

#### 第二，`passthrough` 不等于 allow

很多初学者会把 `passthrough` 看成“没事，通过”。

这里不是。

这里的意思更接近：

> 我还没决定，继续交给后面的 gate 判断。

到最后，如果它仍然没有被明确 allow，就会被转换成 `ask`。

所以默认姿势不是“放行”，而是“需要确认”。

### 3.3 外层增强版：`hasPermissionsToUseTool()`

源码位置：

- `src/utils/permissions/permissions.ts:473-945`

这段是“基础权限结果”之上的增强包装。

删减后的关键逻辑：

```ts
export const hasPermissionsToUseTool = async (...) => {
  const result = await hasPermissionsToUseToolInner(tool, input, context)

  if (result.behavior === 'allow') {
    return result
  }

  if (result.behavior === 'ask') {
    if (appState.toolPermissionContext.mode === 'dontAsk') {
      return { behavior: 'deny' }
    }

    if (feature('TRANSCRIPT_CLASSIFIER') &&
        appState.toolPermissionContext.mode === 'auto') {
      // 先走 acceptEdits fast-path
      // 再走 allowlist fast-path
      // 最后才跑 classifier
    }

    if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
      const hookDecision = await runPermissionRequestHooksForHeadlessAgent(...)
      if (hookDecision) return hookDecision
      return { behavior: 'deny' }
    }
  }

  return result
}
```

可以把这层理解成：

```text
基础权限门已经说了 ask
  ->
现在看当前运行模式怎么处理 ask
```

这里主要多出来三类现实分支。

#### `dontAsk`

如果当前模式是不允许弹审批框，那所有 ask 都会被改成 deny。

也就是：

```text
不是“以后再问”
而是“现在没法问，所以拒绝”
```

#### `auto mode` + classifier

这里最容易让第一次读源码的人发懵，因为分支很多。

其实先抓住顺序就行：

1. 如果某些安全检查明确要求人工批准，就不要进 classifier。
2. 如果 `acceptEdits` 模式下本来就会 allow，那直接 allow，省一次 classifier 成本。
3. 如果工具在 safe allowlist 里，也直接 allow。
4. 前三条都不满足，才跑 classifier。

这说明 classifier 在系统里不是第一道门，而是：

> 基础规则还没判死，但又不想每次都问用户时，拿来做自动决策的附加层。

#### headless / background agent

如果当前上下文不能弹审批框：

```ts
if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
  const hookDecision = await runPermissionRequestHooksForHeadlessAgent(...)
  if (hookDecision) return hookDecision
  return { behavior: 'deny' }
}
```

人话解释：

> 先让 hook 试试能不能代替用户做决定。  
> 如果连 hook 都没有结论，那就拒绝，不要装作批准过了。

## 4. `interactiveHandler.ts`：ask 不是一个按钮，而是一场竞态协调

源码位置：

- `src/hooks/toolPermission/handlers/interactiveHandler.ts:57-430`

第一次读这段，很容易以为它只是“弹个框”。  
其实不是，它更像一个审批总线。

### 4.1 最关键的入口

删减后的骨架：

```ts
function handleInteractivePermission(params, resolve) {
  const { resolve: resolveOnce, isResolved, claim } = createResolveOnce(resolve)

  ctx.pushToQueue({
    onAbort() { ... },
    async onAllow(updatedInput, permissionUpdates) { ... },
    onReject(feedback) { ... },
    async recheckPermission() {
      const freshResult = await hasPermissionsToUseTool(...)
      if (freshResult.behavior === 'allow') {
        resolveOnce(ctx.buildAllow(...))
      }
    },
  })
}
```

这段里最重要的不是 `onAllow`，而是 `claim()`。

它表达的是：

> 这次 ask 可能有多个来源同时给答案，但最终只能有一个来源赢。

为什么要这样？

因为一个 ask 可能同时从这些地方拿到答复：

- 本地 CLI 用户
- Web bridge
- Telegram / iMessage 之类的 channel
- hooks
- 重新检查权限后的自动放行

如果没有 `claim()` 这种“原子抢答”机制，系统就可能出现：

- 本地点了 allow
- 手机上又回了 reject
- hook 又返回了 allow

然后状态就乱掉。

### 4.2 bridge 和 channel 其实是在并发抢答

bridge 的核心片段：

```ts
bridgeCallbacks.sendRequest(...)

const unsubscribe = bridgeCallbacks.onResponse(
  bridgeRequestId,
  response => {
    if (!claim()) return

    if (response.behavior === 'allow') {
      resolveOnce(ctx.buildAllow(...))
    } else {
      resolveOnce(ctx.cancelAndAbort(...))
    }
  },
)
```

channel 的核心片段：

```ts
const mapUnsub = channelCallbacks.onResponse(
  channelRequestId,
  response => {
    if (!claim()) return

    if (response.behavior === 'allow') {
      resolveOnce(ctx.buildAllow(displayInput))
    } else {
      resolveOnce(ctx.cancelAndAbort(...))
    }
  },
)
```

读法非常简单：

```text
把 ask 发到多个审批入口
  ->
谁先给出有效答案
  ->
谁赢
  ->
其他入口自动失效
```

所以 `interactiveHandler.ts` 的本质不是 UI，而是：

**对 ask 的多路协调。**

### 4.3 `recheckPermission()` 为什么重要

这段第一次读很容易忽略：

```ts
async recheckPermission() {
  const freshResult = await hasPermissionsToUseTool(...)
  if (freshResult.behavior === 'allow') {
    resolveOnce(ctx.buildAllow(...))
  }
}
```

它的意思是：

> ask 弹出来之后，环境可能变了。  
> 比如用户切了 mode，或配置改了。  
> 这时可以重新跑一遍权限判断，如果已经能 allow，就不需要继续卡在 ask 上。

这很像“审批弹窗期间，系统状态不是冻结的”。

## 5. `shouldUseSandbox.ts`：Bash 为什么要单独走一条判断线

源码位置：

- `src/tools/BashTool/shouldUseSandbox.ts:21-153`

这文件很短，但很关键。

先看最核心的函数：

```ts
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }

  if (
    input.dangerouslyDisableSandbox &&
    SandboxManager.areUnsandboxedCommandsAllowed()
  ) {
    return false
  }

  if (!input.command) {
    return false
  }

  if (containsExcludedCommand(input.command)) {
    return false
  }

  return true
}
```

这段非常值得用一句人话记住：

> 它不是在判断“这条命令危险不危险”，而是在判断“这条命令执行时要不要被沙箱包起来”。

也就是说，它关心的是执行边界，不是安全语义本身。

### 5.1 `containsExcludedCommand()` 真正在做什么

删减后的关键逻辑：

```ts
function containsExcludedCommand(command: string): boolean {
  const userExcludedCommands = settings.sandbox?.excludedCommands ?? []
  if (userExcludedCommands.length === 0) return false

  const subcommands = splitCommand_DEPRECATED(command)

  for (const subcommand of subcommands) {
    const candidates = [trimmed]
    // 迭代剥离前置 env vars 和 wrapper
    // 然后拿这些候选去匹配 excluded patterns
  }

  return false
}
```

这段在做两件很务实的事：

1. 复合命令会拆开检查，避免 `a && b` 混过去。
2. 会剥掉前置环境变量和一些 wrapper，让匹配更像“真实命令意图”。

例如：

```text
FOO=bar timeout 30 bazel run //app
```

它希望最后仍然能命中类似：

```text
bazel:*
```

但这层作者也写得很清楚：

> 这不是安全边界。  
> 这是配置便利性和产品体验上的辅助判断。

真正的安全边界在沙箱运行时，而不是这里。

## 6. `sandbox-adapter.ts`：把 Claude 的配置翻译成真正的 OS 级沙箱

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:99-404`
- `src/utils/sandbox/sandbox-adapter.ts:422-479`
- `src/utils/sandbox/sandbox-adapter.ts:532-927`

如果说 `permissions.ts` 是“脑子”，  
那 `sandbox-adapter.ts` 更像“接线员”：

> 把 Claude Code 自己的 settings / permission rules，翻译成 sandbox runtime 真能执行的配置。

### 6.1 最核心函数：`convertToSandboxRuntimeConfig()`

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:172-380`

删减后的核心结构：

```ts
export function convertToSandboxRuntimeConfig(settings) {
  const allowedDomains: string[] = []
  const deniedDomains: string[] = []

  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  const denyRead: string[] = []
  const allowRead: string[] = []

  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))

  if (worktreeMainRepoPath && worktreeMainRepoPath !== cwd) {
    allowWrite.push(worktreeMainRepoPath)
  }

  for (const source of SETTING_SOURCES) {
    // 从 permissions.allow / deny 里提取 Edit / Read 路径
    // 从 sandbox.filesystem.* 里提取 allowWrite / denyWrite / ...
  }

  return {
    network: { allowedDomains, deniedDomains, ... },
    filesystem: { denyRead, allowRead, allowWrite, denyWrite },
    ...
  }
}
```

第一次读这段时，抓住两点就够了。

#### 第一，它在做“翻译”，不是在做新的权限决策

输入是 Claude 自己的设置：

- `permissions.allow`
- `permissions.deny`
- `sandbox.network.*`
- `sandbox.filesystem.*`

输出是 sandbox runtime 需要的配置：

- `allowedDomains`
- `allowWrite`
- `denyWrite`
- `denyRead`

所以这层不再回答“该不该执行”，而是在回答：

> 如果执行，要给操作系统什么限制配置。

#### 第二，它很重视“默认安全底座”

即使你没显式加很多规则，它也会主动补一些非常硬的保护：

```ts
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

人话解释：

- 不允许沙箱里的命令改 settings 文件
- 不允许改 managed settings 目录
- 不允许改 `.claude/skills`

为什么 `.claude/skills` 这么重要？

因为技能本身也是高权限入口。  
如果沙箱里的命令能偷偷改技能定义，那等于绕开了更高层的控制。

### 6.2 一段非常值得学的安全细节：bare git repo 防护

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:257-279`
- `src/utils/sandbox/sandbox-adapter.ts:399-404`

删减后的片段：

```ts
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of [originalCwd, cwd]) {
  for (const gitFile of bareGitRepoFiles) {
    const p = resolve(dir, gitFile)
    try {
      statSync(p)
      denyWrite.push(p)
    } catch {
      bareGitRepoScrubPaths.push(p)
    }
  }
}
```

这段第一次读会觉得很怪：

> 为什么沙箱代码要关心 cwd 下有没有 `HEAD`、`objects`、`refs` 这种文件？

原因是作者在补一个真实攻击面：

```text
如果攻击者在 cwd 里伪造出 bare git repo 结构
后续某些不在沙箱内运行的 git 调用
就可能把这里误认成 git 仓库，从而走到危险路径
```

所以这里的防护分两层：

- 如果这些路径已经存在，就直接 denyWrite。
- 如果现在还不存在，就记下来，等沙箱命令跑完后 scrub 掉。

这类代码很有 Day 6 的味道：

> 不是抽象安全课，而是产品在真实实现里踩过坑后加上的硬防线。

### 6.3 `isSandboxingEnabled()`：不是开关开了就算数

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:532-547`

核心片段：

```ts
function isSandboxingEnabled(): boolean {
  if (!isSupportedPlatform()) return false
  if (checkDependencies().errors.length > 0) return false
  if (!isPlatformInEnabledList()) return false
  return getSandboxEnabledSetting()
}
```

这段非常适合第一次读时记成一句话：

> `sandbox.enabled: true` 只是用户意图，  
> 不是运行时一定能真的启用。

真正启用要同时满足：

- 平台支持
- 依赖齐全
- 平台在允许列表内
- 配置里明确开启

也就是说，沙箱不是“布尔配置”，而是“配置 + 环境能力”的交集。

### 6.4 `getSandboxUnavailableReason()`：为什么这段很重要

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:549-592`

这段的意义不在算法，而在产品态度：

> 用户明明打开了 sandbox，但实际上跑不起来时，系统不能默默降级成“没沙箱也照跑”，必须告诉用户原因。

这就是为什么它会专门返回：

- WSL1 不支持
- 当前平台不支持
- 缺依赖
- 不在 enabledPlatforms 里

这类信息。

### 6.5 `initialize()` 和 `wrapWithSandbox()`

源码位置：

- `src/utils/sandbox/sandbox-adapter.ts:704-803`

删减后的骨架：

```ts
async function wrapWithSandbox(command, binShell, customConfig, abortSignal) {
  if (isSandboxingEnabled()) {
    if (initializationPromise) {
      await initializationPromise
    } else {
      throw new Error('Sandbox failed to initialize.')
    }
  }

  return BaseSandboxManager.wrapWithSandbox(
    command,
    binShell,
    customConfig,
    abortSignal,
  )
}

async function initialize(sandboxAskCallback) {
  if (initializationPromise) return initializationPromise
  if (!isSandboxingEnabled()) return

  initializationPromise = (async () => {
    const runtimeConfig = convertToSandboxRuntimeConfig(settings)
    await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)

    settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
      const newConfig = convertToSandboxRuntimeConfig(settings)
      BaseSandboxManager.updateConfig(newConfig)
    })
  })()

  return initializationPromise
}
```

这段说明了两件事：

1. 沙箱不是每次执行时才从头现配一遍，而是有初始化和配置更新机制。
2. 执行命令前，不是系统自己手搓隔离命令，而是委托到底层 `BaseSandboxManager`。

也就是说：

```text
Claude Code adapter
  负责翻译配置 + 管理初始化

sandbox-runtime
  负责真正的 OS 级沙箱执行
```

## 7. 用一个完整例子把四个文件串起来

假设模型想调用 Bash：

```text
npm test
```

系统里的典型链路是：

### 第一步：过权限门

`permissions.ts` 会先判断：

- 有没有整工具 deny rule
- 有没有 ask rule
- `tool.checkPermissions()` 有没有发现更细粒度的限制
- 当前 mode 是否允许

如果最后结果是 `allow`，才进入下一步。

### 第二步：如果是 ask，就进入 ask 协调

`interactiveHandler.ts` 会把这次 ask 发到：

- 本地审批 UI
- Web bridge
- 可用 channel
- hooks

谁先给出有效结论，谁赢。

### 第三步：Bash 判断是否进沙箱

`shouldUseSandbox.ts` 会判断：

- 当前是否启用 sandbox
- 用户是否显式要求不进沙箱
- 命令是否命中 excludedCommands

大多数正常 Bash 命令，默认都会进沙箱。

### 第四步：沙箱适配层生成真实配置

`sandbox-adapter.ts` 会把当前 settings 翻译成：

- 哪些目录能写
- 哪些目录不能读/写
- 哪些域名能访问
- 哪些本地 socket 或 proxy 允许使用

然后再交给 sandbox runtime 真正执行。

## 8. Bash 为什么要有独立安全通道

这是 Day 6 最重要的一句补充。

文件工具和 Bash 最大的区别是：

- `Read`、`Edit` 这类工具的能力边界很清楚
- Bash 的能力边界几乎是整个系统

所以 Bash 不能只靠 Permission Gate 一次判断就结束，原因有两个：

1. 它的输入不是简单参数，而是一种可执行动作描述。
2. 即使你允许它执行，也仍然希望给它更硬的 OS 级边界。

所以 Bash 的典型安全链是：

```text
permission gate
  ->
shouldUseSandbox
  ->
sandbox runtime
```

这就是为什么 Day 6 的产出项会特别强调：

> 单独列出 Bash 为什么要有独立安全通道。

## 9. 一句话记忆

把 Day 6 压成一句话，就是：

> `permissions.ts` 决定“能不能做”，`interactiveHandler.ts` 决定“要问时怎么问”，`shouldUseSandbox.ts` 决定“Bash 要不要进沙箱”，`sandbox-adapter.ts` 决定“进沙箱后操作系统具体限制什么”。

如果你已经能稳定说清这句话，再回头看这些源码分支，就不容易迷路了。

## 10. 建议你读完后自己回答这三个问题

1. 为什么 `passthrough` 最后会变成 `ask`，而不是 `allow`？
2. 为什么 `safetyCheck` 要放在 mode allow 之前？
3. 为什么 Bash 既需要 Permission Gate，又需要 Sandbox？

如果这三个问题你已经能自己回答，Day 6 的主线就算真正吃进去了。
