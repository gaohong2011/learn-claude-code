# Day 17 输出物：自由文本消息 vs 协议消息

Day 17 最该压成一页纸的，不是 shutdown 和 plan approval 的业务细节，而是下面这条边界：

> **自由文本消息只说明“说了什么”；协议消息说明“哪一件协作请求现在走到哪一步”。**

## 1. 一页结论

如果把 Day 17 压成 5 句，最该记住的是这些：

- mailbox 是消息运输层，不等于协议层
- 普通消息只有 `from / to / content` 也可以成立
- 协议消息必须有 `type / request_id / payload or response`
- `RequestRecord` 记录 `pending / approved / rejected`，不是任务板
- 没有 `request_id`，多个请求同时存在时系统只能猜回复对应哪一个

所以 Day 17 真正新增的不是“更多消息类型”，而是：

> 团队第一次拥有了可追踪、可恢复、可明确批准或拒绝的协作流程。

## 2. 对比表

| 维度 | 自由文本消息 | 协议消息 |
| --- | --- | --- |
| 核心问题 | 谁跟谁说了什么 | 哪个请求被怎样处理 |
| 典型字段 | `from / to / content / timestamp` | `type / request_id / from / to / payload / approved` |
| 状态记录 | 通常没有 | `RequestRecord(status=pending/approved/rejected)` |
| 回复关联 | 靠人读语义 | 靠 `request_id` 精确关联 |
| 适合场景 | 提醒、讨论、补充上下文 | 计划审批、优雅关机、交接签收 |
| 失败模式 | 信息可能被看漏 | 可以查 pending、reject、expire |

最短判断法：

```text
只需要沟通 -> 自由文本消息
需要追踪处理结果 -> 协议消息
```

## 3. 协议链路图

```text
actor decides a request is needed
  ->
generate request_id
  ->
save RequestRecord(status=pending)
  ->
send ProtocolEnvelope through mailbox
  ->
recipient drains inbox
  ->
recipient returns response with same request_id
  ->
requester updates RequestRecord
  ->
status becomes approved / rejected / expired
```

把这条链拆开看：

| 阶段 | 它手里拿的是什么 | 它回答的问题 |
| --- | --- | --- |
| `ProtocolEnvelope` | `type / request_id / payload` | 这是一条什么协议消息 |
| `mailbox` | recipient inbox | 这条消息怎样送达 |
| `RequestRecord` | `kind / status / from / to` | 这件协作流程走到哪一步 |
| `response` | same `request_id` + decision | 对方明确批准还是拒绝 |
| `coordinator` | worker result / notification / SendMessage | 多 worker 协作怎样不乱 |

## 4. `request_id` 为什么是硬边界

`request_id` 至少解决四个问题：

1. **并发区分**：同一个 teammate 可以同时提交多个 plan
2. **精确更新**：response 能找到唯一 pending record
3. **恢复检查**：重启或 compact 后还能查出哪些请求没处理
4. **审计收尾**：拒绝、超时、批准都有明确对象

没有 `request_id` 时，系统只能处理这样的文本：

```text
“同意。”
```

有 `request_id` 时，系统能处理这样的状态更新：

```text
request_id=req_123
status=pending -> approved
reviewed_by=lead
feedback="Proceed with tests first."
```

这就是 Day 17 最该守住的边界：

> **没有 request_id，就没有可追踪协议。**

## 5. shutdown 和 plan approval 的共同模板

虽然业务不同，但底层形状一样。

```text
shutdown:
  lead -> shutdown_request(request_id) -> teammate
  teammate -> shutdown_response(request_id, approve/reject) -> lead

plan approval:
  teammate -> plan_approval(request_id, plan) -> lead
  lead -> plan_approval_response(request_id, approve/reject) -> teammate
```

共同模板是：

```text
create request
  ->
send structured envelope
  ->
receive structured response
  ->
update status by request_id
```

所以这一天学的不是“关机工具”，而是团队协议的最小骨架。

## 6. 一句记忆钩子

> **Day 16 让队友能互相发消息；Day 17 让某些消息变成有编号、有状态、有回执的协作请求。**
