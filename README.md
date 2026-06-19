# study

个人工程学习笔记。

## 内容

- [2026-06-18-harness-engineering.md](2026-06-18-harness-engineering.md) —— Harness Engineering 工程实践总结：模型是不确定的内核，工程的价值在它周围的那一整圈确定性外壳。涵盖「用 AI 造 app（agentic coding）」与「把 AI 装进 app（product harness）」两个方向的实战教训，并对标 Anthropic / OpenAI 的 evals 驱动范式。
- [2026-06-19-llm-integration-and-model-selection.md](2026-06-19-llm-integration-and-model-selection.md) —— LLM 接入与「让用户自选模型」：模型只是可替换后端，工程在它外面那层抽象（统一接口、能力感知、BYO-key、结构化输出）。讲清 function calling / tool_choice / temperature / structured outputs / reasoning effort，对比 Vercel AI SDK / LiteLLM / OpenRouter，并给出 evidata 的 BYO-key 多模型落地路线。
- [2026-06-19-mcp-model-context-protocol.md](2026-06-19-mcp-model-context-protocol.md) —— MCP（Model Context Protocol）：工具/数据接入的开放标准（≠ 模型接入，两者正交）。澄清 host/client/server 角色与「大模型在 host 侧、server 通常不碰模型」的关键认知，覆盖原语、传输（stdio + Streamable HTTP，SSE 已弃）、spec 修订史、安全与对 evidata 的意义。
