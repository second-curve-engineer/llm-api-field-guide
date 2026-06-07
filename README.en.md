# LLM API Field Guide for Agent Engineers

English | [中文](README.md)

An independent field guide comparing OpenAI Chat Completions and Anthropic Messages API from an Agent Runtime implementation perspective.

This is not a copy of official API documentation. It reorganizes the APIs around the engineering questions that matter when building agents:

- How should an Agent Loop decide whether to continue, call tools, retry, or stop?
- Where do tool schemas, tool results, and structured outputs differ between providers?
- How should streaming events be parsed and assembled?
- Which response fields should be captured into trace, cost, and eval systems?

## Contents

- Request and response parameter comparison
- `finish_reason` vs `stop_reason`
- OpenAI Responses API vs Chat Completions selection
- Provider Adapter design
- Agent Loop state machine
- Error handling and retry strategy
- Trace, cost, and context management
- Tool schema and tool result differences
- Streaming / SSE event parsing
- Tool Schema vs Structured Output decision guide
- Common production pitfalls for Agent implementations

## Live Site

This repository is published as a static site through GitHub Pages:

```text
https://second-curve-engineer.github.io/llm-api-field-guide/
```

You can also read the Markdown version directly on GitHub:

- [LLM API Field Guide Markdown Version](GUIDE.md)

## Why This Guide Exists

When building a real Agent Runtime, LLM API differences are not just parameter-name differences. They affect:

- Agent Loop state transitions
- Tool Calling argument parsing and tool result injection
- Streaming output assembly
- Trace, eval, and cost telemetry data models
- Provider adapter abstraction boundaries

The goal of this guide is to turn those differences into a practical engineering reference.

## Sources

Last verified: 2026-06-07

- OpenAI Chat Completions API: https://platform.openai.com/docs/api-reference/chat/create
- OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use

Always verify production behavior against the official provider documentation.

## Disclaimer

This is an independent engineering note by `second-curve-engineer`. It is not affiliated with OpenAI or Anthropic.
