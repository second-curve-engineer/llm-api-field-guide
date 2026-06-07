# LLM API Field Guide for Agent Engineers

[English](README.en.md) | 中文

面向 Agent 工程师的 LLM API 对照手册，从 Agent Runtime 实现视角比较 OpenAI Chat Completions 与 Anthropic Messages API。

这不是官方 API 文档的复制版，而是把 API 差异重新组织成 Agent 开发中真正会遇到的工程问题：

- Agent Loop 如何判断继续推理、调用工具、重试或结束？
- OpenAI 与 Anthropic 的 tool schema、tool result、structured output 有哪些关键差异？
- Streaming / SSE 事件应该如何解析和拼接？
- 哪些响应字段应该进入 trace、成本统计和 eval 系统？

## 内容范围

- 请求参数与返回参数对照
- `finish_reason` 与 `stop_reason`
- Tool Schema 与工具结果回填差异
- Streaming / SSE 事件解析
- Tool Schema vs Structured Output 选择指南
- Agent 实现中的常见生产踩坑

## 在线页面

本项目是一个静态页面，已通过 GitHub Pages 发布：

```text
https://second-curve-engineer.github.io/llm-api-field-guide/
```

## 为什么整理这份手册

在实际构建 Agent Runtime 时，LLM API 的差异不会只停留在“参数名不同”这一层。它会影响：

- Agent Loop 的状态判断
- Tool Calling 的参数解析与结果回填
- Streaming 输出的组装方式
- Trace / Eval / 成本统计的数据结构
- 多 provider adapter 的抽象边界

这份手册的目标是把这些差异整理成一份可查、可复用、面向工程实现的参考材料。

## 资料来源

最后校验日期：2026-06-07

- OpenAI Chat Completions API: https://platform.openai.com/docs/api-reference/chat/create
- OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use

生产环境使用前，请始终以官方文档为准。

## 免责声明

这是 `second-curve-engineer` 的独立工程笔记，不隶属于 OpenAI 或 Anthropic，也不代表官方文档。

