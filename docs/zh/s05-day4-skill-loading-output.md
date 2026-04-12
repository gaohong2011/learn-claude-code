# Day 4 输出物：`Skill` 发现、按需加载与输入注入

`s05` 真正讲的不是“多一个工具”，而是把技能知识拆成两层：先让模型知道“有哪些可用技能”，再在需要时把某一份完整正文注入当前工作上下文。这样既不把 system prompt 塞爆，也能在具体任务里拿到足够细的流程说明。
## 0. 一些理解
- Skill本质还是context management，先只对LLM暴露基本信息，减少context，再按需要加载需要执行的完整skill内容。
## 1. 4 步图

```text
skills/*/SKILL.md
  |
  v
[1] 发现技能
    - 扫描 skills 目录 / 动态发现 .claude/skills
    - 找到有哪些 SKILL.md 可用
  |
  v
[2] 生成描述
    - 只解析轻量元信息：name / description / when_to_use
    - 形成 skill catalog
    - 注入 system prompt / session guidance，让模型先知道“能用什么”
  |
  v
模型判断当前任务需要某份专门知识
  |
  v
[3] 按需展开正文
    - 调用 load_skill / Skill
    - 读取完整 SKILL.md
    - 补上 base directory、变量替换等执行期信息
  |
  v
[4] 注入输入管道
    - 教学版：作为 tool_result 追加到 messages[]
    - 参考实现：作为 meta user message / attachment 注入当前轮输入
    - 下一轮模型在当前任务上下文里消费完整 skill 正文
```

## 2. 代码锚点

- 教学版的“发现技能”在 [`agents/s05_skill_loading.py`](../../agents/s05_skill_loading.py) 的 `SkillRegistry._load_all()`；它扫描 `skills/**/SKILL.md`，解析 frontmatter，构造 `SkillManifest` 和 `SkillDocument`。
- 教学版的“生成描述”在同文件的 `describe_available()`，并通过 `SYSTEM` 把 `name + description` 作为轻量目录注入 system prompt。
- 教学版的“按需展开正文”在 `load_full_text()` 和 `TOOL_HANDLERS["load_skill"]`；“注入输入管道”则发生在 `agent_loop()` 把 `tool_result` 追加回 `messages[]` 的地方。
- 参考实现的“发现技能”在 `loadSkillsFromSkillsDir()`、`discoverSkillDirsForPaths()`、`addSkillDirectories()`：先加载静态技能，再按文件路径动态发现更近的 `.claude/skills`。
- 参考实现的“生成描述”在 `parseSkillFrontmatterFields()`、`getSkillToolCommands()`、`getSessionSpecificGuidanceSection()`、`getSystemPrompt()`：只把可执行的技能描述列进 prompt，而不是直接展开全文。
- 参考实现的“按需展开正文 + 注入输入管道”在 `src/tools/SkillTool/SkillTool.ts`：先生成 `finalContent`，再 `addInvokedSkill(...)` 登记到 compact 保留状态，最后用 `createUserMessage({ content: finalContent, isMeta: true })` 把正文塞回当前输入消息流。

## 3. 为什么 `s05` 必须早于 `s06`

`s05` 是上游的“少放进来”，`s06` 是下游的“放进来以后怎么控体积”。顺序不能反，因为正确的上下文治理不是先把 prompt 塞满，再靠 compact 擦屁股，而是先把长期知识拆成轻量目录，只把 `name + description` 暴露给模型，等模型确认需要时才按需展开正文。这样到了 `s06`，compact 管理的才是“已经值得进入上下文的内容”，而不是一开始就错误注入的大块说明。更关键的是，参考实现里 skill 正文注入时会调用 `addInvokedSkill()`，而 compact 又会按 `POST_COMPACT_SKILLS_TOKEN_BUDGET` 重新保留或截断这些已调用技能。这说明 `s06` 的压缩逻辑本身就建立在 `s05` 的 skill 注入模型之上：先有按需加载，后有压缩保留，顺序反过来心智和实现都会变得别扭。
