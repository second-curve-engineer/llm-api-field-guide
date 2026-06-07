# LLM API Field Guide for Agent Engineers

An independent field guide comparing OpenAI Chat Completions and Anthropic Messages API from an Agent Runtime implementation perspective.

This is not a copy of official API documentation. It reorganizes the APIs around the engineering questions that matter when building agents:

- How should an Agent Loop decide whether to continue, call tools, retry, or stop?
- Where do tool schemas, tool results, and structured outputs differ between providers?
- How should streaming events be parsed and assembled?
- Which response fields should be captured into trace, cost, and eval systems?

## Contents

- Request and response parameter comparison
- `finish_reason` vs `stop_reason`
- Tool schema and tool result differences
- Streaming / SSE event parsing
- Tool Schema vs Structured Output decision guide
- Common production pitfalls for Agent implementations

## Live Site

This repository is designed to be deployed as a static site through GitHub Pages.

After pushing to GitHub:

1. Open repository `Settings`
2. Go to `Pages`
3. Choose `Deploy from a branch`
4. Select `main` and `/root`

The site will be served from:

```text
https://second-curve-engineer.github.io/llm-api-field-guide/
```

## Sources

Last verified: 2026-06-07

- OpenAI Chat Completions API: https://platform.openai.com/docs/api-reference/chat/create
- OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use

Always verify production behavior against the official provider documentation.

## Disclaimer

This is an independent engineering note by `second-curve-engineer`. It is not affiliated with OpenAI or Anthropic.

