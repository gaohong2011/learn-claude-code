# Day 9 输出物：System Prompt 组装清单

Day 9 最重要的不是背 prompt 文案，而是把“模型输入到底从哪几层来”分清楚。

## 1. 六类来源清单

| 来源 | 最简单理解 | 真实源码里主要在哪 |
|---|---|---|
| 核心身份 | 告诉模型“你是谁、基本该怎么做” | `prompts.ts` 里的 intro / system / tone sections |
| 工具说明 | 告诉模型“你能用什么工具” | `getUsingYourToolsSection(enabledTools)` |
| skills | 告诉模型“有哪些技能可用、何时发现技能” | `enhanceSystemPromptWithEnvDetails()`、`dynamic_skill`、`skill_listing` |
| memory | 告诉模型“哪些历史知识值得继续带着” | `loadMemoryPrompt()`、`nested_memory`、`relevant_memories` |
| CLAUDE.md 指令链 | 告诉模型“长期规则和项目说明是什么” | 通过 memory / nested instruction 文件注入 |
| 动态环境 | 告诉模型“你现在在哪、当前模式是什么” | `computeSimpleEnvInfo()`、`appendSystemContext()`、`prependUserContext()` |

## 2. 一张最该记住的流向图

```text
section builders
  ->
default system prompt
  ->
effective system prompt

dynamic context
  ->
attachments / systemContext / userContext

effective system prompt
  + final messages
  + tools
  ->
normalized API payload
```

## 3. 三条关键边界

### 3.1 默认 prompt 不等于最终 prompt

`getSystemPrompt()` 只是先拼出默认 prompt。  
真正最后用哪份，还要经过 `buildEffectiveSystemPrompt()` 决定：

```text
override > agent > custom > default
然后再 append
```

### 3.2 动态上下文不一定进 system prompt

很多信息不会直接塞进 system prompt，而是走：

- `attachments`
- `systemContext`
- `userContext`

这比把所有内容硬拼进 prompt 更清楚，也更利于缓存。

### 3.3 本地工具输入不等于 API 输入

运行时可能给 tool input 临时补字段，  
但发给 API 前必须再做一次 `normalizeToolInputForAPI()`，把不属于协议的字段剥掉。

## 4. 简单版本

如果只用最土的话解释 Day 9，可以这样说：

- `prompts.ts` 负责先把“标准说明书”拼出来。
- `systemPrompt.ts` 负责决定“这轮最终拿哪本说明书”。
- `attachments.ts` 负责把临时资料、记忆、技能列表这些补充材料另外装袋。
- `api.ts` 负责出门前最后检查一遍：说明书带没带对、材料塞没塞好、工具参数有没有带错。

## 5. 一句话收口

> Day 9 学的不是“写一段 system prompt”，而是“把模型输入做成一条可维护的流水线”。
