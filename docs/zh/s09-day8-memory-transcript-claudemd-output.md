# Day 8 输出物：Transcript / Session Memory / Auto Memory / CLAUDE.md 边界图

Day 8 最重要的不是把几个名词背下来，而是把“同一条信息到底该往哪层放”分清楚。

这四层虽然最后都可能影响上下文，但它们回答的问题并不一样：

- `transcript` 负责“刚刚到底发生了什么”
- `session memory` 负责“这次长会话接下来还要记住什么”
- `auto memory` 负责“下次开新会话还值得知道什么”
- `CLAUDE.md` 负责“长期默认应该怎么做”

## 1. 给青菜讲的简单解释

| 原名 | 青菜版叫法 | 一句话解释 |
|---|---|---|
| `transcript` | 聊天录像 / 施工录像 | 尽量保真地记录这次会话发生了什么，方便回放和 `/resume` |
| `session memory` | 当前会话便签 | 给这次长会话自己看的工作笔记，帮助后面继续做 |
| `auto memory` | 跨会话备忘卡 | 只留下下次新会话还值得知道的事实 |
| `CLAUDE.md` | 长期规则手册 | 告诉系统长期默认怎么做，不是在记这次发生了什么 |

如果只准用最土的话来区分：

- `transcript` 更像录像，不是摘要。
- `session memory` 更像桌上的便签，不是长期档案。
- `auto memory` 更像会后整理出来的长期卡片，不是所有笔记都往里塞。
- `CLAUDE.md` 更像墙上的规则，不是事实存档。

## 2. 四层边界图

```text
来了一条信息
  |
  +-- 它是在记录“刚刚到底发生了什么”吗？
  |      -> 放 transcript
  |      -> 关键词：可回放、可 resume、尽量保真
  |
  +-- 它是在帮助“这次长会话接下来怎么继续”吗？
  |      -> 放 session memory
  |      -> 关键词：当前会话、低频提炼、工作便签
  |
  +-- 它是“下次开新会话也还值得知道”的事实吗？
  |      -> 放 auto memory
  |      -> 关键词：跨会话、按主题归档、durable fact
  |
  +-- 它其实是在规定“长期默认怎么做”吗？
         -> 放 CLAUDE.md chain
         -> 关键词：规则、约束、说明、include/rules
```

再换一种从系统流向看的图：

```text
live conversation
  ->
recordTranscript()
  ->
session JSONL transcript
  ->
resume / load / replay

live conversation
  ->
post-sampling hook
  ->
session memory markdown
  ->
当前长会话继续使用

conversation after enough progress
  ->
extractMemories fork
  ->
memory/MEMORY.md + topic files
  ->
未来会话按需再次装配

CLAUDE.md / CLAUDE.local.md / .claude/rules/*.md
  ->
discovery + filtering + include expansion
  ->
instruction chain injected into context
```

## 3. 一个最容易记住的口诀

```text
刚刚发生过什么 -> transcript
这次会话后面还要接着用什么 -> session memory
下次开新会话还值得知道什么 -> auto memory
以后默认都该怎么做 -> CLAUDE.md
```

## 4. 哪些信息不该被放进 memory

先立一个总原则：

> memory 不是第二份 transcript，更不是“看起来有用就先存着”的垃圾桶。

### 4.1 三步先判断

```text
这条信息能重新读代码、读配置、查当前环境得到吗？
  能 -> 不进 memory

这条信息只对当前这次任务有用吗？
  是 -> 放 task / plan / session memory，不放 auto memory

这条信息其实是在规定长期规则吗？
  是 -> 放 CLAUDE.md，不放 auto memory
```

### 4.2 这些内容通常不要放进 durable memory

- 当前 repo 现成能重新读出来的结构信息。
  例如“函数在哪个文件”“项目里有 `src/` 和 `tests/`”
- 当前任务过程态。
  例如“这个 PR 还差两步”“我正在改认证模块”
- 一次性 tool output 原文。
  transcript 已经会保存可回放事实；必要时还有 content replacement 机制
- 本应进入 `CLAUDE.md` 的长期规则。
  例如代码规范、团队约束、审批规则
- 明显可能过时、且下次应重新观察验证的事实。
- 密钥、token、敏感凭据。

### 4.3 `session memory` 和 `auto memory` 各自也有禁区

| 层 | 不该塞进去的东西 | 为什么 |
|---|---|---|
| `session memory` | 完整原始对话、逐条 tool output、大段日志回放 | 这层是当前会话便签，不是 transcript 镜像 |
| `session memory` | 长期用户偏好、稳定项目规则 | 这些更适合 durable memory 或 `CLAUDE.md`，不该绑死在单次 session |
| `auto memory` | 当前 PR/branch 状态、临时 TODO、今天做到哪一步 | 这些会快速过期，只适合 task / plan / session memory |
| `auto memory` | repo 结构、函数位置、当前代码就能重新读出来的事实 | 下次重读代码比相信旧 memory 更可靠 |
| `auto memory` | “以后都必须这样做”的规则性描述 | 这是 `CLAUDE.md` 链的职责，不是 durable fact |

## 5. 和真实源码怎么对上

| 层 | 典型载体 | 关键源码 | 核心职责 |
|---|---|---|---|
| `transcript` | `session.jsonl` | `sessionStorage.ts` | 记录事实、维护 parent 链、支持恢复 |
| `session memory` | 当前 session 的 markdown 笔记 | `sessionMemory.ts` | 在长会话里低频提炼“当前便签” |
| `auto memory` | `memory/MEMORY.md` + topic 文件 | `extractMemories.ts` | 把跨会话仍有价值的事实提出来 |
| `CLAUDE.md` chain | `CLAUDE.md` / `CLAUDE.local.md` / `.claude/rules/*.md` | `claudemd.ts` | 发现、筛选、展开长期规则链 |

## 6. 一句话收口

> transcript 保存事实真相，session memory 维护当前会话笔记，auto memory 提炼跨会话事实，CLAUDE.md 维持长期规则；它们都服务上下文，但绝不是同一种上下文。
