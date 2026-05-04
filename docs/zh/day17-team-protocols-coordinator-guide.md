# Day 17 学习指南：Team Protocols 与 Coordinator 纪律

> Day 16 学的是“长期队友怎样通过 mailbox 被叫醒”。
> Day 17 学的是“mailbox 里的某些消息怎样升级成可追踪的协作协议”。
>
> 这一天最重要的不是多加几个消息类型，而是先把下面这句话讲顺：
>
> **自由文本消息只负责沟通；协议消息必须带 `request_id`，并有一条可更新的请求状态记录。**

## 0. 这一天到底在学什么

Day 16 以后，leader 和 teammate 已经能互相发消息。

但只靠自由文本，很快会遇到两个问题：

- 多个请求同时存在时，回复到底对应哪一个请求
- 某些动作必须明确批准或拒绝，不能靠“我知道了”这种模糊文本

Day 17 要回答的是：

> **团队协作什么时候不能再只靠聊天，而必须进入结构化协议？**

所以这一天真正要学的是这条链：

```text
actor 发起协议请求
  ->
生成 request_id
  ->
写入 RequestRecord(status=pending)
  ->
把 ProtocolEnvelope 投递到对方 inbox
  ->
对方下一轮读取协议消息
  ->
明确返回 approved / rejected
  ->
请求状态按 request_id 更新
  ->
请求方根据结果继续、停止或修改计划
```

如果只记一句话，先记这个：

> **`request_id` 是团队协作从“聊天”进入“协议”的硬边界。**

## 1. 这次重点看哪些代码

### 必读

- `docs/zh/s16-team-protocols.md`
- `docs/zh/team-task-lane-model.md`
- `agents/s16_team_protocols.py`
- `/Users/hong.gao/python/src/claude-code-codex/src/coordinator/coordinatorMode.ts:36-111,111-194`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/SendMessageTool/SendMessageTool.ts:268-518,741-917`
- `/Users/hong.gao/python/src/claude-code-codex/src/tools/TaskStopTool/TaskStopTool.ts:39-131`
- `/Users/hong.gao/python/src/claude-code-codex/src/utils/mailbox.ts:19-73`

在教学版 `agents/s16_team_protocols.py` 里，优先盯这几个入口：

- `RequestStore`
- `handle_shutdown_request(...)`
- `shutdown_response` teammate tool
- `plan_approval` teammate tool
- `handle_plan_review(...)`
- `TOOL_HANDLERS`
- `agent_loop(...)`

原因很简单：

- `RequestStore` 说明 request 不应该只存在于一条 inbox message 里
- shutdown 和 plan approval 说明不同业务可以复用同一个 request / response 模板
- `TOOL_HANDLERS` 说明 lead 侧的协议动作也是 tool control plane 的一部分
- `agent_loop(...)` 说明协议消息最终仍然通过 inbox 回到某个 actor 的下一轮输入

## 2. 先建立最小心智模型

先把 Day 17 想成下面这 4 个盒子：

```text
1. MessageEnvelope
2. ProtocolEnvelope
3. RequestRecord
4. Coordinator discipline
```

它们的分工分别是：

- `MessageEnvelope`：谁给谁发了什么
- `ProtocolEnvelope`：这条消息是不是结构化请求或响应
- `RequestRecord`：这件协作流程现在走到哪一步
- `Coordinator discipline`：谁能发起、谁能批准、谁能停止 worker

这里最值得先背下来的结论是：

> **mailbox 是运输层；protocol 是工作流层。**

也就是说，协议消息仍然可以走 mailbox。
但一旦消息带上 `request_id`，它就不再只是普通聊天，而是进入了一个可追踪流程。

## 3. Day 17 的主线到底怎么跑

### 3.1 普通消息只解决“说了什么”

Day 16 的普通消息大概长这样：

```python
{
    "type": "message",
    "from": "lead",
    "content": "Please check the login test.",
    "timestamp": 1710000000.0,
}
```

它适合：

- 提醒
- 补充上下文
- 同步进展
- 给 teammate 新输入

但它不适合表达：

- 请批准这个计划
- 请同意优雅关机
- 这条请求已经被拒绝
- 这条回复对应前面第几个请求

最短判断法：

> **如果后续必须明确对号入座，就不能只用普通消息。**

### 3.2 协议消息必须带 `request_id`

教学版的 `shutdown_request` 会先创建 request record：

```text
request_id=req_123
kind=shutdown
from=lead
to=alice
status=pending
```

然后再通过 mailbox 发出协议 envelope：

```python
{
    "type": "shutdown_request",
    "from": "lead",
    "request_id": "req_123",
    "content": "Please shut down gracefully.",
}
```

这一步的关键不是“多了一个字段”，而是：

> **后续所有 response 都必须引用同一个 `request_id`，系统才能知道要更新哪条记录。**

没有 `request_id`，协议就会退化成自然语言对话。

### 3.3 RequestRecord 记录“流程走到哪一步”

教学版把 request 落到：

```text
.team/requests/{request_id}.json
```

最小状态机是：

```text
pending -> approved
pending -> rejected
pending -> expired
```

这层回答的问题不是“任务做完了吗”，而是：

> **这次协作请求是否已经得到明确处理？**

所以不要把它和 `TaskRecord` 混起来。

- `RequestRecord` 管协作流程
- `TaskRecord` 管工作目标

### 3.4 shutdown 和 plan approval 复用同一个模板

Day 17 教学版只讲两类协议就够了。

第一类是 shutdown：

```text
lead
  ->
shutdown_request(request_id)
  ->
teammate
  ->
shutdown_response(request_id, approve/reject)
  ->
RequestRecord 更新为 approved/rejected
```

第二类是 plan approval：

```text
teammate
  ->
plan_approval(request_id, plan)
  ->
lead
  ->
plan_approval_response(request_id, approve/reject, feedback)
  ->
teammate 根据结果继续或改计划
```

业务不同，但骨架一样：

```text
create request
  ->
send structured envelope
  ->
receive structured response
  ->
update status by request_id
```

这就是 Day 17 最该学稳的抽象。

### 3.5 Coordinator 纪律把“谁该做什么”收紧

参考实现里的 coordinator mode 不是“更聪明的聊天模式”，而是一套编排纪律。

`getCoordinatorSystemPrompt()` 明确规定：

- coordinator 负责分派、综合、和用户沟通
- worker 负责研究、实现、验证
- worker 结果以 `<task-notification>` 形式回到 coordinator
- 继续已有 worker 时用 `SendMessage`
- 停止 worker 时用 `TaskStop`
- coordinator 不应该替 worker 编造结果

这和 protocol 的关系是：

> **协议保证单次请求可追踪；coordinator 纪律保证多 worker 编排不乱。**

Day 17 不是要把 coordinator 的所有策略背下来，而是要理解：

- 有些消息是 user-facing
- 有些消息是 internal task notification
- 有些消息是 structured protocol response
- 这三类不能混成一种自由文本

## 4. 自由文本消息 vs 协议消息

这是 Day 17 的产出项核心。

| 维度 | 自由文本消息 | 协议消息 |
| --- | --- | --- |
| 目的 | 沟通、提醒、补上下文 | 追踪一件需要明确处理的协作流程 |
| 必要字段 | `from / to / content` | `type / request_id / from / to / payload or response` |
| 状态 | 通常没有独立状态 | 至少有 `pending / approved / rejected` |
| 回复关系 | 靠人读语义判断 | 靠 `request_id` 关联 |
| 适合场景 | “再跑一下测试” | “批准这个计划”“同意关机” |
| 系统能否恢复 | 弱，主要靠 transcript | 强，可以从 RequestRecord 恢复 |

一句话区分：

```text
自由文本消息解决“说了什么”
协议消息解决“这件事是否被明确处理”
```

## 5. `request_id` 为什么是硬边界

`request_id` 至少解决 4 个问题：

1. **并发区分**：alice 同时提交两个计划，leader 的批准必须知道对应哪一个
2. **状态更新**：response 不能只写“同意”，必须能更新某条 pending record
3. **恢复检查**：系统重启或对话变长后，还能查出哪些 request 没处理
4. **审计与收尾**：失败、拒绝、超时都有明确对象

没有 `request_id` 时，系统只能猜。

有了 `request_id` 后，系统可以写出确定逻辑：

```text
response.request_id
  ->
find RequestRecord
  ->
validate current status is pending
  ->
set approved / rejected
  ->
notify requester
```

所以 `request_id` 不是好看的元数据。
它是协议消息和普通消息的分界线。

## 6. 参考实现里最值得看的产品化细节

### 6.1 `SendMessageTool.ts`

这部分最值得看的是结构化分支：

```text
message.type == shutdown_request
message.type == shutdown_response
message.type == plan_approval_response
```

参考实现做了几件很现实的事：

- structured message 不能 broadcast
- shutdown response 必须发给 `team-lead`
- 拒绝 shutdown 时必须给 reason
- plan approval 只能由 team lead 批准或拒绝
- cross-session message 只允许 plain text，不能发 structured message

这说明协议消息比普通消息有更强的合法性要求。

### 6.2 `TaskStopTool.ts`

`TaskStopTool` 是另一条停止执行的路径。

它关注的是：

- task id 是否存在
- task 是否正在 running
- 调用 `stopTask(...)` 停止当前执行槽位

把它和 shutdown protocol 分开：

- `TaskStop` 是外部控制面直接停止 runtime task
- `shutdown_request` 是团队协议里的优雅关机请求

两者都可能导致 worker 停止，但语义不同。

### 6.3 `coordinatorMode.ts`

这里最值得看的，是 coordinator prompt 对 worker 编排的约束：

- 不要让 worker 去检查另一个 worker
- 不要把 trivial file read / command run 交给 worker
- worker 结果通过 `<task-notification>` 回来
- 继续已有 worker 时用 `SendMessage`
- 新开 worker 前要考虑是否可以复用已有上下文

这说明 Day 17 的“纪律”不只是协议字段，也是协作角色的边界。

### 6.4 `utils/mailbox.ts`

这个 in-memory mailbox 很小，但模型很清晰：

- `send(...)` 入队或唤醒 waiter
- `poll(...)` 非阻塞取一条
- `receive(...)` 等待一条匹配消息
- `revision` 用来触发 UI / hook 更新

它说明 mailbox 是通道，不是协议本身。
协议要靠 envelope 和 request record 才成立。

## 7. 初学者最容易犯的错

### 7.1 只有消息，没有请求记录

如果发完 `shutdown_request` 后没有保存 request 状态，后面就无法稳定查询、恢复、拒绝或超时。

### 7.2 只有自然语言回复

“好的”不是协议响应。

协议响应至少要有：

```text
type
request_id
approved / rejected
reason or feedback
```

### 7.3 把 `request_id` 和 `task_id` 混起来

`request_id` 管一次协作流程。
`task_id` 管一项工作目标。

一次 task 里可以有多个 request。
一次 request 也不一定对应一个 durable task。

### 7.4 把 coordinator 当成超级 worker

coordinator 的职责是编排，不是替所有 worker 直接做执行细节。

如果 coordinator 一边分派 worker，一边自己随意改文件，还把 worker notification 当普通用户消息处理，系统会很快乱掉。

## 8. 今天的输出物该怎么写

学习计划给 Day 17 的产出要求是：

> 写一页“自由文本消息 vs 协议消息”的对比说明，必须明确 `request_id` 为什么是团队协作的硬边界。

建议输出物压成三部分：

### 第一部分：一张对比表

```text
free text message:
  from / to / content
  no independent workflow status

protocol message:
  type / request_id / payload / response
  backed by RequestRecord
```

### 第二部分：一条协议链

```text
create request_id
  ->
save RequestRecord(status=pending)
  ->
send ProtocolEnvelope
  ->
receive response with same request_id
  ->
update RequestRecord(status=approved/rejected)
```

### 第三部分：一句硬边界

```text
没有 request_id，就没有可追踪协议。
```

## 9. 读完 Day 17 应该能自己说清的话

如果这一天真的吃透了，你应该能不用看文档，自己说清下面这几句：

- mailbox 是运输层，protocol 是工作流层
- 普通消息解决“说了什么”，协议消息解决“这件事走到哪一步”
- `request_id` 让 response 能准确关联 request
- RequestRecord 不是 TaskRecord
- shutdown 和 plan approval 可以复用同一套请求-响应模板
- coordinator 纪律的重点是编排边界，而不是多开 worker 本身

## 10. 最低验证

```sh
cd learn-claude-code
python agents/s16_team_protocols.py
```

这个脚本会启动交互式 agent loop，并依赖 `MODEL_ID` 和可用的 Anthropic API 配置。
如果当前环境不适合真实调用模型，可以先做语法级验证：

```sh
python -m py_compile agents/s16_team_protocols.py
```

真实运行时重点观察三件事：

1. `shutdown_request` 是否创建 `.team/requests/{request_id}.json`
2. teammate 的 `shutdown_response` 是否用同一个 `request_id` 更新状态
3. `plan_approval` 和 `plan_approval_response` 是否复用同一套 pending -> approved/rejected 状态机

最后把这句话记住就够了：

> **Day 17 的关键词是“request_id + RequestRecord”。**
