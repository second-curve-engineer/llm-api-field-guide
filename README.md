# Agent 工程师的 LLM API 对照手册

> LLM API Field Guide for Agent Engineers

[English](README.en.md) | 中文

> 想看排版更友好的版本，可以直接打开 HTML 页面：  
> https://second-curve-engineer.github.io/llm-api-field-guide/

面向 Agent 工程师的 LLM API 对照手册，从 Agent Runtime 实现视角比较 OpenAI Chat Completions 与 Anthropic Messages API。

这不是官方 API 文档的复制版，而是把 API 差异重新组织成 Agent 开发中真正会遇到的工程问题：

- Agent Loop 如何判断继续推理、调用工具、重试或结束？
- OpenAI 与 Anthropic 的 tool schema、tool result、structured output 有哪些关键差异？
- Streaming / SSE 事件应该如何解析和拼接？
- 哪些响应字段应该进入 trace、成本统计和 eval 系统？

## 为什么整理这份手册

在实际构建 Agent Runtime 时，LLM API 的差异不会只停留在“参数名不同”这一层。它会影响：

- Agent Loop 的状态判断
- Tool Calling 的参数解析与结果回填
- Streaming 输出的组装方式
- Trace / Eval / 成本统计的数据结构
- 多 provider adapter 的抽象边界

这份手册的目标是把这些差异整理成一份可查、可复用、面向工程实现的参考材料。

> 说明：这是 `second-curve-engineer` 的独立工程笔记，不隶属于 OpenAI 或 Anthropic，也不代表官方文档。生产环境使用前，请始终以官方文档为准。

## 目录

- [关键差异速查](#diff)
- [枚举值汇总](#enums)
- [OpenAI Responses API 选择](#responses-api)
- [Provider Adapter 设计](#provider-adapter)
- [Agent Loop 状态机](#loop-state-machine)
- [Agent Loop 核心要点](#agent-loop)
- [错误处理与重试策略](#error-retry)
- [Trace、成本与上下文管理](#trace-cost-context)
- [常见踩坑](#gotchas)
- [OpenAI Responses API 参数](#openai-responses)
- [OpenAI Chat Completions 请求参数](#openai-req)
- [Anthropic Messages API 请求参数](#anthropic-req)
- [OpenAI Chat Completions 返回参数](#openai-resp)
- [Anthropic Messages API 返回参数](#anthropic-resp)
- [Tool Schema 用法](#tool-schema)
- [Structured Output](#structured-output)
- [Tool Schema vs Structured Output 如何选择](#schema-vs-structured)
- [SSE 协议格式](#sse-protocol)
- [OpenAI SSE 解析](#sse-openai)
- [Anthropic SSE 解析](#sse-anthropic)
- [工具调用 Streaming](#sse-tool-stream)
- [完整 HTTP 示例：System Prompt](#ex-system)
- [完整 HTTP 示例：Tools Schema](#ex-tools)
- [完整 HTTP 示例：工具结果回填](#ex-toolresult)
- [完整 HTTP 示例：Response 解析](#ex-response)

## 图例

- `必填`：必填字段，缺少会报错
- `重点`：Agent 场景重点关注
- `枚举值`：有固定取值的字段

<a id="diff"></a>
## 关键差异速查

`两家规范最容易混淆的地方`

| 对比项 | OpenAI Chat Completions | Anthropic Messages |
| --- | --- | --- |
| System prompt 位置 | messages 数组里，role="system" 或 "developer" | 顶层 system 字段， 不在 messages 里 |
| messages role 种类 | 5种：system / developer / user / assistant / tool | 2种：user / assistant |
| max_tokens 是否必填 | 可选 | 必填 |
| 工具 schema 字段名 | function. parameters | input_schema |
| tool_choice 格式 | 字符串 "auto"/"none"/"required" 或对象 | 始终是对象 {type:"auto"/"any"/"tool"/"none"} |
| 工具结果 role | role=" tool "，带 tool_call_id | role=" user "，content block type=tool_result，带 tool_use_id |
| tool call 参数类型 | function.arguments 是 字符串 ，需 JSON.parse | tool_use.input 已是 JSON 对象 ，直接用 |
| 文字 + 工具并存 | 有 tool_calls 时 content= null | content 数组里 text block 和 tool_use block 可并存 |
| Loop 判断字段位置 | choices[0]. finish_reason | 顶层 stop_reason |
| 正常结束枚举值 | " stop " | " end_turn " |
| 调工具枚举值 | " tool_calls " | " tool_use " |
| 截断枚举值 | " length " | " max_tokens " |
| temperature 上限 | 2.0 | 1.0 |
| 思考模式参数 | reasoning.effort (low/medium/high)，Responses API | thinking.type (enabled/disabled/ adaptive ) |
| token 统计字段名 | prompt_tokens / completion_tokens | input_tokens / output_tokens |
| 鉴权请求头 | Authorization: Bearer $OPENAI_API_KEY | x-api-key: $ANTHROPIC_API_KEY + anthropic-version（必填） |

<a id="enums"></a>
## 枚举值汇总

`所有有固定取值的字段`

### finish_reason / stop_reason —— Agent Loop 判断

| 含义 | OpenAI finish_reason | Anthropic stop_reason |
| --- | --- | --- |
| 正常结束，读文本 | stop | end_turn |
| 需要执行工具 | tool_calls | tool_use |
| 输出被截断 | length | max_tokens |
| 安全拒绝 | content_filter | refusal |
| 命中停止符 | stop （同正常结束） | stop_sequence |
| 长任务中断可恢复 | — | pause_turn |

### tool_choice —— 工具调用策略

| 策略 | OpenAI | Anthropic |
| --- | --- | --- |
| 模型自决 | "auto" （字符串） | {type:"auto"} （对象） |
| 禁止工具 | "none" | {type:"none"} |
| 必须调工具（模型自选） | "required" | {type:"any"} |
| 强制指定工具 | {type:"function",function:{name:"..."}} | {type:"tool",name:"..."} |

### Streaming 事件类型

| 含义 | OpenAI SSE | Anthropic SSE |
| --- | --- | --- |
| 消息开始 | （无独立事件） | message_start |
| 文本增量 | choices[0].delta.content | content_block_delta (text_delta) |
| 工具参数增量（需拼接） | delta.tool_calls[].function.arguments | content_block_delta (input_json_delta) |
| stop_reason 出现 | finish_reason 非 null | message_delta |
| 流结束 | [DONE] | message_stop |

### messages role 枚举

| 用途 | OpenAI role | Anthropic role |
| --- | --- | --- |
| 系统指令 | system / developer | 顶层 system 字段（不在 messages） |
| 用户输入 | user | user |
| 模型输出 | assistant | assistant |
| 工具结果 | tool （第 5 种 role） | user （content block type=tool_result） |

<a id="responses-api"></a>
## OpenAI Responses API 选择

`新项目、旧项目与跨 Provider 抽象`

> 结论： 本手册保留 Chat Completions，是因为它仍然是大量存量项目、SDK 示例和兼容服务的事实接口；新项目如果只接 OpenAI，优先评估 Responses API；如果要做 OpenAI / Anthropic 双 Provider，建议在 Adapter 层统一抽象，不把业务逻辑绑死在任一原始 API 上。

| 选择项 | 适合场景 | Agent 工程关注点 | 建议 |
| --- | --- | --- | --- |
| Chat Completions | 存量代码、OpenAI 兼容服务、简单 tool calling | messages / tools / tool_calls 模型清晰，但和新能力的统一入口割裂 | 适合作为兼容层和对照基线 |
| Responses API | 新 OpenAI 项目、多模态、内建工具、长期演进 | 更像统一响应对象，适合承载新能力；但和 Anthropic Messages 仍需 Adapter | 只接 OpenAI 时优先评估 |
| Anthropic Messages | Claude 工具调用、thinking、长上下文与 prompt cache | content block 模型和 OpenAI 差异明显，工具结果回填格式完全不同 | 不要在业务层直接拼 Anthropic 原始结构 |
| Provider Adapter | 需要多模型、多厂商或可替换 LLM 能力 | 把 stop reason、tool call、usage、stream event 收敛成内部协议 | Agent Runtime 推荐方案 |

<a id="provider-adapter"></a>
## Provider Adapter 设计

`把 API 差异收敛成 Agent Runtime 内部协议`

> 核心判断： 业务代码不应该直接依赖 OpenAI 的 choices[0].finish_reason，也不应该直接依赖 Anthropic 的 content block。Agent Runtime 应该只面对统一的 AgentLlmResponse。

**TypeScript Interface 内部协议**

```ts
type AgentStopReason =
  | "final"
  | "tool_call"
  | "truncated"
  | "refusal"
  | "pause"
  | "unknown"

interface AgentToolCall {
  id: string
  name: string
  input: Record<string, unknown>
}

interface AgentLlmResponse {
  provider: "openai" | "anthropic"
  model: string
  requestId?: string
  stopReason: AgentStopReason
  text: string
  toolCalls: AgentToolCall[]
  usage?: {
    inputTokens?: number
    outputTokens?: number
    reasoningTokens?: number
    cacheReadTokens?: number
  }
  raw: unknown
}

interface LlmAdapter {
  complete(request: AgentLlmRequest): Promise<AgentLlmResponse>
  stream(request: AgentLlmRequest): AsyncIterable<AgentStreamEvent>
}
```

| 差异点 | OpenAI | Anthropic | Adapter 内部处理 |
| --- | --- | --- | --- |
| 停止原因 | choices[0].finish_reason | 顶层 stop_reason | 映射为 AgentStopReason |
| 工具参数 | function.arguments 是字符串 | tool_use.input 是对象 | 统一输出 AgentToolCall.input 对象 |
| 工具结果回填 | role="tool" + tool_call_id | role="user" + tool_result block | 由 formatToolResult(provider, result) 隔离 |
| 文本位置 | message.content | content[].text block | 统一拼成 response.text |
| usage 字段 | prompt_tokens / completion_tokens | input_tokens / output_tokens | 统一成 inputTokens / outputTokens |
| Streaming | data-only SSE + [DONE] | event typed SSE | 统一成 text_delta / tool_input_delta / done |

<a id="loop-state-machine"></a>
## Agent Loop 状态机

`从字段判断升级为 Runtime 行为`

| 状态 | OpenAI 信号 | Anthropic 信号 | Agent Runtime 行为 |
| --- | --- | --- | --- |
| 正常结束 | finish_reason=stop | stop_reason=end_turn | 提取文本，生成最终回答，写入 trace |
| 需要工具 | tool_calls | tool_use | 校验工具白名单与 schema，执行工具，回填结果，再次请求 LLM |
| 输出截断 | length | max_tokens | 记录不完整原因；可扩大 token 重试、请求续写或返回部分结果 |
| 安全拒绝 | content_filter / refusal | refusal | 进入 fallback，给用户可解释提示，记录安全事件 |
| 长任务暂停 | 无直接等价 | pause_turn | 持久化 PendingRun，等待 HITL 或外部事件后精准 resume |

<a id="agent-loop"></a>
## Agent Loop 核心要点

> 一句话： LLM API 只解决"一次无状态推理"，Agent Runtime 负责上下文、工具执行、状态、安全、可观测性。

**TypeScript 伪代码 通用**

```ts
messages = [
  { role: "system", content: systemPrompt },
  { role: "user",   content: userInput },
]

for (turn of range(maxTurns)) {
  traceRequest(messages, tools)

  response = await callLLM({ model, messages, tools })

  traceResponse(response)

  // OpenAI: response.choices[0].finish_reason
  // Anthropic: response.stop_reason
  if (isFinalText(response)) {
    return extractText(response)
  }

  toolCalls = extractToolCalls(response)

  // 把 assistant 那条存入历史（含 tool_calls / content blocks）
  messages.push(toAssistantMessage(response))

  for (call of toolCalls) {
    if (!allowedTools.has(call.name)) {
      toolResult = { error: "tool_not_allowed" }
    } else if (guardrailRejects(call)) {
      toolResult = { error: "guardrail_rejected" }
    } else if (needsApproval(call)) {
      savePendingState(messages, call)
      return { status: "pending_approval" }
    } else {
      toolResult = await runTool(call)
    }

    // OpenAI: role="tool", tool_call_id
    // Anthropic: role="user", type="tool_result", tool_use_id
    messages.push(formatToolResult(call, toolResult))
  }
}

return fallback("max_turns_exceeded")
```

### finish_reason / stop_reason 决策树

| 值（OpenAI / Anthropic） | 含义 | Agent 动作 |
| --- | --- | --- |
| stop / end_turn | 正常结束 | 提取文本，返回给用户 |
| tool_calls / tool_use | 需要工具 | 执行工具 → 回填 → 再次请求 LLM |
| length / max_tokens | 输出截断 | 可续写，或提示用户结果不完整 |
| content_filter / refusal | 安全拒绝 | 给用户友好提示，记录 Trace |
| stop_sequence | 命中停止符 | 按业务逻辑处理 |
| pause_turn （Anthropic only） | 长任务中断 | 保存状态，HITL 审批后恢复 |

<a id="error-retry"></a>
## 错误处理与重试策略

`把 API 错误变成可控的 Runtime 分支`

| 错误类型 | 常见原因 | Agent 行为 | 是否重试 |
| --- | --- | --- | --- |
| 400 | role 写错、tool_result 顺序错误、schema 不合法、max_tokens 缺失 | 记录 request snapshot，标记为 developer error，不自动盲重试 | 通常不重试 |
| 401 / 403 | API key 无效、权限不足、模型不可用 | 熔断当前 provider，提示配置问题，避免循环调用 | 不重试 |
| 429 | RPM / TPM / 并发限制触发 | 指数退避，按 run_id 记录 retry_count，可切备用模型 | 可重试 |
| 5xx | Provider 短暂故障 | 短重试 + jitter；超过上限后 fallback 或切 provider | 可重试 |
| Streaming parse error | 网络中断、JSON delta 未拼完整、[DONE] 前连接断开 | 保留 partial buffer，标记 stream_incomplete，必要时非流式重试 | 谨慎重试 |
| Tool input invalid | 模型填参不符合 schema 或业务约束 | schema validation error 回填给模型，触发 self-correction | 可在 loop 内修正 |

<a id="trace-cost-context"></a>
## Trace、成本与上下文管理

`Agent 可观测性不是日志打印，而是结构化证据链`

| Trace 字段 | 来源 | 用途 | 说明 |
| --- | --- | --- | --- |
| provider / model | 请求配置 + 响应体 | 模型效果、成本和故障归因 | 响应中的实际 model 可能与请求别名不同 |
| request_id | OpenAI id / Anthropic id | 和 provider 侧日志对齐 | 写入每次 LLM 调用 trace |
| stop_reason | finish_reason / stop_reason | 解释 Agent 为什么停、为什么调工具、为什么失败 | 建议使用内部枚举 |
| tool_calls | 响应中的工具调用 | 复盘工具选择、参数质量和风险动作 | 记录前先做敏感字段脱敏 |
| latency_ms / retry_count | Runtime 计时与重试器 | 定位慢请求、限流和 provider 抖动 | 按 turn 记录，不只按 run 记录 |
| input_tokens / output_tokens | usage 字段 | 成本统计和上下文膨胀预警 | OpenAI / Anthropic 字段名不同，Adapter 统一 |
| reasoning_tokens / thinking_budget | reasoning / thinking 相关 usage 或请求参数 | 分析推理成本和复杂问题开销 | 不要只看 output_tokens |
| cache_read_tokens | cached_tokens / cache_read_input_tokens | 评估 prompt cache 是否生效 | 适合长 system prompt 或工具说明复用场景 |
| redaction_applied | Runtime 脱敏模块 | 公开演示和生产安全边界 | 保留“已脱敏”证据，不保留原文敏感值 |

### 上下文管理建议

| 问题 | 风险 | 处理方式 |
| --- | --- | --- |
| 历史消息无限追加 | 成本上升、上下文被挤爆、工具结果噪声干扰推理 | 按 turn 保留关键证据，对旧工具结果做摘要 |
| System prompt 很长 | 每次请求重复计费，延迟增加 | 使用 prompt cache；Trace 中记录 cache 命中 token |
| Reasoning / thinking 打开过大 | 复杂问题质量提升，但成本不可见地增加 | 按 workflow 设置预算，简单分类不用高 reasoning |
| 工具返回过大 | LLM 被日志噪声淹没，诊断变慢 | 工具侧先聚合、截断、排序，只回填证据摘要 |

<a id="gotchas"></a>
## 常见踩坑

| 场景 | OpenAI 注意 | Anthropic 注意 |
| --- | --- | --- |
| 多轮历史维护 | tool_calls 的 assistant message 要完整保存（包括 content=null 那条） | assistant 的 content 是数组，要整个 block 数组存，不能只存 text |
| 工具参数解析 | arguments 是 字符串 ，必须 JSON.parse，streaming 时需拼接完整再 parse | input 是对象直接用，但 streaming 的 input_json_delta 需拼接后再 parse |
| System prompt | 新模型推荐 role="developer"，兼容服务通常只认 "system" | messages 里写 role="system" 直接报错 |
| 结构化输出 | response_format.type="json_schema" 可精确约束 | 没有 response_format，用 tool_choice 强制调特定工具 来实现 |
| max_tokens | 可以不填（有默认值） | 必须填， 忘填直接报错 |
| 工具调用 ID | 回填用 tool_call_id ，对应 tool_calls[].id（call_ 前缀） | 回填用 tool_use_id ，对应 content[].id（toulu_ 前缀） |
| 请求头 | Authorization: Bearer $OPENAI_API_KEY | x-api-key + anthropic-version 必须带 ，缺少报 400 |
| temperature 上限 | 最高 2.0 | 最高 1.0 ，超过报错 |
| usage 字段名 | prompt_tokens / completion_tokens | input_tokens / output_tokens （不同） |

<a id="openai-responses"></a>
## OpenAI Responses API 参数

`POST /v1/responses`

> 定位： Responses API 是 OpenAI 面向新项目的统一响应接口。和 Chat Completions 相比，它把文本、多模态输入、内建工具、function calling、结构化输出、reasoning 和状态续接放到同一个 response 对象里。做 Agent Runtime 时，建议把它当作新的 OpenAI Adapter 目标，而不是直接把业务逻辑写死在原始字段上。

| 字段 | 类型 | 典型取值 / 结构 | 说明与 Agent 关注点 |
| --- | --- | --- | --- |
| model 必填 | string | gpt-5 gpt-4.1 o3 | 指定模型。模型名变化快，生产环境以官方 model list 为准；Agent 代码不要把具体模型名写死在 workflow 里 |
| input 重点 | string/array | string message items input_text input_image input_file | 核心输入字段。可以是简单字符串，也可以是 item list。相比 Chat Completions 的 messages，Responses API 更强调统一 input item 模型，适合多模态与状态续接 |
| instructions 重点 | string/array | system / developer instructions | 系统或开发者指令。配合 previous_response_id 使用时，新的 instructions 不会自动继承上一轮，适合 Agent 在不同 workflow 间切换系统策略 |
| tools 重点 | array | function file_search web_search computer_use | 工具列表。既支持自定义 function，也支持 OpenAI 内建工具。Agent Runtime 仍然要维护自己的 ToolRegistry、白名单和风险策略，不能只依赖 provider tools |
| tool_choice 重点 | string/object | auto none required 指定工具 | 控制工具调用策略。建议在 Adapter 层统一为 allow / deny / require / forceTool，不要让业务层感知各 provider 原始格式 |
| text.format | object | text json_schema json_object | 结构化输出配置。Responses API 中对应 Chat Completions 的 response_format；新模型优先使用 json_schema，而不是旧 JSON mode |
| reasoning 重点 | object | effort summary | reasoning 模型相关配置。影响质量、延迟和成本；Agent 应按 workflow 设置预算，分类/路由类任务不应默认开启高 reasoning |
| max_output_tokens 重点 | integer | 输出上限 | 限制响应生成 token，上限包含可见输出和 reasoning token。对应 Chat Completions 的 max_completion_tokens 语义，但字段名不同 |
| previous_response_id | string | resp_xxx | 用于服务端状态续接。适合简单会话；如果 Agent Runtime 要完整可复盘、可 replay，仍建议自己持久化 input/output/trace |
| truncation | string | disabled auto | 上下文超长处理策略。disabled 默认更可控，超长时直接失败；auto 会从会话开头丢弃内容，生产 Agent 需要谨慎使用，避免丢关键证据 |
| stream | boolean | true false | 开启 SSE 流式输出。Responses API 的 streaming event 类型比 Chat Completions 更细，Adapter 应统一成 text_delta / tool_call_delta / done |
| metadata | object | 自定义 KV | 写入 run_id / workflow / user_id 等追踪信息，便于成本归因和排障 |

### Responses API 返回结构关注点

| 字段 | 含义 | Agent 关注点 |
| --- | --- | --- |
| id | response id，例如 resp_xxx | 写入 Trace；如果使用 previous_response_id，下一轮会引用它 |
| status | completed / incomplete / failed 等状态 | 不要只看有没有文本输出；incomplete 时要检查 incomplete_details |
| output[] | 模型输出 item 列表 | 可能包含 message、function/tool call、reasoning 等多类 item；Adapter 需要分类提取 |
| output_text | SDK/响应对象中的文本聚合便捷字段 | 适合最终文本读取；但 Agent Loop 判断工具调用时不能只读 output_text |
| usage.input_tokens / output_tokens | 输入和输出 token 统计 | 字段名更接近 Anthropic；Adapter 可统一为 inputTokens / outputTokens |
| usage.output_tokens_details.reasoning_tokens | reasoning token 消耗 | 成本和延迟分析必须记录，不能只看可见输出 token |
| incomplete_details.reason | 不完整原因，例如 max_output_tokens | 映射成内部 stopReason=truncated，并进入重试/续写策略 |

<a id="openai-req"></a>
## OpenAI Chat Completions 请求参数

`POST /v1/chat/completions`

| 字段 | 类型 | 枚举值 / 取值范围 | 说明与 Agent 关注点 |
| --- | --- | --- | --- |
| model 必填 | string | gpt-4o gpt-4o-mini gpt-4.1 o3 o4-mini | 指定模型。o 系列是 reasoning 模型，不支持 temperature 参数 |
| messages 必填 重点 | array | role: system developer user assistant tool | Agent 核心。完整对话历史 + tool call + tool result 全部在这里。 role=tool 是工具结果专属 role ，必须带 tool_call_id |
| tools 重点 | array | type: function | 工具定义列表，每个工具含 name / description / parameters （JSON Schema） |
| tool_choice 重点 | string/obj | auto none required {type,function} | auto=模型自决；none=禁止工具；required=必须调某工具（模型自选）；对象形式指定具体工具名 |
| parallel_tool_calls | boolean | true false | 是否允许单次返回多个 tool call。Agent 并行工具场景需开启，默认 true |
| response_format 重点 | object | type: text json_object json_schema | 结构化输出。json_schema 可指定精确 schema。 Anthropic 没有此参数 ，需用 tool_choice 强制工具输出代替 |
| temperature | number | 0.0 ~ 2.0，默认 1.0 | 控制随机性。工具调用场景建议调低（0~0.3）； o 系列不支持此参数 |
| top_p | number | 0.0 ~ 1.0，默认 1.0 | nucleus sampling。不要和 temperature 同时调整，二选一 |
| max_tokens / max_completion_tokens 重点 | integer | 模型上限各不同 | 控制最大输出 token。 reasoning 模型用 max_completion_tokens （含思考 token）。不设则模型可能输出很长 |
| stream | boolean | true false | 是否流式输出。true 时返回 SSE， tool call arguments 需拼接后再 JSON.parse |
| stop | string/array | 自定义字符串，最多 4 个 | 遇到指定序列停止。finish_reason 会变为 "stop" |
| n | integer | 默认 1 | 生成多个候选。生产保持 1，多候选成本倍增且 Agent 逻辑复杂 |
| store | boolean | true false | 是否存储请求/响应到 OpenAI 侧。涉及隐私合规时设 false |
| metadata | object | 自定义 KV | 打业务标记，便于 Trace 和成本归因。建议带 run_id / user_id / workflow |
| logprobs / top_logprobs | bool/int | top_logprobs: 0~20 | 返回 token 概率。Eval / 分类置信度场景使用，普通 Agent 不需要 |

<a id="anthropic-req"></a>
## Anthropic Messages API 请求参数

`POST /v1/messages`

> 请求头必须带 anthropic-version: 2023-06-01 ，缺少直接报 400。鉴权用 x-api-key ，不是 Authorization Bearer。

| 字段 | 类型 | 枚举值 / 取值范围 | 说明与 Agent 关注点 |
| --- | --- | --- | --- |
| model 必填 | string | claude-opus-4-5 claude-sonnet-4-5 claude-haiku-4-5 | 指定 Claude 模型版本 |
| max_tokens 必填 重点 | integer | 模型上限各不同 | Anthropic 特有——必填 ，OpenAI 是可选。开启 thinking 时需设大（16000+），thinking token 也计入此上限 |
| messages 必填 重点 | array | role: user role: assistant | 只有两种 role 。tool result 也用 user role 承载（content block type=tool_result）。写 role=system 会报错 |
| system 重点 | string/array | — | 顶层字段，不在 messages 里 。可以是字符串，也可以是 content block 数组（支持 prompt caching） |
| tools 重点 | array | type: custom computer_use text_editor bash | 自定义工具含 name / description / input_schema （注意不是 parameters，是 Anthropic 特有字段名） |
| tool_choice 重点 | object | {type:"auto"} {type:"any"} {type:"tool",name:"..."} {type:"none"} | 格式是对象，不是字符串 。any=必须调工具（模型自选）；tool=强制指定工具名。对应 OpenAI 的 required 和 function 对象 |
| temperature | number | 0.0 ~ 1.0，默认 1.0 | 上限是 1.0（OpenAI 是 2.0） 。开启 thinking 时不建议修改 |
| top_p | number | 0.0 ~ 1.0 | 同 OpenAI，不要和 temperature 同时调 |
| top_k | integer | 正整数 | OpenAI 没有此参数。限制候选 token 数量，一般不需要调 |
| stream | boolean | true false | 事件类型更细（message_start / content_block_delta / message_stop），结构和 OpenAI 不同 |
| stop_sequences | array | 字符串数组 | 对应 OpenAI 的 stop。命中后 stop_reason="stop_sequence" |
| thinking 重点 | object | type: enabled disabled adaptive | 开启思考模式。enabled 需指定 budget_tokens；adaptive 让模型自决思考深度（新模型推荐） |
| metadata | object | 含 user_id | 打用户标记，用于安全和滥用检测 |

<a id="openai-resp"></a>
## OpenAI Chat Completions 返回参数

`Chat Completions Response`

| 字段路径 | 类型 | 枚举值 | 说明与 Agent 关注点 |
| --- | --- | --- | --- |
| id | string | chatcmpl-xxx | 请求唯一 ID，写入 Trace 用 |
| model | string | — | 实际使用的模型（可能和请求别名不同） |
| choices[0].finish_reason 重点 | string | stop tool_calls length content_filter | Agent Loop 最核心判断字段 。tool_calls=需执行工具；length=被截断；content_filter=安全拒绝 |
| choices[0].message.role | string | assistant | 固定为 assistant |
| choices[0].message.content | string/null | — | 文本内容。 有 tool_calls 时此字段为 null |
| choices[0].message.tool_calls 重点 | array/null | — | 工具调用列表。每个含 id / type / function.name / function.arguments（字符串，需 JSON.parse） |
| choices[0].message.refusal | string/null | — | 安全拒绝时的说明文字 |
| usage 重点 | object | — | 含 prompt_tokens / completion_tokens / total_tokens。判断上下文是否快满，计算成本 |
| usage.prompt_tokens_details | object | — | 细分 cached_tokens（命中缓存的 token 数），做成本优化时关注 |
| usage.completion_tokens_details | object | — | 含 reasoning_tokens（o 系列模型思考消耗的 token 数） |
| system_fingerprint | string | — | 模型系统版本指纹，调试输出一致性问题时用 |
| created | integer | Unix timestamp | 请求创建时间，写 Trace 时记录 |

<a id="anthropic-resp"></a>
## Anthropic Messages API 返回参数

`Messages Response`

| 字段路径 | 类型 | 枚举值 | 说明与 Agent 关注点 |
| --- | --- | --- | --- |
| id | string | msg_xxx | 请求唯一 ID，写 Trace 用 |
| type | string | message | 固定为 message |
| role | string | assistant | 固定为 assistant |
| stop_reason 重点 | string | end_turn tool_use max_tokens stop_sequence refusal pause_turn | 在顶层 （OpenAI 在 choices[0] 里）。pause_turn 用于长任务中断恢复（HITL 场景） |
| content 重点 | array | — | 内容块数组。 可同时含文本和工具调用 （OpenAI 有 tool_calls 时 content=null，Anthropic 可并存） |
| content[].type 重点 | string | text tool_use thinking redacted_thinking | block 类型。tool_use 表示发起工具调用；thinking 是思考过程（开启 thinking 模式时出现） |
| content[tool_use].id | string | toolu_xxx | 工具调用 ID。 回填 tool_result 时字段名为 tool_use_id（不是 tool_call_id） |
| content[tool_use].input 重点 | object | — | 已是 JSON 对象 ，直接用（OpenAI 是字符串，需 JSON.parse）。streaming 时 input_json_delta 需拼接后再 parse |
| model | string | — | 实际使用的模型 |
| stop_sequence | string/null | — | 命中的停止序列，stop_reason=stop_sequence 时不为 null |
| usage 重点 | object | — | 含 input_tokens / output_tokens（字段名与 OpenAI 不同） |
| usage.cache_read_input_tokens | integer | — | 命中 prompt cache 的 token 数。做 system prompt 缓存优化时关注 |

<a id="tool-schema"></a>
## Tool Schema 用法

`告诉模型可以调用哪些工具、参数约束是什么`

> Tool Schema 本质是 JSON Schema，放在 tools 数组里。 description 字段 决定模型什么时候调用这个工具，写得越清晰，模型选工具越准确。

**完整 HTTP 请求 OpenAI**

```ts
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "你是 SRE 助手。" },
    { "role": "user", "content": "查 api-gateway 近1小时 ERROR 日志" }
  ],
  "tools": [
    {
      // OpenAI：外层包一层 type:function
      "type": "function",
      "function": {
        "name": "query_logs",
        "description": "查询服务错误日志。用户想了解服务错误时使用。",
        // OpenAI 字段名：parameters
        "parameters": {
          "type": "object",
          "properties": {
            "service": { "type": "string" },
            "duration_minutes": { "type": "integer" },
            "level": {
              "type": "string",
              "enum": ["ERROR", "WARN", "INFO"]
            }
          },
          "required": ["service", "level"],
          "additionalProperties": false
        }
      }
    }
  ],
  // 字符串形式
  "tool_choice": "auto"
}
```

**完整 HTTP 请求 Anthropic**

```ts
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "system": "你是 SRE 助手。",
  "messages": [
    { "role": "user", "content": "查 api-gateway 近1小时 ERROR 日志" }
  ],
  "tools": [
    {
      // Anthropic：直接平铺，没有外层 type:function
      "name": "query_logs",
      "description": "查询服务错误日志。用户想了解服务错误时使用。",
      // Anthropic 字段名：input_schema（不是 parameters）
      "input_schema": {
        "type": "object",
        "properties": {
          "service": { "type": "string" },
          "duration_minutes": { "type": "integer" },
          "level": {
            "type": "string",
            "enum": ["ERROR", "WARN", "INFO"]
          }
        },
        "required": ["service", "level"]
      }
    }
  ],
  // 始终是对象形式
  "tool_choice": { "type": "auto" }
}
```

### 模型返回（工具调用）对比

**Response Body OpenAI**

```ts
{
  "choices": [{
    "message": {
      "role": "assistant",
      // content=null 当有工具调用时
      "content": null,
      "tool_calls": [{
        "id": "call_xyz",
        "type": "function",
        "function": {
          "name": "query_logs",
          // arguments 是字符串，需 JSON.parse
          "arguments": "{\"service\":\"api-gateway\",\"level\":\"ERROR\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

**Response Body Anthropic**

```ts
{
  "stop_reason": "tool_use",
  // content 可同时含文字和工具调用
  "content": [
    {
      "type": "text",
      "text": "我来查一下日志。"
    },
    {
      "type": "tool_use",
      "id": "toulu_xyz",
      "name": "query_logs",
      // input 已是对象，直接用
      "input": {
        "service": "api-gateway",
        "level": "ERROR"
      }
    }
  ]
}
```

### tool_choice 四种策略完整示例

**tool_choice 示例 OpenAI**

```ts
// 模型自己决定（默认）
"tool_choice": "auto"

// 禁止调工具，只输出文字
"tool_choice": "none"

// 必须调工具，模型自选哪个
"tool_choice": "required"

// 强制调指定工具
"tool_choice": {
  "type": "function",
  "function": { "name": "query_logs" }
}
```

**tool_choice 示例 Anthropic**

```ts
// 模型自己决定（始终是对象）
"tool_choice": { "type": "auto" }

// 禁止调工具
"tool_choice": { "type": "none" }

// 必须调工具，模型自选哪个（对应 OpenAI required）
"tool_choice": { "type": "any" }

// 强制调指定工具
"tool_choice": {
  "type": "tool",
  "name": "query_logs"
}
```

<a id="structured-output"></a>
## Structured Output

`让模型的文字输出本身就是合法 JSON`

> 适合 不需要模型执行动作、只需要结构化分析结果 的场景（如 Router 决策、诊断报告、意图分类）。OpenAI 用 response_format ，Anthropic 没有此参数，用"输出工具 + tool_choice 强制"代替。

**完整 HTTP 请求 OpenAI — response_format**

```ts
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "你是 SRE 诊断助手，输出结构化报告。" },
    { "role": "user", "content": "api-gateway CPU 95%，大量 502" }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "diagnosis_report",
      // strict:true 保证字段不多不少
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "severity": {
            "type": "string",
            "enum": ["P0","P1","P2","P3"]
          },
          "root_cause": { "type": "string" },
          "actions": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "action":   { "type": "string" },
                "priority": { "type": "integer" },
                "need_approval": { "type": "boolean" }
              },
              "required": ["action","priority","need_approval"],
              "additionalProperties": false
            }
          },
          "confidence": { "type": "number" }
        },
        "required": ["severity","root_cause","actions","confidence"],
        "additionalProperties": false
      }
    }
  }
}
```

**完整 HTTP 请求 Anthropic — 输出工具模拟**

```ts
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "system": "你是 SRE 诊断助手，输出结构化报告。",
  "messages": [
    { "role": "user", "content": "api-gateway CPU 95%，大量 502" }
  ],
  // 定义一个"输出工具"，实际上是结构化输出的载体
  "tools": [{
    "name": "emit_diagnosis",
    "description": "输出诊断报告。分析完成后必须调用此工具。",
    "input_schema": {
      "type": "object",
      "properties": {
        "severity": {
          "type": "string",
          "enum": ["P0","P1","P2","P3"]
        },
        "root_cause": { "type": "string" },
        "actions": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "action":       { "type": "string" },
              "priority":     { "type": "integer" },
              "need_approval": { "type": "boolean" }
            },
            "required": ["action","priority","need_approval"]
          }
        },
        "confidence": { "type": "number" }
      },
      "required": ["severity","root_cause","actions","confidence"]
    }
  }],
  // 强制调用，模型填的参数就是结构化结果
  "tool_choice": { "type": "tool", "name": "emit_diagnosis" }
}
```

### 模型返回 & 如何取结果

**Response Body OpenAI**

```ts
{
  "choices": [{
    "message": {
      "role": "assistant",
      // content 是 JSON 字符串，需 JSON.parse
      "content": "{\"severity\":\"P0\",\"root_cause\":\"连接池耗尽\",\"actions\":[{\"action\":\"限流\",\"priority\":1,\"need_approval\":false}],\"confidence\":0.87}"
    },
    "finish_reason": "stop"
  }]
}

// 取结果
const report = JSON.parse(
  response.choices[0].message.content
)
```

**Response Body Anthropic**

```ts
{
  "stop_reason": "tool_use",
  "content": [{
    "type": "tool_use",
    "name": "emit_diagnosis",
    // input 已是对象，直接用，无需 JSON.parse
    "input": {
      "severity": "P0",
      "root_cause": "连接池耗尽",
      "actions": [{
        "action": "限流",
        "priority": 1,
        "need_approval": false
      }],
      "confidence": 0.87
    }
  }]
}

// 取结果
const toolBlock = response.content.find(b => b.type === "tool_use")
const report    = toolBlock.input  // 直接用
```

<a id="schema-vs-structured"></a>
## Tool Schema vs Structured Output 如何选择

| 维度 | Tool Schema | Structured Output |
| --- | --- | --- |
| 模型行为 | 决定是否调用、填参数 | 直接输出结构化文字 |
| 是否真正执行动作 | 是（查日志、重启服务） | 否（只需要分析结果） |
| 后续还需要 LLM | 是（工具结果回填，继续推理） | 否（拿到结果直接用） |
| 典型场景 | query_logs、restart_service、call_api | Router 决策、诊断报告、意图分类 |
| Anthropic 实现方式 | tools + tool_choice: auto | 输出工具 + tool_choice: {type:"tool",name:"..."} |

> 在 SRE Agent 里的典型分工： Router 阶段判断 workflow 类型 → Structured Output（只要决策，不执行动作）。工具执行阶段调 query_logs / check_metrics → Tool Schema（真正执行，结果回填继续推理）。最终生成诊断报告 → Structured Output（把所有结论整理成结构化格式）。

<a id="sse-protocol"></a>
## SSE 协议格式

`W3C 标准，OpenAI / Anthropic 均基于此`

> SSE 是 W3C 标准协议 ，规范地址： html.spec.whatwg.org 。OpenAI 和 Anthropic 都基于这个标准，但 data: 里放什么内容是各家自定义的。

### 协议规定的四个字段

| 字段名 | 是否必须 | 说明 |
| --- | --- | --- |
| event: | 可选 | 事件类型名。OpenAI 不用，Anthropic 每个事件都带 |
| data: | 必须 | 数据内容。两家都把 JSON 放在这里 |
| id: | 可选 | 事件 ID，用于断线重连（Last-Event-ID） |
| retry: | 可选 | 断线后重连等待毫秒数 |

### 格式规则

**SSE 格式规范**

```ts
# 单个事件结构（字段之间用 \n 分隔）
event: content_block_delta
data: {"type":"content_block_delta","index":0}

# 事件与事件之间用【空行】分隔，这是事件边界的唯一标志
event: message_start
data: {"type":"message_start"}

event: content_block_start
data: {"type":"content_block_start","index":0}

# data: 可以多行，会被自动拼接（中间加 \n）
data: line1
data: line2
# 等价于 data 值为 "line1\nline2"

# 以 : 开头是注释行，客户端忽略
: this is a comment
```

### 两家实现对比

| 对比项 | OpenAI | Anthropic |
| --- | --- | --- |
| event: 字段 | 不使用 | 每个事件都带 |
| data: 内容 | JSON 或 [DONE] | 始终是 JSON |
| 事件边界 | 空行 \n\n | 空行 \n\n |
| 结束标志 | data: [DONE] （不是 JSON） | event: message_stop |
| 事件类型判断 | 靠 data 里的 finish_reason | 靠 event: 字段值 |
| Content-Type | text/event-stream （W3C 标准规定） |  |

<a id="sse-openai"></a>
## OpenAI SSE 解析

`文本输出场景`

### 请求

**HTTP Request OpenAI**

```ts
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json

{
  "model": "gpt-4o",
  "stream": true,
  "messages": [
    { "role": "system", "content": "你是 SRE 助手。" },
    { "role": "user",   "content": "解释 CPU 飙高的常见原因" }
  ]
}
```

### SSE 响应流

**SSE Stream OpenAI**

```ts
# 第一片：角色声明，content 为空字符串
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

# 后续片：逐 token 推送文字
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"CPU"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 飙高"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"的常见原因"},"finish_reason":null}]}

# 最后片：delta 为空，finish_reason 出现
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

# 结束标志：不是 JSON，不能 JSON.parse
data: [DONE]
```

### 解析代码

> buffer 拼接不能省。 TCP 传输时，一次 read() 拿到的字节数不确定，一个 SSE 事件可能被切成两个 chunk，必须用 buffer 拼接再按 \n\n 切事件。

**TypeScript OpenAI**

```ts
const response = await fetch("https://api.openai.com/v1/chat/completions", {
  method: "POST",
  headers: { "Authorization": `Bearer ${apiKey}`, "Content-Type": "application/json" },
  body: JSON.stringify({ model: "gpt-4o", stream: true, messages })
})

const reader  = response.body!.getReader()
const decoder = new TextDecoder()
let buffer = ""

while (true) {
  const { done, value } = await reader.read()
  if (done) break

  buffer += decoder.decode(value, { stream: true })

  // 按空行切事件边界，最后一段可能不完整，留给下次
  const events = buffer.split("\n\n")
  buffer = events.pop()!

  for (const event of events) {
    const dataLine = event.split("\n").find(l => l.startsWith("data: "))
    if (!dataLine) continue

    const raw = dataLine.slice(6).trim()

    // [DONE] 不是 JSON，必须先判断
    if (raw === "[DONE]") return

    const chunk      = JSON.parse(raw)
    const delta      = chunk.choices[0]?.delta
    const stopReason = chunk.choices[0]?.finish_reason

    // 文本增量，实时输出
    if (delta?.content) {
      process.stdout.write(delta.content)
    }

    if (stopReason === "stop")       { /* 正常结束 */ }
    if (stopReason === "tool_calls") { /* 进入工具执行，见工具调用 Streaming */ }
  }
}
```

<a id="sse-anthropic"></a>
## Anthropic SSE 解析

`事件类型更细，靠 event: 字段区分`

### 请求

**HTTP Request Anthropic**

```ts
POST https://api.anthropic.com/v1/messages
x-api-key: $ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "stream": true,
  "system": "你是 SRE 助手。",
  "messages": [
    { "role": "user", "content": "解释 CPU 飙高的常见原因" }
  ]
}
```

### SSE 响应流（文本输出）

**SSE Stream Anthropic**

```ts
# 每个事件有两行：event: 在前，data: 在后

event: message_start
data: {"type":"message_start","message":{"id":"msg_abc","role":"assistant","content":[],"stop_reason":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"CPU"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" 飙高的常见原因"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

# stop_reason 在 message_delta 里，不在 content 里
event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":45}}

# 结束标志（不是 [DONE]，是一个正常 JSON 事件）
event: message_stop
data: {"type":"message_stop"}
```

### 解析代码

**TypeScript Anthropic**

```ts
let buffer     = ""
let stopReason = ""

// 按 content block index 收集文字和工具参数
const blocks: Record<number, { type: "text"|"tool_use"; textBuf: string; jsonBuf: string; id?: string; name?: string }> = {}

while (true) {
  const { done, value } = await reader.read()
  if (done) break

  buffer += decoder.decode(value, { stream: true })
  const events = buffer.split("\n\n")
  buffer = events.pop()!

  for (const event of events) {
    const lines     = event.split("\n")
    const eventType = lines.find(l => l.startsWith("event: "))?.slice(7) ?? ""
    const dataLine  = lines.find(l => l.startsWith("data: "))
    if (!dataLine) continue

    const data = JSON.parse(dataLine.slice(6))  // Anthropic data 始终是 JSON

    switch (eventType) {
      case "content_block_start":
        blocks[data.index] = {
          type:    data.content_block.type,
          id:      data.content_block.id,
          name:    data.content_block.name,
          textBuf: "", jsonBuf: ""
        }
        break

      case "content_block_delta":
        const b = blocks[data.index]
        if (data.delta.type === "text_delta") {
          b.textBuf += data.delta.text
          process.stdout.write(data.delta.text)  // 实时输出
        }
        if (data.delta.type === "input_json_delta") {
          b.jsonBuf += data.delta.partial_json    // 工具参数逐片拼接
        }
        break

      case "message_delta":
        stopReason = data.delta.stop_reason
        break

      case "message_stop":
        if (stopReason === "tool_use") {
          // jsonBuf 已拼完，现在可以 parse
          const toolCalls = Object.values(blocks)
            .filter(b => b.type === "tool_use")
            .map(b => ({ id: b.id, name: b.name, input: JSON.parse(b.jsonBuf) }))
          // 执行工具...
        }
        break
    }
  }
}
```

<a id="sse-tool-stream"></a>
## 工具调用 Streaming

`arguments / input 分片传回，必须拼接后再 JSON.parse`

> 最容易出错的地方： 工具参数在 streaming 里是字符串片段逐步推送的，不能拿到一片就 JSON.parse，必须等 content_block_stop（Anthropic）或 finish_reason=tool_calls（OpenAI）出现后，整体 parse 一次。

**SSE Stream（工具调用） OpenAI**

```ts
# content=null，tool_calls 出现
data: {"choices":[{"delta":{"role":"assistant","content":null},"finish_reason":null}]}

# 第一片：id + name + arguments 首片（空字符串）
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"id":"call_xyz","type":"function","function":{"name":"query_logs","arguments":""}}]},"finish_reason":null}]}

# arguments 逐片推送（JSON 字符串被切开）
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"ser"}}]},"finish_reason":null}]}

data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"vice\":"}}]},"finish_reason":null}]}

data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"\"api-gateway\"}"}}]},"finish_reason":null}]}

# finish_reason 出现 → 此时 arguments 已全部到达
data: {"choices":[{"index":0,"delta":{},"finish_reason":"tool_calls"}]}

data: [DONE]
```

**SSE Stream（工具调用） Anthropic**

```ts
# content_block_start 声明这是 tool_use block
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_xyz","name":"query_logs","input":{}}}

# input_json_delta 逐片推送参数
event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"ser"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"vice\":\"api-gateway\""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"}"}}

# content_block_stop → jsonBuf 拼完，可以 parse
event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"}}
```

### OpenAI 多工具并行时的 index 处理

**TypeScript OpenAI**

```ts
// 用 map 按 index 分别收集每个工具调用的 arguments
const buffers: Record<number, { id: string; name: string; argsBuf: string }> = {}

if (delta?.tool_calls) {
  for (const tc of delta.tool_calls) {
    const idx = tc.index  // 多工具时靠 index 区分

    if (!buffers[idx]) {
      buffers[idx] = { id: "", name: "", argsBuf: "" }
    }

    // id 和 name 只在第一片出现
    if (tc.id)             buffers[idx].id   = tc.id
    if (tc.function?.name) buffers[idx].name = tc.function.name

    // arguments 逐片累加
    if (tc.function?.arguments) {
      buffers[idx].argsBuf += tc.function.arguments
    }
  }
}

if (finishReason === "tool_calls") {
  // 全部拼完，统一 parse
  const toolCalls = Object.values(buffers).map(b => ({
    id:        b.id,
    name:      b.name,
    arguments: JSON.parse(b.argsBuf),  // 整体 parse
  }))
}
```

<a id="ex-system"></a>
## 完整 HTTP 示例：System Prompt

**HTTP Request OpenAI**

```ts
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json

{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "你是 SRE 助手。"
    },
    {
      "role": "user",
      "content": "CPU 飙到 95%"
    }
  ]
}
```

**HTTP Request Anthropic**

```ts
POST https://api.anthropic.com/v1/messages
x-api-key: $ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  // system 是顶层字段，不在 messages 里
  "system": "你是 SRE 助手。",
  "messages": [
    {
      "role": "user",
      "content": "CPU 飙到 95%"
    }
  ]
}
```

<a id="ex-tools"></a>
## 完整 HTTP 示例：Tools Schema

**HTTP Request OpenAI**

```ts
{
  "model": "gpt-4o",
  "messages": [...],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "query_logs",
        "description": "查询错误日志",
        // 字段名：parameters
        "parameters": {
          "type": "object",
          "properties": {
            "service": {
              "type": "string"
            },
            "level": {
              "type": "string",
              "enum": ["ERROR", "WARN"]
            }
          },
          "required": ["service"]
        }
      }
    }
  ],
  // tool_choice 可以是字符串
  "tool_choice": "auto"
}
```

**HTTP Request Anthropic**

```ts
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "system": "...",
  "messages": [...],
  "tools": [
    {
      // 没有外层 type:function 包裹
      "name": "query_logs",
      "description": "查询错误日志",
      // 字段名：input_schema（不是 parameters）
      "input_schema": {
        "type": "object",
        "properties": {
          "service": {
            "type": "string"
          },
          "level": {
            "type": "string",
            "enum": ["ERROR", "WARN"]
          }
        },
        "required": ["service"]
      }
    }
  ],
  // tool_choice 始终是对象
  "tool_choice": { "type": "auto" }
}
```

<a id="ex-toolresult"></a>
## 完整 HTTP 示例：工具结果回填

`差异最大的地方`

> 工具执行完后，必须把完整历史（含 assistant 工具调用那条）+ 工具结果一起发给 LLM。两家格式完全不同，这是最容易写错的地方。

**HTTP Request（含回填） OpenAI**

```ts
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "你是 SRE 助手。"
    },
    {
      "role": "user",
      "content": "CPU 飙到 95%"
    },
    // ① assistant 原始输出（content=null）
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_xyz789",
        "type": "function",
        "function": {
          "name": "query_logs",
          // arguments 是字符串！
          "arguments": "{\"service\":\"api-gw\"}"
        }
      }]
    },
    // ② tool result：专属 role="tool"
    {
      "role": "tool",
      "tool_call_id": "call_xyz789",
      "content": "{\"logs\":[...]}"
    }
  ]
}
```

**HTTP Request（含回填） Anthropic**

```ts
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "system": "你是 SRE 助手。",
  "messages": [
    {
      "role": "user",
      "content": "CPU 飙到 95%"
    },
    // ① assistant：content 是数组，含文字+工具调用
    {
      "role": "assistant",
      "content": [
        { "type": "text",
          "text": "我查一下日志。" },
        { "type": "tool_use",
          "id": "toolu_xyz789",
          "name": "query_logs",
          // input 是对象，直接用
          "input": {
            "service": "api-gw"
          }
        }
      ]
    },
    // ② tool result：用 role="user"，不是 tool
    {
      "role": "user",
      "content": [{
        "type": "tool_result",
        "tool_use_id": "toolu_xyz789",
        "content": "{\"logs\":[...]}"
      }]
    }
  ]
}
```

<a id="ex-response"></a>
## 完整 HTTP 示例：Response 解析

`模型返回体`

### 正常结束（文本输出）

**Response Body OpenAI**

```ts
{
  "id": "chatcmpl-abc",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "建议先限流..."
    },
    // Agent Loop 判断字段
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 120,
    "completion_tokens": 45
  }
}
```

**Response Body Anthropic**

```ts
{
  "id": "msg_abc",
  // stop_reason 在顶层
  "stop_reason": "end_turn",
  "content": [{
    "type": "text",
    "text": "建议先限流..."
  }],
  "usage": {
    // 字段名不同
    "input_tokens": 120,
    "output_tokens": 45
  }
}
```

### 需要调工具

**Response Body OpenAI**

```ts
{
  "choices": [{
    "message": {
      "role": "assistant",
      // content=null 当有工具调用时
      "content": null,
      "tool_calls": [{
        "id": "call_xyz789",
        "type": "function",
        "function": {
          "name": "query_logs",
          // arguments 是字符串，需 JSON.parse
          "arguments": "{\"service\":\"api-gw\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

**Response Body Anthropic**

```ts
{
  "stop_reason": "tool_use",
  // content 可同时含文字和工具调用
  "content": [
    {
      "type": "text",
      "text": "我来查一下日志。"
    },
    {
      "type": "tool_use",
      "id": "toolu_xyz789",
      "name": "query_logs",
      // input 已是对象，直接用
      "input": {
        "service": "api-gw"
      }
    }
  ]
}
```

## 资料来源

最后校验日期：2026-06-07

- OpenAI Responses API: https://platform.openai.com/docs/api-reference/responses/create
- OpenAI Chat Completions API: https://platform.openai.com/docs/api-reference/chat/create
- OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use

生产环境使用前，请始终以官方文档为准。
