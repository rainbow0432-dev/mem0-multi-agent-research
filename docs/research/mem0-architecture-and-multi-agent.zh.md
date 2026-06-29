# mem0 架构与多 Agent 记忆研究

> **纯研究参考文档。** 不包含设计或实现计划。涵盖 mem0 的架构、REST API、存储模型，以及多个 Agent 如何共享同一个 mem0 服务。

---

## 1. mem0 是什么

mem0 是面向 LLM 应用的记忆层。它使用 LLM 从对话中提取"事实"，将其存储为向量嵌入（可选地存储为图关系），并按需进行语义检索。

### 记忆生命周期（添加 → 更新 → 删除）

当你调用 `add(messages, user_id=...)` 时，mem0 的 LLM 会：

1. **提取**消息中的事实（通过结构化提示词 / function-calling）
2. **比较**新事实与该用户/Agent 已有记忆
3. **决策**：新增 / 更新已有 / 删除矛盾的 / 无操作
4. **写入**向量存储（如配置了图存储则一并写入）

这不是简单的追加——mem0 会智能管理记忆状态。如果用户说"我现在更喜欢 Python 而不是 Java"，mem0 会更新已有的"偏好 Java"记忆，而不是创建一个矛盾的重复项。

---

## 2. 部署模式

mem0 提供四种部署方式：

| 模式 | 运行方式 | 端口/URL | 认证 | 适用场景 |
|---|---|---|---|---|
| **开源 Python 库** | `from mem0 import Memory` — 嵌入你的进程 | N/A | N/A | 单进程应用，Python 运行时同机部署 |
| **自托管 REST 服务器** | FastAPI Docker 容器 | `:8888` | JWT + X-API-Key (`m0sk_...`) | 多进程/多语言客户端需要 HTTP 记忆服务 |
| **mem0 Platform（云端）** | 托管 SaaS | `api.mem0.ai` | `Authorization: Token <key>` | 零基础设施持久记忆；多租户 |
| **MCP 服务器** | 云端托管 MCP | `mcp.mem0.ai/mcp` | API key (bearer) | 支持 MCP 协议的 Agent（Claude 等） |

**注意**：`mem0ai/mem0-mcp` GitHub 仓库已**归档**。官方 MCP 服务器现在云端托管在 `https://mcp.mem0.ai/mcp`，提供 11 个工具。还有一个可自托管的 `mem0-mcp-server` PyPI 包，但成熟度较低。

### OpenAI 兼容代理

mem0 还提供了一个 OpenAI 兼容代理（`from mem0.proxy.main import Mem0`），暴露 `client.chat.completions.create(messages, model, user_id, agent_id, run_id, ...)` —— 这是 OpenAI 客户端的直接替代，自动处理记忆。

---

## 3. REST API（自托管开源服务器）

**从源码验证**：`mem0ai/mem0` 仓库中 `server/main.py`，SHA `8d6b7c1d671af329dbf43a984fe1b3207ef59fe7`。

### 请求/响应模型

- **`MemoryCreate`**（L179-189）：`messages: list[Message]`, `user_id: str`, `agent_id: str | None`, `run_id: str | None`, `metadata: dict | None`, `infer: bool = True`, `memory_type: str | None`, `prompt: str | None`
- **`SearchRequest`**（L197-207）：`query: str`, `filters: dict | None`, `top_k: int = 5`, `threshold: float | None`, `explain: bool = False`
- **`MemoryUpdate`**（L210-215）：`text: str | None`, `metadata: dict | None`

### 端点

| 方法 | 路径 | 用途 | 认证 |
|---|---|---|---|
| `POST` | `/memories` | 从对话中添加/更新/删除记忆 | X-API-Key 或 JWT |
| `POST` | `/search` | 带过滤器的语义搜索 | X-API-Key 或 JWT |
| `GET` | `/memories` | 列出记忆（按 user_id/agent_id/run_id 过滤） | X-API-Key 或 JWT |
| `GET` | `/memories/{memory_id}` | 获取单条记忆 | X-API-Key 或 JWT |
| `PUT` | `/memories/{memory_id}` | 更新记忆 | X-API-Key 或 JWT |
| `DELETE` | `/memories/{memory_id}` | 删除单条记忆 | X-API-Key 或 JWT |
| `GET` | `/memories/{memory_id}/history` | 变更历史 | X-API-Key 或 JWT |
| `POST` | `/api-keys` | 创建每 Agent API key（管理员） | 仅 JWT |
| `GET` | `/requests` | 请求日志（审计） | 仅 JWT |
| `GET` | `/entities` | 列出 user_id/agent_id/run_id 及计数 | 仅 JWT |

### 认证方式

- **JWT Bearer**：通过 `POST /auth/login` 获取（管理员仪表盘）
- **X-API-Key**：每用户密钥，格式 `m0sk_...`，通过 `POST /api-keys` 用 JWT 创建
- **ADMIN_API_KEY**：旧版基于环境变量的管理员密钥

开源 REST 服务器没有 `/v1/` 前缀（与 mem0 Platform 使用 `/v3/memories/` 不同）。

FastAPI 自动在 `http://mem0:8888/openapi.json` 生成 OpenAPI 规范，在 `/docs` 提供交互式文档。

### mem0 Platform API (api.mem0.ai)

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/v3/memories/add/` | 添加记忆（异步，返回 `event_id`） |
| `POST` | `/v3/memories/search/` | 搜索（实体 ID 必须放在 `filters` 中） |
| `POST` | `/v3/memories/` | 列出记忆（分页） |

认证：`Authorization: Token <MEM0_API_KEY>`。多租户：组织 + 项目。API key 通过 `/v1/ping/` 自动解析组织/项目。

---

## 4. 配置

mem0 通过 JSON 配置文件进行配置，包含 5 个配置块：

### 4.1 向量存储

```json
{
  "vector_store": {
    "provider": "pgvector",
    "config": {
      "dbname": "mem0",
      "collection_name": "memories",
      "embedding_model_dims": 1536
    }
  }
}
```

支持的 provider：`pgvector`、`qdrant`、`milvus`、`weaviate`、`chroma`、`redis`、`valkey`、`opensearch`、`elasticsearch`、`azure_ai_search`、`mongodb`、`faiss`、`cassandra`、`azure_mysql`、`databricks`、`neptune_analytics`。

**`collection_name`** 在初始化时设置一次。这是所有 Agent 共享的单一 collection —— 详见第 6 节。

### 4.2 LLM（用于事实提取）

```json
{
  "llm": {
    "provider": "openai",
    "config": { "model": "gpt-4o-mini", "temperature": 0.1 }
  }
}
```

支持：OpenAI、Anthropic、Ollama、Azure OpenAI、Gemini、Groq、LM Studio、Together AI、Langchain。

### 4.3 嵌入模型

```json
{
  "embedder": {
    "provider": "openai",
    "config": { "model": "text-embedding-3-small" }
  }
}
```

### 4.4 图存储（可选）

```json
{
  "graph_store": {
    "provider": "neo4j",
    "config": { "url": "bolt://neo4j:7687", "username": "neo4j", "password": "..." }
  }
}
```

支持：Neo4j、Memgraph。启用关系感知记忆（例如"用户对 X 有什么看法？"）。

### 4.5 重排器（可选）

```json
{
  "reranker": {
    "provider": "cohere",
    "config": { "model": "rerank-english-v3.0" }
  }
}
```

---

## 5. 记忆作用域维度

mem0 使用四个作用域维度来控制记忆可见性：

| 键 | 是否必需 | 级别 | 用途 |
|---|---|---|---|
| `user_id` | `add` 操作**必需** | 用户 | 终端用户身份。按用户隔离记忆。 |
| `agent_id` | 可选 | Agent | 将记忆范围限定到特定 Agent。**多 Agent 隔离键。** |
| `run_id` | 可选 | 会话 | 追踪单次工作流/会话运行。仅用于追踪——**不**用于过滤。 |
| `app_id` | 可选 | 应用 | 应用级默认值。仅 Platform（mem0 Cloud）支持。 |
| `metadata` | 可选 | 自定义 | 自定义 JSON，用于丰富上下文（标签、来源等） |

### 作用域交互方式

- `agent_id` 在 `user_id` **范围内**过滤。不带 `agent_id` 的搜索返回该用户所有 Agent 的记忆。
- 这同时支持**隔离**（Agent 范围搜索：包含 `agent_id`）和**共享**（跨 Agent 搜索：省略 `agent_id`）。
- `run_id` 仅用于追踪/审计——它**不**过滤搜索结果。
- `app_id` 在 mem0 Platform 上提供租户/应用级作用域。

---

## 6. 多 Agent：共享 Collection 还是独立 Collection

### 明确答案：共享 Collection

**mem0 为所有 Agent 使用一个共享的向量 collection。`agent_id` 是每条记忆记录上的元数据字段，在搜索时用作过滤器——而不是每个 Agent 一个独立 collection。**

这一点从所有支持的向量存储的源码中得到了确认：

#### 源码证据

**Redis**（`mem0/vector_stores/redis.py`）：
```python
DEFAULT_FIELDS = [
    {"name": "memory_id", "type": "tag"},
    {"name": "hash", "type": "tag"},
    {"name": "agent_id", "type": "tag"},     # ← 元数据字段，不是 collection
    {"name": "run_id", "type": "tag"},
    {"name": "user_id", "type": "tag"},
    {"name": "memory", "type": "text"},
    {"name": "metadata", "type": "text"},
]
```

**Weaviate**（`mem0/vector_stores/weaviate.py`）：
```python
# agent_id 是 collection schema 上的一个 Property
wvcc.Property(name="agent_id", data_type=wvcc.DataType.TEXT),

# 在共享 collection 内按属性过滤搜索
if value and key in ["user_id", "agent_id", "run_id"]:
    filter_conditions.append(Filter.by_property(key).equal(value))
```

**OpenSearch**（`mem0/vector_stores/opensearch.py`）：
```python
# agent_id 是索引映射中的 keyword 字段
"metadata": {
    "type": "object",
    "properties": {
        "user_id": {"type": "keyword"},
        "agent_id": {"type": "keyword"},     # ← keyword 字段，不是独立索引
        "run_id": {"type": "keyword"},
    },
}

# 在共享索引上使用 term 过滤搜索
for key in ["user_id", "run_id", "agent_id"]:
    value = filters.get(key)
    if value:
        filter_clauses.append({"term": {f"payload.{key}.keyword": value}})
```

**Azure AI Search**（`mem0/vector_stores/azure_ai_search.py`）：
```python
# agent_id 是索引上可过滤的字段
SimpleField(name="agent_id", type=SearchFieldDataType.String, filterable=True),
```

**Qdrant**（`mem0/vector_stores/qdrant.py`）：
```python
# 在 collection 上创建 payload 索引（元数据索引，不是独立 collection）
common_fields = ["user_id", "agent_id", "run_id", "actor_id"]
for field in common_fields:
    self.client.create_payload_index(
        collection_name=self.collection_name,
        field_name=field,
        ...
    )
```

**Milvus**（`mem0/vector_stores/milvus.py`）：
```python
def _create_filter(self, filters: dict):
    """Prepare filters for efficient query.
    Args:
        filters (dict): filters [user_id, agent_id, run_id]
    """
    # 在共享 collection 上构建过滤表达式
```

**所有向量存储**都在初始化时接受 `collection_name` 参数——这是整个 mem0 实例的一个 collection。你不会为每个 Agent 创建独立 collection。

### 这对多 Agent 部署意味着什么

| 问题 | 答案 |
|---|---|
| Agent 是否共享同一个 collection？ | **是的。** 每个 mem0 实例一个 collection。 |
| 每个 Agent 是否有独立 collection？ | **没有。** `agent_id` 是元数据字段，不是 collection 名。 |
| 我可以为每个 Agent 创建独立 collection 吗？ | 技术上可以，通过运行不同的 mem0 实例并配置不同的 `collection_name`。但这不是 mem0 的设计方式。 |
| 如何实现隔离？ | 通过在搜索查询中使用 `agent_id` 作为元数据过滤器。 |
| Agent 之间可以共享记忆吗？ | 可以——在搜索过滤器中省略 `agent_id` 即可搜索所有 Agent 的记忆。 |
| 有性能影响吗？ | `agent_id` 上的 payload 索引（Qdrant、Redis）确保过滤搜索很快。所有支持的存储都为 `agent_id` 建立了索引以提高过滤效率。 |

### 关键：开源版与 Platform 的存储行为差异

**开源自托管 mem0 和 mem0 Platform (api.mem0.ai) 在多实体写入时有不同的存储行为。** 这是多 Agent 部署最重要的注意事项。

| 方面 | 开源版（自托管） | Platform (api.mem0.ai) |
|--------|---------------------------|------------------------|
| `agent_id` 如何存储 | 与 `user_id` 在**同一条**向量记录上的 payload 元数据 | **每个实体一条独立记录** —— 同时写入 `user_id` + `agent_id` 会创建两条记录 |
| AND 过滤（`user_id` + `agent_id`） | ✅ **有效** —— 返回同一条记录上同时匹配两个条件的记忆 | ❌ **返回空** —— 实体存储为独立记录，没有一条记录同时包含两者 |
| OR 过滤 | ✅ 有效 | ✅ 有效（推荐用于跨范围查询） |
| 查询模式 | `filters={"user_id": "alice", "agent_id": "bot"}`（隐式 AND） | `filters={"OR": [{"user_id": "alice"}, {"agent_id": "bot"}]}`（显式 OR） |

**来源**：[entity-scoped-memory 文档](https://docs.mem0.ai/platform/features/entity-scoped-memory) 和仓库中 `skills/mem0/references/architecture.md`（SHA `8d6b7c1d`）：

> "Platform writes that include both `user_id` and `agent_id` (or other combinations) are persisted as separate records per entity so we can enforce privacy boundaries. Each record carries exactly one primary entity, which is why `{"AND": [{"user_id": ...}, {"agent_id": ...}]}` never returns results."

### ⚠️ 官方博客文章中的矛盾

[多 Agent 博客文章](https://mem0.ai/blog/multi-agent-memory-systems)（2026 年 3 月）展示了使用 Platform 客户端的以下代码：

```python
client = MemoryClient(api_key="your-api-key")
client.add("Customer upgraded to Pro plan", user_id="cust_123", agent_id="billing_agent")
billing_results = client.search("What plan?",
    filters={"AND": [{"user_id": "cust_123"}, {"agent_id": "billing_agent"}]})
```

**这与 Platform 自己的文档相矛盾** —— AND 过滤器在 Platform 上会返回**空结果**，因为实体存储为独立记录。展示的模式只在使用**开源版 mem0** 时有效（`agent_id` 是同一记录上的 payload 元数据）。这看起来是博客文章中的一个文档错误。

### 最安全的跨平台模式

**每次查询一个范围**，在应用代码中合并结果。这在开源版和 Platform 上的行为完全一致：

```python
def get_agent_context(user_id, agent_id, query):
    user_mems = mem0.search(query, filters={"user_id": user_id})
    agent_mems = mem0.search(query, filters={"agent_id": agent_id})
    return merge_and_dedupe(user_mems, agent_mems)
```

### ChatDev 的 OR 过滤模式（Platform 正确用法）

[ChatDev 集成](https://docs.mem0.ai/integrations/chatdev)使用了**正确的** Platform 模式 —— 双范围 OR：

```yaml
memory:
  - name: shared_store
    type: mem0
    config:
      api_key: ${MEM0_API_KEY}
      user_id: alice              # 存储用户偏好
      agent_id: support-bot       # 存储 Agent 学到的上下文
```

> "When both `user_id` and `agent_id` are configured, Mem0 uses an OR filter to search across both scopes in a single query."

---

## 7. 多 Agent mem0 最佳实践

### 7.1 三种架构模式（来自 mem0 官方博客）

来源：[How to Design Multi-Agent Memory Systems for Production](https://mem0.ai/blog/multi-agent-memory-systems)，作者 Taranjeet Singh（mem0 联合创始人兼 CEO），2026-03-03。

| 模式 | 结构 | 最适合 | 权衡 |
|---|---|---|---|
| **集中式** | 单一共享存储，所有 Agent 读写 | 小团队（<5 个 Agent），简单编排 | 强一致性，但超过 N 个 Agent 后成为瓶颈 |
| **分布式** | 每个 Agent 拥有私有记忆，选择性同步 | 大规模，隐私敏感 | 更好的隔离，但同步/一致性困难 |
| **混合式** | 私有 + 共享分层（mem0 实现的模式） | 生产多 Agent 工作流 | 可按用例配置；作用域决策后期难以更改 |

**mem0 通过其四个作用域维度实现混合模式。** 来自博客：

> "Mem0 implements multi-level memory scoping through four dimensions: user_id for personal memories, agent_id for agent-specific context, run_id for session isolation, and app_id for application-level defaults."

### 7.2 推荐的作用域策略

| 场景 | `user_id` | `agent_id` | 行为 |
|---|---|---|---|
| **Agent 隔离**（每个 Agent 有私有记忆） | 按终端用户设置 | 按 Agent 设置 | Agent-A 只看到自己的记忆；Agent-B 只看到自己的 |
| **共享黑板**（所有 Agent 共享记忆） | 按终端用户设置 | 省略（或所有 Agent 相同） | 所有 Agent 看到该用户的所有记忆 |
| **混合**（私有 + 共享） | 按终端用户设置 | 写入时按 Agent 设置；跨 Agent 读取时省略 | Agent 私有写入，但需要时可搜索所有 Agent |
| **跨用户隔离** | 每个用户不同 | N/A | 用户之间永远看不到彼此的记忆 |

### 7.3 混合模式的实践

大多数多 Agent 系统的推荐方式：

1. **每个 Agent 用自己的 `agent_id` 写入** —— 防止上下文污染（计费 Agent 看不到工单）
2. **跨 Agent 搜索时省略 `agent_id`** —— 当 Agent 需要了解其他 Agent 对该用户所知的信息时
3. **`user_id` 始终设置** —— 确保按用户隔离
4. **`run_id` 用于追踪** —— 传入工作流/会话 ID 用于审计追踪（不用于过滤）

来自博客：

> "The agent_id scoping prevents context pollution, so the billing agent never sees raw support tickets and the support agent never sees payment method details. But because both agents share the same user_id, they can both contribute to the same customer's memory when needed."

### 7.4 何时不使用共享记忆

来自博客：

> "Not every multi-agent workflow needs shared memory. If agents are doing one-off, isolated tasks, persistent memory is overhead. Multi-agent memory becomes essential once agents must collaborate on the same evolving state, persist decisions across steps/sessions, or scale in parallel without duplicating and contradicting work."

### 7.5 硬隔离：`mem0 init --agent`（Platform）

对于需要 Agent 之间**完全隔离**的场景（合规、多租户、不同信任级别），mem0 Platform 支持为每个 Agent 配置一个完全隔离的账户：

```bash
mem0 init --agent --agent-caller "your-tool-name" --json
```

来自[单命令博客文章](https://mem0.ai/blog/how-to-enable-memory-in-your-agentic-stack-with-a-single-command)：

> "Each call creates a standalone shadow account with its own isolated (Organization, Project, APIKey) trio. No email address, no usable password. Two calls on two machines produce two fully independent accounts."

这是最强的 Agent 隔离形式——每个 Agent 有独立的组织、项目和 API key。隔离账户之间无法共享记忆。当 Agent 服务于不同租户或法规合规要求硬性数据边界时使用。

### 7.6 没有共享记忆时的常见故障模式

来自博客（引用 Cemri 等人，分析了 7 个框架的 200+ 执行轨迹）：

- **36.9% 的多 Agent 故障**来自 Agent 间的错位（Agent 忽略、重复或矛盾彼此的工作）
- **工作重复**：Agent 独立调用相同的 API，因为它们看不到彼此的结果
- **状态不一致**：面向客户的 Agent 说"订单已发货"，而履约 Agent 显示"处理中"
- **通信开销**：Agent 之间传递完整对话历史（token 成本随对话长度线性增长）
- **级联故障**：一个 Agent 幻觉出一个细节，下游 Agent 将其视为事实

> "Interventions through improved prompting and orchestration yielded only modest accuracy gains of 14 to 15 percentage points. So better base models alone will not fix these problems, because the failures are structural."

### 7.7 关键警告

来自博客：

> "Scoping decisions you make early (which agent_ids map to which memory partitions) can be hard to restructure later as your system grows. So you also need to think carefully about what gets stored as a memory versus what stays ephemeral in the context window, because over-storing creates noise that degrades retrieval quality over time."

**在编写第一个 Agent 之前设计好你的作用域策略。** 需要回答的三个问题：
1. 共享状态存放在哪里？
2. 哪些 Agent 可以看到什么？
3. 当两个 Agent 对某个事实有分歧时怎么办？

### 7.8 上下文污染

来自 AWS Bedrock + Strands + mem0 文章：

> "Mem0 supports complex multi-agent systems with agent-specific memory: Multiple users shared the same agent, causing context contamination."

这突出了始终设置 `user_id` 的重要性——没有它，多个用户的记忆会在共享 collection 中混在一起。

---

## 8. 框架集成模式

### 8.1 LlamaIndex（mem0 官方 cookbook）

来源：[Multi-Agent Collaboration](https://docs.mem0.ai/cookbooks/frameworks/llamaindex-multiagent)

官方 LlamaIndex 示例使用**共享黑板模式**——两个 Agent 共享同一个记忆实例，不设置 `agent_id`：

```python
class MultiAgentLearningSystem:
    def __init__(self, student_id: str):
        # 该学生的记忆上下文——不设置 agent_id！
        self.memory_context = {"user_id": student_id, "app": "learning_assistant"}
        self.memory = Mem0Memory.from_client(context=self.memory_context)

    def _setup_agents(self):
        # TutorAgent 和 PracticeAgent——共享同一个记忆
        self.workflow = AgentWorkflow(
            agents=[self.tutor_agent, self.practice_agent],
            root_agent=self.tutor_agent.name,
        )

    async def start_learning_session(self, topic, student_message):
        # 两个 Agent 共享同一个记忆实例
        response = await self.workflow.run(
            user_msg=request,
            memory=self.memory  # ← 在所有 Agent 间共享
        )
```

文档中的关键引用：

> "multi-agent memory is automatically shared! Shared memory prevents duplication and ensures consistency"

这是最简单的模式：一个 `user_id`，不设 `agent_id`，所有 Agent 看到所有记忆。适用于需要完全共享上下文的协作型 Agent。

### 8.2 CrewAI

CrewAI 通过 `memory_config` 原生集成 mem0。**Crew 中的所有 Agent 共享同一个 `user_id`——不设置每 Agent 的 `agent_id`。** 这是集中式/共享黑板模式：

```python
crew = Crew(
    agents=[agent_a, agent_b],
    tasks=[task_a, task_b],
    memory=True,
    memory_config={
        "provider": "mem0",
        "config": {"user_id": "crew_user_1"},  # 单一共享 user_id，无 agent_id
    }
)
```

来自 [CrewAI 记忆博客文章](https://mem0.ai/blog/crewai-memory-production-setup-with-mem0)：

> "user_id is the only required parameter. It scopes all memory to a specific user, which is the single change that solved my multi-user isolation problem."

**已知问题**：CrewAI 的 `memory_config={"provider": "mem0"}` 有一个 bug，除非你在配置中同时包含 `"user_memory": {}`，否则 `UserMemory` 会被设为 `None`。（来源：[community.crewai.com](https://community.crewai.com/t/mem0-integration-with-crew-ai/4951)）

**另外**：使用不同的 `project_id` 值区分 staging 和 production，防止测试数据泄露到真实用户响应中。

### 8.3 LangGraph

LangGraph 集成是手动的——你需要将 mem0 调用接入图节点：

```python
from mem0 import Memory

memory = Memory.from_config(config)

def memory_node(state):
    # 在 Agent 运行前搜索相关记忆
    results = memory.search(
        query=state["messages"][-1].content,
        filters={"user_id": state["user_id"], "agent_id": "research_agent"},
        top_k=5
    )
    state["prior_context"] = results

def persist_node(state):
    # 在 Agent 响应后持久化
    memory.add(
        messages=state["messages"],
        user_id=state["user_id"],
        agent_id="research_agent"
    )
```

每个 LangGraph Agent 节点用自己的 `agent_id` 写入，可以带或不带 `agent_id` 搜索以实现跨 Agent 读取。

### 8.4 AutoGen

AutoGen 使用**共享 `MemoryClient` 和单一 `USER_ID`**，所有 Agent 共享。聊天 Agent 和管理 Agent 共享同一个记忆范围——不使用 `agent_id`：

```python
USER_ID = "alice"
memory_client = MemoryClient()

# 聊天 Agent 检索记忆
relevant_memories = memory_client.search(question, filters={"user_id": USER_ID})

# 管理 Agent（升级处理）检索相同的记忆——相同范围
relevant_memories = memory_client.search(question, filters={"user_id": USER_ID})
```

来自 [AutoGen 集成文档](https://docs.mem0.ai/integrations/autogen)：

> "AutoGen integration follows the same structure, but uses `ConversableAgent.generate_reply` in the generation step, and supports multi-agent escalation by having the manager agent also retrieve memories before generation."

**关键发现**：没有每 Agent 隔离——聊天 Agent 和管理 Agent 都共享 `user_id="alice"`。管理 Agent 是手动升级路径，不是具有独立记忆范围的自动交接。

### 8.5 AWS Strands Agents SDK

来自 AWS 博客：

> "The AWS Strands Agents SDK includes Mem0 natively, using ElastiCache for vector storage and Neptune Analytics for graph memory."

这是一个托管集成——mem0 内置于 Strands SDK，使用 AWS 原生后端。

### 8.6 框架模式总结

| 框架 | 集成类型 | 作用域模式 | 是否使用 `agent_id`？ | 实际情况 |
|---|---|---|---|---|
| **LlamaIndex** | 官方 cookbook（`llama-index-memory-mem0`） | 共享黑板 | ❌ 未设置 | 所有 Agent 共享一个 `Mem0Memory` 实例，仅使用 `user_id` + `app` |
| **CrewAI** | 原生（`memory_config`） | 共享 `user_id` | ❌ 未设置 | Crew 中所有 Agent 共享一个 `user_id`；无每 Agent `agent_id` |
| **LangGraph** | 手动（节点接线） | `user_id` 范围 | ❌ 通常未设置 | 应用驱动；可在每个节点手动添加 `agent_id` |
| **AutoGen** | 手动封装 | 共享 `user_id` | ❌ 未设置 | 聊天 Agent 和管理 Agent 共享同一个 `USER_ID` 范围 |
| **ChatDev** | 基于配置 | 双范围 OR 过滤 | ✅ 已设置 | 使用 OR 过滤跨 `user_id` + `agent_id` 范围搜索（Platform 正确用法） |
| **AWS Strands** | 原生 SDK | 托管 | 由 SDK 处理 | 使用 ElastiCache + Neptune Analytics |

**关键洞察**：跨框架的主导模式是**共享 `user_id` 且不使用 `agent_id`**——即集中式/共享黑板模式。博客文章中描述的 `agent_id` 作用域是**推荐**模式，但大多数框架集成尚未实现。如果你需要每 Agent 隔离，必须手动接入 `agent_id`。

### 8.7 注意事项与限制

1. **Platform AND 过滤静默失败**：`{"AND": [{"user_id": ...}, {"agent_id": ...}]}` 在 Platform 上返回空。使用 `OR` 或分开查询。（开源版 AND 过滤正常工作。）
2. **通配符 `*` 排除 null**：`{"user_id": "*"}` 只匹配非 null 值，不匹配从未设置 `user_id` 的记忆。
3. **异步处理**：记忆异步处理。在 `add()` 后等待 2-3 秒再搜索，否则记忆可能尚不可检索。
4. **作用域决策难以更改**："Scoping decisions you make early can be hard to restructure later as your system grows."——提前设计作用域策略。
5. **过度存储产生噪声**："Over-storing creates noise that degrades retrieval quality over time."——选择性持久化记忆 vs. 保留在上下文窗口中的临时信息。
6. **CrewAI bug**：`memory_config={"provider": "mem0"}` 有一个 bug，除非同时包含 `"user_memory": {}`，否则 `UserMemory` 被设为 `None`。
7. **上下文污染**：如果多个用户共享同一个 Agent 而没有正确的 `user_id` 作用域，他们的记忆会混在一起。始终设置 `user_id`。

---

## 9. Dify 集成

### 9.1 Dify 编排模型

- **Workflow**：单次运行，START → 节点 → END
- **Chatflow**：对话层工作流，每轮触发，以 Answer 节点结束，有 `conversation_id`
- **Agent 节点**：嵌入 Agent 策略（Function Calling 或 ReAct），赋予 LLM 自主工具调用能力
- **多 Agent**：Dify 没有原生多 Agent 编排器。你在 Workflow/Chatflow 中组合多个 Agent 节点（串行、并行或路由模式）。每条路径最多 50 个节点。
- **原生记忆**：`TokenBufferMemory`——会话级，跨会话重置。Window Size 设置（硬限制：2000 token，500 条消息）。
- **Custom Tool**：粘贴 OpenAPI 规范；Dify 解析为可调用工具。通过 API key 头认证。
- **MCP Tool**：Dify 可以通过 HTTP 传输连接外部 MCP 服务器（Tools → MCP → Add Server）。

### 9.2 mem0 + Dify 的五种集成路径

| 路径 | 工作方式 | 最适合 |
|---|---|---|
| **Dify Custom Tool (OpenAPI)** → mem0 REST | 将 mem0 自动生成的 OpenAPI 规范粘贴到 Dify；筛选为 Agent 安全端点 | 自托管 mem0 REST；每 Agent API key |
| **Dify MCP Tool** → mem0 云端 MCP | 将 Dify 连接到 `https://mcp.mem0.ai/mcp`；11 个工具自动导入 | mem0 Cloud 用户；零插件代码 |
| **Dify HTTP Request 节点** → mem0 REST | 在工作流中直接调用 mem0 REST 端点 | 最大控制，无工具抽象 |
| **Dify 插件**（社区） | 从 Marketplace 安装 `beersoccer/mem0ai`（自托管，12 工具）或 `Feversun/dify-plugin-mem0`（云端，8 工具） | 预构建集成，最少工作量 |
| **mem0 OpenAI 兼容代理** | 将 Dify 的模型 provider 指向 mem0 代理 | 小众——Dify 模型 provider 用于 LLM，不是记忆 |

### 9.3 社区 Dify 插件

| 插件 | 模式 | 工具 | 备注 |
|---|---|---|---|
| `beersoccer/mem0ai` (v0.3.1) | 自托管（在插件容器内运行 mem0） | 12（add, search, get, update, delete, extract_long_term, forget 等） | 异步模式，基于访问日志的遗忘曲线，功能最丰富 |
| `Feversun/dify-plugin-mem0` | 云端（mem0 Platform API v2） | 8 | 更简单，使用托管 mem0 |
| `yevanchen/mem0` | 云端 | 基本添加/检索 | 被 mem0 自己的文档页面引用 |
| `sysam68/mem0_dify_plugin` | 自托管（beersoccer 的 fork） | 与 beersoccer 相同 | Fork |

**注意**：`beersoccer/mem0ai` 插件在 Dify 插件容器**内部**以库的形式运行 mem0——它**不**连接到外部 mem0 REST 服务器。它本身就是 mem0 实例。如果你想要一个被多个服务共享的中央 REST 服务器，请使用 Custom Tool 或 HTTP Request 路径。

### 9.4 mem0 的 `marketplace.json` 不是 Dify 插件

mem0 仓库根目录有一个 `marketplace.json` 文件，但它引用的 `./integrations/mem0-plugin/` 是一个**编程 Agent 插件**（用于 Cursor、Codex、opencode）——**不是** Dify 插件。mem0 不提供官方 Dify 插件。Dify 生态系统完全由社区构建。

---

## 10. Dify 多 Agent 设置工作流（参考）

适用于自托管 mem0 REST 服务器 + 自托管 Dify，多个 Agent 共享一个 mem0 实例：

### 10.1 部署 mem0 REST 服务器

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: mem0
      POSTGRES_USER: mem0
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data

  mem0:
    build:
      context: ./mem0-repo
      dockerfile: server/Dockerfile
    environment:
      - POSTGRES_URL=postgresql://mem0:${POSTGRES_PASSWORD}@postgres:5432/mem0
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ADMIN_API_KEY=${ADMIN_API_KEY}
      - CONFIG_PATH=/app/config.json
    volumes:
      - ./config.json:/app/config.json:ro
    ports: ["8888:8888"]
    depends_on: [postgres]

volumes:
  pgdata:
```

### 10.2 引导每 Agent API key

```bash
# 1. 注册管理员
curl -X POST http://localhost:8888/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@mem0.local", "password": "..."}'

# 2. 登录 → 获取 JWT
JWT=$(curl -s -X POST http://localhost:8888/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@mem0.local", "password": "..."}' | jq -r .access_token)

# 3. 为每个 Agent 创建 API key
for AGENT in agent_a agent_b agent_c; do
  curl -X POST http://localhost:8888/api-keys \
    -H "Authorization: Bearer ${JWT}" \
    -H "Content-Type: application/json" \
    -d "{\"name\": \"${AGENT}\"}"
done
# 每个返回: {"api_key": "m0sk_...", "name": "agent_a"}
```

### 10.3 创建 Dify Custom Tool

1. 获取 OpenAPI 规范：`curl http://mem0:8888/openapi.json -o mem0-openapi.json`
2. 筛选为 Agent 安全端点（移除 `/auth/*`、`/api-keys`、`/configure`、`/reset`、`/requests`、`/entities`）
3. 在 Dify 中：**Tools → Custom → Create Custom Tool** → 粘贴筛选后的 YAML
4. 设置认证：`X-API-Key` 头，每个 Agent 一个工具实例（Agent-A 使用 `m0sk_A`，Agent-B 使用 `m0sk_B`）

### 10.4 接入每 Agent Chatflow

```
START
  → HTTP: POST /search（带 agent_id 过滤器）     [搜索已有记忆]
  → Code: 将搜索结果格式化为上下文字符串
  → Agent 节点（Function Calling，带 mem0 工具）  [LLM 生成响应]
  → HTTP: POST /memories（带 agent_id）           [持久化本轮对话]
  → ANSWER
```

每个 Agent 的 Chatflow：
- 将 `agent_id` 设为常量（如 `"agent_a"`）
- 使用自己的 API key（`m0sk_A`）
- 从终端用户传入 `user_id`（Dify 的 `sys.user_id`）
- 搜索包含 `agent_id` 过滤器用于隔离，或省略用于跨 Agent 共享

### 10.5 Dify 中的多 Agent 作用域模式

| 模式 | 如何接入 |
|---|---|
| **Agent 隔离** | 每个 Chatflow 的搜索 HTTP 节点在 filters 中包含 `"agent_id": "{{agent_id}}"` |
| **共享黑板** | 从搜索过滤器中省略 `agent_id`——所有 Agent 看到该 `user_id` 的所有记忆 |
| **混合（推荐）** | 默认带 `agent_id` 搜索（隔离）；需要跨 Agent 上下文时添加第二个不带 `agent_id` 的搜索节点 |

---

## 11. 参考资料链接

### mem0 源码
- 仓库：https://github.com/mem0ai/mem0
- 服务器源码（SHA `8d6b7c1d`）：`server/main.py`——REST 端点已从源码验证
- 向量存储实现：`mem0/vector_stores/*.py`——`agent_id` 处理已在 8+ 个存储中验证

### mem0 文档
- 主文档：https://docs.mem0.ai
- Dify 集成：https://docs.mem0.ai/integrations/dify
- LlamaIndex 多 Agent cookbook：https://docs.mem0.ai/cookbooks/frameworks/llamaindex-multiagent
- OpenAI 兼容性：https://docs.mem0.ai/open-source/features/openai_compatibility

### mem0 博客
- 多 Agent 记忆系统：https://mem0.ai/blog/multi-agent-memory-systems（Taranjeet Singh，2026-03-03）
- Dify Agent 记忆指南：https://mem0.ai/blog/dify-agent-memory-configuration-guide

### Dify
- Dify 文档：https://docs.dify.ai
- Agent 节点博客：https://dify.ai/blog/dify-agent-node-introduction-when-workflows-learn-autonomous-reasoning
- Dify Marketplace (beersoccer/mem0ai)：https://marketplace.dify.ai/plugin/beersoccer/mem0ai

### mem0 MCP
- 云端 MCP 服务器：https://mcp.mem0.ai/mcp（11 个工具：add_memory, search_memories, get_memories, get_memory, update_memory, delete_memory, delete_all_memories, delete_entities, list_entities, list_events, get_event_status）
- 已归档的自托管 MCP 仓库：https://github.com/mem0ai/mem0-mcp（已归档，被云端版本取代）

### 研究引用
- Cemri 等人：分析了 7 个多 Agent 框架的 200+ 执行轨迹，36.9% 的故障来自 Agent 间错位
- Bazeley（MongoDB 博客）："记忆工程"概念
- Yu 和 Zhao：三层 Agent 记忆模型（I/O、缓存、记忆）
- Rezazadeh 等人：协作记忆论文（分布式记忆与二分图访问控制，90% 准确率，61% 资源使用减少）
- Wegner（1980 年代）：交互式记忆系统（谁知道什么）
