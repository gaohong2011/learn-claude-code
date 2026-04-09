# Day 1 输出物：最小 Query Loop、Control Plane 与 `tool_use_id`

## 1. 最小闭环图

```text
用户输入 query
  ↓
query state / LoopState
(messages, turn_count, transition_reason)
  ↓
client.messages.create(...)
  ↓
模型响应 response
  ├─ stop_reason != "tool_use"
  │    ↓
  │  结束，输出最终文本
  │
  └─ stop_reason == "tool_use"
       ↓
     execute_tool_calls(response.content)
       ↓
     生成 tool_result
     {
       "type": "tool_result",
       "tool_use_id": block.id,
       "content": output
     }
       ↓
     追加回 state.messages 作为下一轮 user content
       ↓
     run_one_turn(state) 再来一轮
```

也可以压成一句调用链：

```text
用户输入 -> query state -> 模型响应 -> tool_use -> 本地执行 -> tool_result -> 写回 state -> 下一轮模型响应
```

## 2. 单 Agent 为什么必须长出 Query Control Plane

1. 只有 control plane，模型的一次 `tool_use` 才能和下一次 `tool_result` 被稳定串成同一个闭环，而不是一次性吐完就丢状态。
2. 它负责保存 `messages / turn_count / transition_reason` 这类查询级状态，否则 agent 无法知道“现在处于第几轮、为什么继续、该把什么喂回模型”。
3. 它把“模型推理”和“真实副作用”隔开了，因为执行 bash、权限控制、超时处理、错误回传都不该塞进 prompt 本身。
4. 它提供可观测性和可调试性，因为请求包、响应包、工具结果、终止原因都需要在 query 维度被记录和重放。
5. 它是后续能力的挂载点，因为 memory、retry、budget、checkpoint、permission、subagent 本质上都不是模型本身，而是 query orchestration。

## 3. `tool_use_id` 是怎么来的

先看 `s01` 的最小工具闭环：

- 请求里只传 `model / system / messages / tools / max_tokens`
- 你的代码没有手工生成任何工具调用 ID
- 响应里的 `tool_use` block 已经自带 `id`
- 本地执行工具后，再把这个 `id` 原样回填到 `tool_result.tool_use_id`

对应 [agents/s01_agent_loop.py](../../agents/s01_agent_loop.py) 的关键逻辑：

```python
response = client.messages.create(**request_payload)
```

```python
results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": output,
})
```

所以职责分工是：

- 上游模型服务返回 `tool_use.id`
- 本地 agent 执行工具
- 本地 agent 用同一个值回传 `tool_result.tool_use_id`

也就是说：

```text
tool_use.id -> tool_result.tool_use_id
```

这是一对请求-结果关联 ID，用来告诉模型“这个工具结果属于上一轮哪一次工具调用”。

从客户端工程视角，最稳妥的表述是：

- `tool_use.id` 是 API 响应里给出的协议字段
- `tool_result.tool_use_id` 只是对这个字段的引用
- 本地代码不负责生成它，只负责原样带回去

## 4. LLM 为什么“知道”该怎么填这个 ID

准确说法不是“你本地代码教会了 LLM 填 ID”，而是你把 `tools` 定义传给上游模型服务后，响应会进入工具调用协议模式。这个协议要求当模型选择调用工具时，返回结构化的 `tool_use` block，其中包含 `type / name / input / id`。

Anthropic SDK 的生成文档示例里，`tool_use` 和 `tool_result` 就是这样配对的：

```json
{
  "type": "tool_use",
  "id": "toolu_01D7FLrfh4GYq7yT1ULFeyMV",
  "name": "get_stock_price",
  "input": { "ticker": "^GSPC" }
}
```

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01D7FLrfh4GYq7yT1ULFeyMV",
  "content": "259.75 USD"
}
```

Anthropic 官方文档明确确认的是：

- `tool_use` block 里会有唯一 `id`
- 这个 `id` 用来匹配后续工具结果
- `tool_result.tool_use_id` 应该填写前一轮 `tool_use.id`

Anthropic 官方文档没有继续拆开说明这个 `id` 在服务端内部究竟是“模型直接生成”还是“API/网关层补充生成”。

因此最稳妥的工程结论是：

- `tool_use.id` 由上游 API 作为响应字段提供
- 本地代码不应该自己猜、更不应该自己生成一个新的值
- 本地代码只需要把它原样带回去

## 5. 当前环境里的一个实际细节

当前 `s01` 通过 `ANTHROPIC_BASE_URL` 走的是兼容 Anthropic 协议的上游接口，因此你看到的工具调用 ID 可能是 `bash_2` 这种格式，而不一定是 Anthropic 官方文档里常见的 `toolu_...`。这不影响协议语义，关键点只有一个：

`tool_result.tool_use_id` 必须等于上一轮 `tool_use.id`。

结合当前兼容接口返回 `bash_2` 这类 provider 风格的格式，一个合理推断是：这个 ID 很可能由兼容 API 的服务层产出，而不是由你本地 agent 生成。

这句是推断，不是官方明文。

## 6. 参考资料

- Anthropic 官方工具调用文档：<https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use>
- Moonshot 官方博客，说明其接口完全兼容 Anthropic API 并保证 toolcall 格式：<https://platform.moonshot.cn/blog/posts/kimi-k2-0905>
