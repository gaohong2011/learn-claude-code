# Day 9 教学讲义：System Prompt Pipeline 源码导读

> 这份讲义面向第一次读真实 prompt 装配源码的人。  
> Day 9 最重要的不是背几段提示词，而是看懂一条完整流水线：
>
> **默认 system prompt 先按 section 组装，运行时再决定“最终采用哪套 prompt”，然后把动态上下文做成 attachments / message context，一起送进 API。**

## 0. 这一天到底在学什么

很多人刚接触 agent 时，会把 system prompt 理解成一大段固定文案。

但真实产品里不会这样做，因为下面这些东西都在变化：

- 当前启用了哪些工具
- 用户有没有自定义语言偏好和输出风格
- 当前有没有 agent prompt 覆盖默认 prompt
- MCP server 有没有中途连上来
- memory、skills、IDE 选区、diagnostics 这些上下文要不要补进去
- 本地运行时需要的工具输入，和真正发给 API 的字段，是不是同一份

所以 Day 9 真正要学的是这条链：

```text
prompt sections
  ->
default system prompt
  ->
effective system prompt
  ->
attachments / systemContext / userContext
  ->
normalized API payload
```

翻成人话就是：

> 不是“写 prompt”，而是在做一条“模型输入装配管线”。

## 1. 推荐阅读顺序

建议按下面顺序读，最顺：

1. `docs/zh/s10-system-prompt.md`
   先建立教学版里“prompt 是流水线，不是大字符串”的心智
2. `docs/zh/s10a-message-prompt-pipeline.md`
   再把 prompt / messages / attachments 这几条输入面分开
3. `agents/s10_system_prompt.py`
   先看教学版最小实现，知道“理想模型”长什么样
4. `/Users/hong.gao/python/src/claude-code-codex/src/constants/prompts.ts`
   看真实产品里默认 system prompt 是怎么拼的
5. `/Users/hong.gao/python/src/claude-code-codex/src/constants/systemPromptSections.ts`
   看 section 级缓存是怎么做的
6. `/Users/hong.gao/python/src/claude-code-codex/src/utils/systemPrompt.ts`
   看最后到底采用哪套 prompt
7. `/Users/hong.gao/python/src/claude-code-codex/src/utils/attachments.ts`
   看哪些动态信息不走 system prompt，而走 attachments
8. `/Users/hong.gao/python/src/claude-code-codex/src/utils/api.ts`
   最后看 API 前怎么补 context、怎么清洗 tool input

原因很简单：

- `prompts.ts` 告诉你“默认 prompt 长什么样”
- `systemPrompt.ts` 告诉你“最后真正生效的是谁”
- `attachments.ts` 告诉你“还有哪些信息根本不该塞进 system prompt”
- `api.ts` 告诉你“真正发给模型前还要再收拾一遍”

## 2. 先背一张总图

先把 Day 9 的主线图背下来：

```text
getSystemPrompt()
  ->
静态 section + 动态 section
  ->
resolveSystemPromptSections()
  ->
default system prompt: string[]

default system prompt
  + custom/agent/override/append
  ->
buildEffectiveSystemPrompt()
  ->
effective system prompt

effective system prompt
  + appendSystemContext(systemContext)
  ->
full system prompt

messages
  + prependUserContext(userContext)
  + getAttachmentMessages(...)
  ->
final messages

final messages + full system prompt + tools
  ->
normalizeToolInputForAPI()
  ->
callModel()
```

如果你只记一句口诀，可以记这个：

```text
默认 prompt 先拼好
最终 prompt 再选好
动态上下文分流进 attachment / context
发 API 前再清洗一次
```

## 3. `prompts.ts`：默认 system prompt 不是一个字符串，而是一堆 section

### 3.1 先看几个最小 section builder

`127-167` 这一段看起来像小工具函数，其实很重要。

例如：

```ts
function getLanguageSection(
  languagePreference: string | undefined,
): string | null {
  if (!languagePreference) return null

  return `# Language
Always respond in ${languagePreference}. ...`
}
```

这类函数说明一件事：

- 每一类 prompt 来源单独做成一个 section builder
- 没有值时直接返回 `null`
- 最后统一 `filter(s => s !== null)`

这比写一整坨字符串容易维护得多，因为你能看清：

- 这一段来自哪里
- 什么时候会出现
- 什么时候应该缺席

`prependBullets()` 也值得注意：

```ts
export function prependBullets(items: Array<string | string[]>): string[] {
  return items.flatMap(item =>
    Array.isArray(item)
      ? item.map(subitem => `  - ${subitem}`)
      : [` - ${item}`],
  )
}
```

它在做的不是“语法小技巧”，而是把 prompt 结构标准化。  
真实大系统里，prompt 的稳定排版本身就有价值，因为这会影响 cache 命中和后续 diff 稳定性。

### 3.2 `getSystemPrompt()`：把默认 prompt 拆成“静态前缀 + 动态后缀”

Day 9 的第一段核心代码就是这里：

```ts
const dynamicSections = [
  systemPromptSection('session_guidance', () => ...),
  systemPromptSection('memory', () => loadMemoryPrompt()),
  systemPromptSection('language', () => getLanguageSection(settings.language)),
  DANGEROUS_uncachedSystemPromptSection('mcp_instructions', () => ...),
  systemPromptSection('scratchpad', () => getScratchpadInstructions()),
  systemPromptSection('frc', () => getFunctionResultClearingSection(model)),
]

const resolvedDynamicSections =
  await resolveSystemPromptSections(dynamicSections)

return [
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  getSimpleDoingTasksSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

这里最该看懂的是三层意思：

1. 默认 prompt 被拆成一堆 section，而不是一个大字符串。
2. 前半段偏稳定，后半段偏动态。
3. 中间还专门放了 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 当边界标记。

也就是说，源码里真正关心的不是“文案写得漂不漂亮”，而是：

- 哪些 section 稳定，适合缓存
- 哪些 section 会变，必须重算
- 这些部分怎样拼起来最省 token、最稳定

### 3.3 为什么 `mcp_instructions` 是 `DANGEROUS_uncached`

这一段很能体现真实工程味道：

```ts
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () =>
    isMcpInstructionsDeltaEnabled()
      ? null
      : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
)
```

这里的重点不是函数名吓人，而是它在明确表达：

> 这段内容跨轮可能变化，不能假装它是稳定前缀。

因为 MCP server 可能在对话进行中才连接上来、断开、切换指令。  
如果你把这类 section 也放进普通缓存里，模型拿到的就可能是旧指令。

### 3.4 `computeSimpleEnvInfo()`：环境信息不是默认天生存在，而是显式注入

`651-760` 这段是 Day 9 很容易被忽略，但实际上非常关键的部分：

```ts
const envItems = [
  `Primary working directory: ${cwd}`,
  isWorktree
    ? `This is a git worktree ... Do NOT \`cd\` to the original repository root.`
    : null,
  [`Is a git repository: ${isGit}`],
  additionalWorkingDirectories ? `Additional working directories:` : null,
  additionalWorkingDirectories ? additionalWorkingDirectories : null,
  `Platform: ${env.platform}`,
  getShellInfoLine(),
  `OS Version: ${unameSR}`,
  modelDescription,
  knowledgeCutoffMessage,
].filter(item => item !== null)
```

再拼成：

```ts
return [
  `# Environment`,
  `You have been invoked in the following environment: `,
  ...prependBullets(envItems),
].join(`\n`)
```

这说明一个关键事实：

> 模型并不是天然知道“自己在哪个目录、是不是 git repo、现在是什么 shell、知识截止到哪里”，这些信息都要由系统显式喂进去。

所以 Day 9 不只是 prompt engineering，也是 runtime state injection。

### 3.5 `enhanceSystemPromptWithEnvDetails()`：给已有 prompt 再补一层运行时约束

`760-860` 这里还能看到另一种做法：

```ts
return [
  ...existingSystemPrompt,
  notes,
  ...(discoverSkillsGuidance !== null ? [discoverSkillsGuidance] : []),
  envInfo,
]
```

这表示：

- prompt 不是“一次成型”
- 有时会先得到一个基础 prompt
- 再按场景补“环境细则”和“技能发现指导”

`notes` 里那几条也很典型，例如：

- agent 线程 bash 调用之间 cwd 会重置，所以必须用绝对路径
- 最终回复里要给绝对路径
- 不要用 emoji

这些都不是“业务知识”，而是运行这个 agent 时的环境约束。

## 4. `systemPromptSections.ts`：真正关键的是 section 级缓存

这一段代码很短，但 Day 9 的核心工程价值几乎都在这里：

```ts
export function systemPromptSection(name, compute) {
  return { name, compute, cacheBreak: false }
}

export function DANGEROUS_uncachedSystemPromptSection(name, compute, _reason) {
  return { name, compute, cacheBreak: true }
}

export async function resolveSystemPromptSections(sections) {
  const cache = getSystemPromptSectionCache()

  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null
      }
      const value = await s.compute()
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

把它翻成最朴素的人话：

- 每个 section 都有自己的名字
- 默认可以缓存
- 少数 section 明确声明“不准缓存”
- 解析时优先命中缓存，没命中再重算

这一步为什么重要？

因为真实 agent 的 prompt 很长、很贵、很容易反复重复。  
如果每轮都重拼所有 section：

- token 浪费
- cache 命中下降
- 动态内容还会把稳定内容一起拖着失效

所以最合理的粒度不是“整段 prompt 缓存”，而是“section 级缓存”。

### 4.1 为什么 `/clear` 和 `/compact` 还要清 section 状态

结尾的 `clearSystemPromptSections()` 也别跳过：

```ts
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()
  clearBetaHeaderLatches()
}
```

意思是：

- 新会话不能沿用旧 section 状态
- compact 之后，某些“上一段会话里锁住的判断结果”也要重算

这说明 prompt section cache 不是全局永生缓存，而是有会话生命周期的。

## 5. `systemPrompt.ts`：默认 prompt 拼完了，还要决定“这轮到底用谁”

这是 Day 9 第二个必须看懂的地方。  
`getSystemPrompt()` 只是生成默认 prompt，**不代表它一定会被用上。**

真正决定最终 prompt 的，是 `buildEffectiveSystemPrompt()`：

```ts
if (overrideSystemPrompt) {
  return asSystemPrompt([overrideSystemPrompt])
}

const agentSystemPrompt = mainThreadAgentDefinition
  ? mainThreadAgentDefinition.getSystemPrompt(...)
  : undefined

if (agentSystemPrompt && proactiveMode) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}

return asSystemPrompt([
  ...(agentSystemPrompt
    ? [agentSystemPrompt]
    : customSystemPrompt
      ? [customSystemPrompt]
      : defaultSystemPrompt),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

这段代码建议直接背出优先级：

```text
override
  >
agent prompt
  >
custom system prompt
  >
default system prompt

最后再追加 appendSystemPrompt
```

### 5.1 这里真正解决的问题是什么

真实系统里，prompt 可能来自很多层：

- 框架默认 prompt
- 用户临时传入的 custom prompt
- 当前 agent 自己带的 prompt
- 特殊模式下的 override prompt
- 某些实验/模式要求 append 一小段尾部提示

如果没有这一层统一决策，最后就很容易变成：

- 有的地方覆盖
- 有的地方拼接
- 有的地方替换一半
- 调试时谁生效完全说不清

所以 `buildEffectiveSystemPrompt()` 的价值是：

> 把“多来源 prompt 冲突”变成一个清晰、可预测的优先级系统。

## 6. `attachments.ts`：很多动态上下文根本不该塞进 system prompt

读 Day 9 时，最容易读偏的一点是：

> 以为所有输入来源最终都塞进了 system prompt。

真实情况不是。  
大量动态信息走的是 attachments 通道。

### 6.1 `getAttachments()`：把附件分成三大类

先看最核心结构：

```ts
const userInputAttachments = input ? [
  maybe('at_mentioned_files', () => processAtMentionedFiles(input, context)),
  maybe('mcp_resources', () => processMcpResourceAttachments(input, context)),
  maybe('agent_mentions', () => processAgentMentions(input, ...)),
  maybe('skill_discovery', () => ...),
] : []

const allThreadAttachments = [
  maybe('queued_commands', () => getQueuedCommandAttachments(queuedCommands)),
  maybe('changed_files', () => getChangedFiles(context)),
  maybe('nested_memory', () => getNestedMemoryAttachments(context)),
  maybe('dynamic_skill', () => getDynamicSkillAttachments(context)),
  maybe('skill_listing', () => getSkillListingAttachments(context)),
  maybe('todo_reminders', () => ...),
]

const mainThreadAttachments = isMainThread ? [
  maybe('ide_selection', () => getSelectedLinesFromIDE(...)),
  maybe('diagnostics', () => getDiagnosticAttachments(toolUseContext)),
  maybe('token_usage', () => getTokenUsageAttachment(...)),
] : []
```

这三类分别代表：

1. 用户输入触发的附件  
   例如 `@文件`、MCP resource、agent mention、skill discovery
2. 所有线程都可能有的附件  
   例如 changed files、nested memory、dynamic skill、task reminder
3. 只属于主线程的附件  
   例如 IDE 选区、diagnostics、token usage

这就是 Day 9 特别重要的一层边界：

> “模型输入上下文”不等于“一段 system prompt 文本”；其中有相当大一块是按时机生成的附件消息。

### 6.2 `maybe()`：附件是独立容错的

`maybe()` 也值得专门看一下：

```ts
async function maybe(label: string, f: () => Promise<A[]>): Promise<A[]> {
  try {
    const result = await f()
    ...
    return result
  } catch (e) {
    logError(e)
    return []
  }
}
```

意思很直接：

- 每个附件自己算
- 自己记日志
- 自己失败就自己吞掉，返回空数组

这样某一类附件挂了，不会把整轮 query 拖死。

### 6.3 `memoryFilesToAttachments()`：nested memory 注入时为什么还要做 read state 标记

`1711-1793` 这里能看到内存/规则类文件注入时的细节：

```ts
if (toolUseContext.loadedNestedMemoryPaths?.has(memoryFile.path)) {
  continue
}
if (!toolUseContext.readFileState.has(memoryFile.path)) {
  attachments.push({
    type: 'nested_memory',
    path: memoryFile.path,
    content: memoryFile,
  })
  toolUseContext.loadedNestedMemoryPaths?.add(memoryFile.path)

  toolUseContext.readFileState.set(memoryFile.path, {
    content: memoryFile.contentDiffersFromDisk
      ? (memoryFile.rawContent ?? memoryFile.content)
      : memoryFile.content,
    isPartialView: memoryFile.contentDiffersFromDisk,
  })
}
```

这里不是单纯“把 `CLAUDE.md` 内容塞进上下文”这么简单。

还做了两件事：

1. 去重  
   同一份 nested memory 不要反复注入
2. 标记 read state  
   告诉系统：模型看到的这份内容是不是完整磁盘视图

第二点很重要，因为后续如果模型要 edit/write 某个文件，系统得知道：

- 它之前看到的是完整内容
- 还是裁剪过、删过注释、去过 frontmatter 的 partial view

这属于“prompt 注入”和“文件编辑安全边界”的联动。

### 6.4 `getRelevantMemoryAttachments()`：相关 memory 是按需检索，不是全部塞回去

`2197-2380` 里最核心的是：

```ts
const selected = allResults
  .flat()
  .filter(m => !readFileState.has(m.path) && !alreadySurfaced.has(m.path))
  .slice(0, 5)

const memories = await readMemoriesForSurfacing(selected, signal)

if (memories.length === 0) {
  return []
}
return [{ type: 'relevant_memories' as const, memories }]
```

这说明 auto memory 回注不是：

- 把 `.memory/` 全量塞进 prompt

而是：

- 先根据当前输入做相关性筛选
- 去掉已经看过的和已经 surfacing 过的
- 最多只取前 5 个

再看 `readMemoriesForSurfacing()`：

```ts
const result = await readFileInRange(
  filePath,
  0,
  MAX_MEMORY_LINES,
  MAX_MEMORY_BYTES,
  signal,
  { truncateOnByteLimit: true },
)
```

如果太长，还会在结尾补一句：

```ts
Use the FileRead tool to view the complete file at: ${filePath}
```

意思非常清楚：

> memory 注入的目标是“把最相关的部分提醒模型”，不是“把所有 memory 文件全文贴进上下文”。

### 6.5 `getDynamicSkillAttachments()` 和 `getSkillListingAttachments()`：skills 也走附件通道

`2548-2689` 里能看到两个重点。

先是动态扫描 skill 目录：

```ts
const perDirResults = await Promise.all(
  Array.from(toolUseContext.dynamicSkillDirTriggers).map(async skillDir => {
    const entries = await readdir(skillDir, { withFileTypes: true })
    ...
    await stat(resolve(skillDir, name, 'SKILL.md'))
  }),
)
```

这说明 skill 不是写死在 prompt 模板里的，而是可以按目录变化动态发现。

再是 skill listing 去重：

```ts
const agentKey = toolUseContext.agentId ?? ''
let sent = sentSkillNames.get(agentKey)
...
const newSkills = allCommands.filter(cmd => !sent.has(cmd.name))
```

这里做的是：

- 每个 agent 自己维护一份“已经发过哪些 skills”
- 初次注入发全量
- 之后只发增量

这也是典型的 Day 9 思路：

> 稳定知识别重复灌，变化知识按 delta 发送。

### 6.6 `getAttachmentMessages()`：附件最后会变成真正的 message

附件不是停留在内存对象里，它最后会被转成消息：

```ts
const attachments = await getAttachments(...)

for (const attachment of attachments) {
  yield createAttachmentMessage(attachment)
}
```

再看 `messages.ts` 对几类附件的渲染：

```ts
case 'nested_memory': {
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `Contents of ${attachment.content.path}:\n\n${attachment.content.content}`,
      isMeta: true,
    }),
  ])
}

case 'relevant_memories': {
  return wrapMessagesInSystemReminder(
    attachment.memories.map(m => createUserMessage({...}))
  )
}

case 'skill_listing': {
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `The following skills are available ...`,
      isMeta: true,
    }),
  ])
}
```

这一步非常关键：

> 很多动态上下文最终是以“带 `<system-reminder>` 的 meta user message”形式进入模型，而不是被硬拼进 system prompt。

## 7. `api.ts`：system prompt 和 messages 在 API 前还要再补一层 context

### 7.1 `appendSystemContext()`：补到 system prompt 尾部

`437-447` 很直白：

```ts
export function appendSystemContext(
  systemPrompt: SystemPrompt,
  context: { [k: string]: string },
): string[] {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n'),
  ].filter(Boolean)
}
```

也就是说，`systemContext` 会被当成一小段尾部文本，直接追加到 system prompt 数组后面。

在调用现场可以看到：

```ts
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

### 7.2 `prependUserContext()`：补到 messages 最前面

另一条通道是：

```ts
return [
  createUserMessage({
    content: `<system-reminder>\nAs you answer ...\n</system-reminder>\n`,
    isMeta: true,
  }),
  ...messages,
]
```

调用现场是：

```ts
messages: prependUserContext(messagesForQuery, userContext),
systemPrompt: fullSystemPrompt,
```

所以这里有个很值得专门记的点：

```text
systemContext -> system prompt 尾部
userContext   -> messages 头部的 meta user message
attachments   -> 独立 attachment message
```

三者都在补上下文，但入口不一样。

### 7.3 `normalizeToolInput()`：本地运行时用的输入，不一定能直接发给 API

`566-681` 这一大段很像“工具参数处理”，但它的真正价值是区分：

- 本地执行时需要什么字段
- API schema 允许什么字段

比如 `EXIT_PLAN_MODE_V2`：

```ts
const plan = getPlan(agentId)
const planFilePath = getPlanFilePath(agentId)
return plan !== null ? { ...input, plan, planFilePath } : input
```

这说明本地运行时会额外注入 `plan` 和 `planFilePath`，方便 hooks / SDK / transcript 使用。

再看 `BashTool`：

```ts
let normalizedCommand = command.replace(`cd ${cwd} && `, '')
...
normalizedCommand = normalizedCommand.replace(/\\\\;/g, '\\;')
```

也就是会把模型生成的一些“表面正确、实际多余”的命令前缀清掉，规整成本地工具真正想要的输入。

### 7.4 `normalizeToolInputForAPI()`：发给 API 前再剥掉本地私货

这一步是 Day 9 最后一道边界：

```ts
case EXIT_PLAN_MODE_V2_TOOL_NAME: {
  const { plan, planFilePath, ...rest } = input as Record<string, unknown>
  return rest as z.infer<T['inputSchema']>
}
```

还有 `FileEditTool` 的老兼容字段剥离：

```ts
const { old_string, new_string, replace_all, ...rest } =
  input as Record<string, unknown>
return rest as z.infer<T['inputSchema']>
```

这表示：

> 运行时为了方便，可能会给工具输入塞一些内部字段；但发 API 时必须恢复成“接口协议允许的那份干净输入”。

这也是为什么 Day 9 不能只看 prompt 文本，还得看 API 层。

## 8. 用一句人话重新解释整条 Pipeline

如果把这天的源码全都翻成一段最土的话，大概是这样：

1. 先在 `prompts.ts` 里把默认 system prompt 拆成一块一块 section。
2. 用 `systemPromptSections.ts` 对这些 section 做缓存和重算控制。
3. 再在 `systemPrompt.ts` 里决定这一轮最终到底是默认 prompt、生效 agent prompt，还是 override prompt。
4. 与此同时，很多临时上下文不进 system prompt，而是在 `attachments.ts` 里做成附件消息。
5. 最后到 `api.ts`，把 systemContext 和 userContext 分别补到 system prompt / messages 上，再把 tool input 清洗成 API 接受的格式。

## 9. Day 9 最该记住的六类来源

结合教学版和真实源码，可以把 system prompt pipeline 里的主要来源记成这六类：

| 来源 | 在真实源码里的典型位置 | 它回答的问题 |
|---|---|---|
| 核心身份与系统规则 | `getSimpleIntroSection()` / `getSimpleSystemSection()` | 你是谁、基本行为边界是什么 |
| 工具说明 | `getUsingYourToolsSection(enabledTools)` | 你能调用什么工具，怎么调用 |
| skills / skill guidance | `getSkillListingAttachments()` / `getDiscoverSkillsGuidance()` | 你有哪些技能、何时应该发现技能 |
| memory | `loadMemoryPrompt()` / `getRelevantMemoryAttachments()` / `nested_memory` | 哪些历史知识值得继续带着 |
| CLAUDE.md / 指令链 | `nested_memory` 与 memory rule 注入链 | 长期规则和项目说明是什么 |
| 动态环境 | `computeSimpleEnvInfo()` / `appendSystemContext()` / `prependUserContext()` | 当前工作目录、模式、模型、系统状态是什么 |

## 10. 最后收口：Day 9 真正学到的不是 Prompt，而是输入架构

Day 9 最容易犯的错，是把它理解成：

> “哦，这一天就是在看 system prompt 文案。”

其实不是。

这一天真正学到的是：

> **模型输入不是一大段固定文本，而是一个有分层、有缓存、有优先级、有动态附件、有 API 规整步骤的输入架构。**

只要你把这条主线看清，后面的 compact、resume、memory、tool use、subagent，都会更好理解。
