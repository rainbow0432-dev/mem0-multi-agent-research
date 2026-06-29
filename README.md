# mem0 Architecture & Multi-Agent Memory Research

Pure research on mem0's architecture, REST API, storage model, and how multiple agents share a single mem0 service. Also covers Dify integration patterns.

## Documents

| File | Language | Description |
|---|---|---|
| [docs/research/mem0-architecture-and-multi-agent.md](docs/research/mem0-architecture-and-multi-agent.md) | English | Full research document (731 lines) |
| [docs/research/mem0-architecture-and-multi-agent.zh.md](docs/research/mem0-architecture-and-multi-agent.zh.md) | 中文 | Chinese translation |

## Key Findings

- **mem0 uses a single shared vector collection** for all agents. `agent_id` is a metadata field/filter, NOT a separate collection per agent. Verified from source code across 8+ vector store implementations.

- **Open-source vs Platform storage difference**: The open-source self-hosted mem0 stores `agent_id` as payload metadata on the same record (AND filtering works). The mem0 Platform (api.mem0.ai) persists multi-entity writes as separate records per entity (AND filtering returns empty — use OR instead).

- **Framework integrations don't use `agent_id`**: Despite the blog post recommending `agent_id` for per-agent scoping, the actual CrewAI, AutoGen, LlamaIndex, and LangGraph integrations all use a shared `user_id` with no `agent_id`.

- **Recommended pattern**: Hybrid — shared collection with `agent_id` scoping. Each agent writes with its own `agent_id`; cross-agent search omits `agent_id`.

## Topics Covered

1. mem0 architecture overview
2. Deployment modes (Python lib, REST server, Platform, MCP)
3. REST API (verified from source, SHA `8d6b7c1d`)
4. Configuration (vector store, LLM, embedder, graph store, reranker)
5. Memory scoping dimensions (user_id, agent_id, run_id, app_id)
6. Shared collection vs separate collection (with source code evidence)
7. Open-source vs Platform storage behavior (critical AND vs OR filter difference)
8. Best practices for multi-agent (3 architecture patterns, gotchas)
9. Framework integration patterns (LlamaIndex, CrewAI, LangGraph, AutoGen, ChatDev, Strands)
10. Dify integration (5 paths, 4 community plugins)
11. Setup workflow reference
