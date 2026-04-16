# Day 6 输出物：Permission Gate 与 Sandbox 决策树

Day 6 最重要的不是“又多了一个安全模块”，而是把两层边界彻底分开：

- `Permission Gate` 回答“这次动作能不能做”
- `Sandbox` 回答“就算能做，最多能碰到哪里”

所以主线不是“模型提了 Bash 就直接执行”，而是：

```text
tool_use
  ->
permission gate
  -> deny | ask | allow
  ->
if allow and tool is Bash
  -> shouldUseSandbox
  -> sandbox runtime
  -> execute
```

## 1. “deny rules -> mode -> allow rules -> ask user -> sandbox” 决策树

这是最适合 Day 6 记忆的教学版主树：

```text
tool_use
  |
  +-- deny rules 命中？
  |     +-- 是 -> deny
  |     +-- 否
  |
  +-- mode check
  |     +-- plan 且是写操作 -> deny
  |     +-- auto 且是只读/安全操作 -> allow
  |     +-- 其他 -> 继续
  |
  +-- allow rules 命中？
  |     +-- 是 -> allow
  |     +-- 否
  |
  +-- ask user
        +-- 用户拒绝 -> deny
        +-- 用户允许
              |
              +-- 是 Bash？
                    +-- 否 -> execute
                    +-- 是
                          +-- shouldUseSandbox = true -> sandbox -> execute
                          +-- shouldUseSandbox = false -> unsandboxed execute
```

这棵树对应教学仓库的最小权限系统：先挡掉不能做的，再看当前模式，再放行稳定安全的，最后把灰区交给用户确认。

## 2. 真实源码里的更细顺序

产品源码比教学树更严，真正骨架更接近：

```text
tool_use
  ->
deny rule?
  -> yes: deny
  -> no
ask rule?
  -> yes 且不是 “Bash + 已启用 sandbox + autoAllowBashIfSandboxed + shouldUseSandbox”
     -> ask
  -> no / 可跳过
tool.checkPermissions()
  -> deny -> deny
  -> ask(rule / safetyCheck / requiresUserInteraction) -> ask
  -> passthrough / allow -> 继续
mode allow?
  -> bypassPermissions / 特殊 plan -> allow
always-allow rule?
  -> allow
剩余 passthrough
  -> ask
ask 被批准后
  -> 若工具是 Bash，再走 shouldUseSandbox -> sandbox runtime -> execute
```

这里最关键的两个理解：

- `mode` 不是最大优先级，`tool.checkPermissions()` 给出的 `deny`、内容级 `ask`、`safetyCheck` 可以压过 mode。
- `passthrough` 不是 allow，而是“前面还没放行”，最后默认收口成 `ask`。

## 3. Bash 为什么要有独立安全通道

- Bash 不是窄工具。`Read/Edit` 的能力边界很清楚，Bash 的能力边界几乎就是整个操作系统。
- Bash 输入不是普通参数，而是一种可执行动作描述；`&&`、`|`、`$()`、环境变量前缀、wrapper 命令都会改变真实执行意图。
- `Permission Gate` 只解决“能不能做”，不能解决“执行后最多碰到哪里”；这正是 `Sandbox` 的职责。
- 即使一条 Bash 被允许执行，也不代表应该裸奔执行；很多命令应该被 OS 级文件系统/网络限制包起来。
- `shouldUseSandbox()` 判断的不是“危险不危险”，而是“这条命令是否需要沙箱包装”；这是执行边界判断，不是语义安全判断。
- 沙箱层还负责补规则层看不全的真实攻击面，比如 settings 改写、`.claude/skills` 注入、bare git repo 植入、网络外联。
- 正因为 Bash 有这条独立通道，系统才敢提供 `autoAllowBashIfSandboxed`：不是少一道安全，而是把“硬隔离”拿来换取更少的人工审批。

## 4. 一句话结论

Day 6 可以压成一句话：

> `permissions.ts` 决定“能不能做”，`interactiveHandler.ts` 决定“要问时怎么收口”，`shouldUseSandbox.ts` 决定“Bash 要不要进沙箱”，`sandbox-adapter.ts` 决定“进沙箱后操作系统具体限制什么”。

## 5. 参考位置

- 教学主线：`docs/zh/s07-permission-system.md`，`docs/zh/day6-permission-gate-and-sandbox-guide.md`
- 教学实现：`agents/s07_permission_system.py`
- 真实源码：`src/utils/permissions/permissions.ts`，`src/hooks/toolPermission/handlers/interactiveHandler.ts`，`src/tools/BashTool/shouldUseSandbox.ts`，`src/utils/sandbox/sandbox-adapter.ts`
