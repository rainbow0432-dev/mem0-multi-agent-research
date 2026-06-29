# mem0 Architecture & Multi-Agent Memory Research

> **Pure research reference.** No design or implementation plan. Covers mem0's architecture, REST API, storage model, and how multiple agents share a single mem0 service.

---

## 1. What is mem0

mem0 is a memory layer for LLM applications. It extracts "facts" from conversations using an LLM, stores them as vector embeddings (and optionally as graph relationships), and retrieves them semantically on demand.

### Memory lifecycle (add → update → delete)

When you call `add(messages, user_id=...)`, mem0's LLM:

1. **Extracts** facts from the messages (structured prompt / function-calling)
2. **Compares** new facts against existing memories for that user/agent
3. **Decides**: ADD new / UPDATE existing / DELETE contradictory / NOOP
4. **Writes** to vector store (+ graph store if configured)

This is not a simple append — mem0 intelligently manages memory state. If a user says "I now prefer Python over Java," mem0 updates the existing "prefers Java" memory rather than creating a contradictory duplicate.

### Internal Component Architecture

mem0 is a modular system with a central orchestrator (`Memory` class) and pluggable component providers. All components use the **Provider pattern** — configurable via JSON, swappable without code changes.

#### Components

| Component | Source directory | Role | Providers (examples) |
|---|---|---|---|
| **Memory** (orchestrator) | `mem0/memory/main.py` | Central hub — orchestrates the entire add/search/update/delete pipeline | N/A (single class) |
| **LLM** | `mem0/llms/` | Extracts facts from conversations; makes A.U.D.N. decisions (Add/Update/Delete/Noop) | OpenAI, Anthropic, Ollama, Azure, Gemini, Groq, DeepSeek, vLLM, LM Studio (18+) |
| **Embedder** | `mem0/embeddings/` | Converts text to vector embeddings for semantic search | OpenAI, HuggingFace, Ollama, Azure, Gemini, FastEmbed, Together (13+) |
| **Vector Store** | `mem0/vector_stores/` | Stores and retrieves vector embeddings with metadata filters | Qdrant, pgvector, Milvus, Weaviate, Redis, Chroma, Pinecone, OpenSearch, MongoDB (23+) |
| **Graph Store** (optional) | `mem0/graphs/` | Models entity relationships as a knowledge graph (Mem0g variant) | Neo4j, Memgraph |
| **Reranker** (optional) | `mem0/reranker/` | Re-ranks search results for better relevance | Cohere, HuggingFace, SentenceTransformer, LLM, ZeroEntropy |
| **History DB** | `mem0/memory/storage.py` | SQLite — tracks every add/update/delete for audit | SQLite (built-in) |
| **Config** | `mem0/configs/base.py` | Defines the config schema for all components | N/A |
| **Utils** | `mem0/utils/factory.py` | Factory pattern — instantiates the right provider from config | N/A |
| **Client** | `mem0/client/main.py` | Platform API client (for mem0 Cloud) | N/A |
| **Proxy** | `mem0/proxy/main.py` | OpenAI-compatible proxy that auto-handles memory | N/A |

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         mem0 Internal Architecture                      │
│                                                                         │
│                    ┌─────────────────┐                                  │
│                    │  Memory Class   │  ← Central orchestrator          │
│                    │  (main.py)      │    Entry point: add(), search()  │
│                    └───────┬─────────┘                                  │
│                            │                                           │
│              ┌─────────────┼─────────────┐                              │
│              │             │             │                              │
│              ▼             ▼             ▼                              │
│     ┌────────────┐ ┌────────────┐ ┌────────────┐                       │
│     │   Config   │ │   Utils    │ │ History DB │                       │
│     │ (base.py)  │ │ (factory)  │ │ (SQLite)   │                       │
│     │            │ │            │ │ audit trail│                       │
│     └────────────┘ └────────────┘ └────────────┘                       │
│                            │                                           │
│    ┌───────────────────────┼───────────────────────┐                   │
│    │                       │                       │                   │
│    ▼                       ▼                       ▼                   │
│ ┌──────────┐      ┌──────────────┐      ┌──────────────────┐          │
│ │   LLM    │      │   Embedder   │      │  Vector Store    │          │
│ │ (llms/)  │      │ (embeddings/ │      │ (vector_stores/) │          │
│ │          │      │              │      │                  │          │
│ │ Extracts │      │ Text → Vector│      │ Stores & searches│          │
│ │ facts,   │      │ embeddings   │      │ vectors +        │          │
│ │ makes    │      │              │      │ metadata         │          │
│ │ A.U.D.N. │      │              │      │                  │          │
│ │ decisions│      │              │      │                  │          │
│ └──────────┘      └──────────────┘      └──────────────────┘          │
│                                                 ▲                      │
│                                                 │                      │
│                          ┌──────────────────────┘                      │
│                          │                                              │
│                   ┌──────────────┐     ┌──────────────┐                │
│                   │ Graph Store  │     │   Reranker   │                │
│                   │  (optional)  │     │  (optional)  │                │
│                   │              │     │              │                │
│                   │ Entity-      │     │ Re-orders    │                │
│                   │ relationship │     │ search       │                │
│                   │ knowledge    │     │ results for  │                │
│                   │ graph        │     │ relevance    │                │
│                   └──────────────┘     └──────────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Provider Pattern

Every pluggable component (LLM, Embedder, Vector Store, Graph Store, Reranker) follows the same pattern:

1. **Base class** (`mem0/<component>/base.py`) defines the interface
2. **Provider implementations** (`mem0/<component>/<provider>.py`) implement the interface
3. **Config** (`mem0/configs/<component>/`) defines provider-specific config schemas
4. **Factory** (`mem0/utils/factory.py`) instantiates the right provider from config

```python
# Config drives everything — no code changes needed to swap providers
config = {
    "vector_store": {"provider": "qdrant", "config": {...}},
    "llm": {"provider": "ollama", "config": {...}},
    "embedder": {"provider": "openai", "config": {...}},
    # ... swap any provider by changing the "provider" string
}
memory = Memory.from_config(config)
```

#### `add()` Flow — Two-Phase Pipeline

The `add()` method (line 648 in `main.py`) runs a two-phase pipeline:

```
Phase 1: EXTRACTION                          Phase 2: A.U.D.N. CYCLE
                                            (per memory candidate)

┌──────────────────────┐                    ┌──────────────────────────┐
│ Input: messages       │                    │ For each candidate:       │
│ (user + assistant)    │                    │                          │
│                       │                    │  ┌─────────────────────┐ │
│ ┌───────────────────┐ │                    │  │ 1. EMBED            │ │
│ │ Context sources:  │ │                    │  │ embedder.embed(     │ │
│ │ • Latest exchange │ │                    │  │   candidate)        │ │
│ │ • Rolling summary │ │                    │  │ → query_vector      │ │
│ │ • Last N messages │ │                    │  └─────────┬───────────┘ │
│ └───────────────────┘ │                    │            │             │
│           │           │                    │            ▼             │
│           ▼           │                    │  ┌─────────────────────┐ │
│ ┌───────────────────┐ │                    │  │ 2. SEARCH           │ │
│ │ LLM.generate_     │ │                    │  │ vector_store.search(│ │
│ │   response()      │ │                    │  │   query_vector,     │ │
│ │                   │ │   ┌──────────────┐ │  │   filters)          │ │
│ │ Prompt:           │ │   │ Memory      │ │  │ → similar_memories  │ │
│ │ MEMORY_DEDUCTION_ │──┼──▶│ Candidates  │─┼──│ (top-k)             │ │
│ │ PROMPT            │ │   │ (list of    │ │  └─────────┬───────────┘ │
│ │                   │ │   │  facts)     │ │            │             │
│ Output: JSON list   │ │   └──────────────┘ │            ▼             │
│ of "memory          │ │                    │  ┌─────────────────────┐ │
│ │ candidates"       │ │                    │  │ 3. A.U.D.N. DECISION│ │
│ └───────────────────┘ │                    │  │ LLM.generate_       │ │
│                       │                    │  │   response()        │ │
└──────────────────────┘                    │  │                     │ │
                                            │  │ Prompt:             │ │
                                            │  │ UPDATE_MEMORY_PROMPT│ │
                                            │  │                     │ │
                                            │  │ Input: candidate +  │ │
                                            │  │ similar_memories    │ │
                                            │  │                     │ │
                                            │  │ Decision: ADD /     │ │
                                            │  │ UPDATE / DELETE /   │ │
                                            │  │ NOOP                │ │
                                            │  └─────────┬───────────┘ │
                                            │            │             │
                                            │            ▼             │
                                            │  ┌─────────────────────┐ │
                                            │  │ 4. EXECUTE          │ │
                                            │  │ vector_store.insert/│ │
                                            │  │   update/delete     │ │
                                            │  │ + history_db.log()  │ │
                                            │  └─────────────────────┘ │
                                            │                          │
                                            │  (Optional: graph_store  │
                                            │   .add entities+edges)   │
                                            │                          │
                                            └──────────────────────────┘
```

Source code references (SHA `8d6b7c1d`):
- `self.llm.generate_response()` at L833 — Phase 1 extraction
- `self.embedding_model.embed()` — vector creation
- `self.vector_store.search()` at L803 — Phase 2 semantic search for similar memories
- `UPDATE_MEMORY_PROMPT` in `mem0/configs/prompts.py` — A.U.D.N. decision prompt

#### `search()` Flow

The `search()` method (line 1238 in `main.py`) runs a simpler pipeline:

```
┌────────────────────────────────────────────────────┐
│  search(query, user_id, agent_id, ...)             │
│                                                    │
│  1. BUILD FILTERS                                  │
│     filters = {user_id, agent_id, run_id}          │
│                                                    │
│  2. EMBED QUERY                                    │
│     embedder.embed(query) → query_vector           │
│                                                    │
│  3. VECTOR SEARCH                                  │
│     vector_store.search(                           │
│       query_vector,                                │
│       filters,                                     │
│       top_k                                        │
│     ) → raw_results                                │
│                                                    │
│  4. RERANK (optional)                              │
│     reranker.rerank(query, raw_results)             │
│     → reranked_results                             │
│                                                    │
│  5. GRAPH SEARCH (optional, Mem0g)                 │
│     graph_store.search(query, filters)             │
│     → related_entities                             │
│                                                    │
│  6. FORMAT & RETURN                                │
│     [{memory, score, metadata, ...}, ...]          │
└────────────────────────────────────────────────────┘
```

#### Why the LLM is central

mem0's key innovation is using the LLM not just for text generation, but as a **decision engine** for database operations. The A.U.D.N. cycle delegates Add/Update/Delete/Noop decisions to the LLM, which understands semantic nuance that would be extremely hard to program with if/else rules. For example, "Actually, I don't like cheese anymore" is understood as a DELETE/UPDATE of the existing "likes cheese" memory, not a new fact.

Source: [VirtusLab analysis](https://virtuslab.com/blog/ai/git-hub-all-stars-2/) and [arxiv paper](https://arxiv.org/html/2504.19413v1) ("Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory").

---

## 2. Deployment Modes

mem0 ships four deployment surfaces:

| Mode | How it runs | Port/URL | Auth | When to use |
|---|---|---|---|---|
| **OSS Python library** | `from mem0 import Memory` — embedded in your process | N/A | N/A | Single-process apps, co-located Python runtime |
| **Self-hosted REST server** | FastAPI Docker container | `:8888` | JWT + X-API-Key (`m0sk_...`) | Multi-process / multi-language clients needing an HTTP memory service |
| **mem0 Platform (Cloud)** | Hosted SaaS | `api.mem0.ai` | `Authorization: Token <key>` | Zero-infra persistent memory; multi-tenant |
| **MCP server** | Cloud-hosted MCP | `mcp.mem0.ai/mcp` | API key (bearer) | Agents that speak MCP (Claude, etc.) |

**Note**: The `mem0ai/mem0-mcp` GitHub repo is **archived**. The official MCP server is now cloud-hosted at `https://mcp.mem0.ai/mcp` with 11 tools. A self-hostable `mem0-mcp-server` PyPI package exists but is less mature.

### OpenAI-compatible proxy

mem0 also ships an OpenAI-compatible proxy (`from mem0.proxy.main import Mem0`) that exposes `client.chat.completions.create(messages, model, user_id, agent_id, run_id, ...)` — a drop-in replacement for the OpenAI client that automatically handles memory.

---

## 3. REST API (Self-Hosted OSS Server)

**Verified from source**: `server/main.py` at SHA `8d6b7c1d671af329dbf43a984fe1b3207ef59fe7` in `mem0ai/mem0`.

### Request/response models

- **`MemoryCreate`** (L179-189): `messages: list[Message]`, `user_id: str`, `agent_id: str | None`, `run_id: str | None`, `metadata: dict | None`, `infer: bool = True`, `memory_type: str | None`, `prompt: str | None`
- **`SearchRequest`** (L197-207): `query: str`, `filters: dict | None`, `top_k: int = 5`, `threshold: float | None`, `explain: bool = False`
- **`MemoryUpdate`** (L210-215): `text: str | None`, `metadata: dict | None`

### Endpoints

| Method | Path | Purpose | Auth |
|---|---|---|---|
| `POST` | `/memories` | Add/update/delete memories from a conversation turn | X-API-Key or JWT |
| `POST` | `/search` | Semantic search with filters | X-API-Key or JWT |
| `GET` | `/memories` | List memories (filter by user_id/agent_id/run_id) | X-API-Key or JWT |
| `GET` | `/memories/{memory_id}` | Get one memory | X-API-Key or JWT |
| `PUT` | `/memories/{memory_id}` | Update a memory | X-API-Key or JWT |
| `DELETE` | `/memories/{memory_id}` | Delete one memory | X-API-Key or JWT |
| `GET` | `/memories/{memory_id}/history` | Change history | X-API-Key or JWT |
| `POST` | `/api-keys` | Create per-agent API key (admin) | JWT only |
| `GET` | `/requests` | Request logs (audit) | JWT only |
| `GET` | `/entities` | List user_id/agent_id/run_id with counts | JWT only |

### Auth modes

- **JWT Bearer**: obtained via `POST /auth/login` (admin dashboard)
- **X-API-Key**: per-user key, format `m0sk_...`, created via `POST /api-keys` with JWT
- **ADMIN_API_KEY**: legacy env-var-based admin key

No `/v1/` prefix for the OSS REST server (unlike mem0 Platform which uses `/v3/memories/`).

FastAPI auto-generates OpenAPI at `http://mem0:8888/openapi.json` and interactive docs at `/docs`.

### mem0 Platform API (api.mem0.ai)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v3/memories/add/` | Add memories (async, returns `event_id`) |
| `POST` | `/v3/memories/search/` | Search (entity IDs MUST be in `filters`) |
| `POST` | `/v3/memories/` | List memories (paginated) |

Auth: `Authorization: Token <MEM0_API_KEY>`. Multi-tenancy: Organizations + Projects. API key auto-resolves org/project via `/v1/ping/`.

---

## 4. Configuration

mem0 is configured via a JSON config with 5 blocks:

### 4.1 Vector store

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

Supported providers: `pgvector`, `qdrant`, `milvus`, `weaviate`, `chroma`, `redis`, `valkey`, `opensearch`, `elasticsearch`, `azure_ai_search`, `mongodb`, `faiss`, `cassandra`, `azure_mysql`, `databricks`, `neptune_analytics`.

**`collection_name`** is set once at initialization. This is a single collection shared by all agents — see §6 for details.

### 4.2 LLM (for fact extraction)

```json
{
  "llm": {
    "provider": "openai",
    "config": { "model": "gpt-4o-mini", "temperature": 0.1 }
  }
}
```

Supported: OpenAI, Anthropic, Ollama, Azure OpenAI, Gemini, Groq, LM Studio, Together AI, Langchain.

### 4.3 Embedder

```json
{
  "embedder": {
    "provider": "openai",
    "config": { "model": "text-embedding-3-small" }
  }
}
```

### 4.4 Graph store (optional)

```json
{
  "graph_store": {
    "provider": "neo4j",
    "config": { "url": "bolt://neo4j:7687", "username": "neo4j", "password": "..." }
  }
}
```

Supported: Neo4j, Memgraph. Enables relationship-aware memory (e.g., "What does the user think about X?").

### 4.5 Reranker (optional)

```json
{
  "reranker": {
    "provider": "cohere",
    "config": { "model": "rerank-english-v3.0" }
  }
}
```

---

## 5. Memory Scoping Dimensions

mem0 uses four scoping dimensions to control memory visibility:

| Key | Required? | Level | Purpose |
|---|---|---|---|
| `user_id` | **Required** for `add` | User | End-user identity. Isolates memories per user. |
| `agent_id` | Optional | Agent | Scopes memories to a specific agent. **The multi-agent isolation key.** |
| `run_id` | Optional | Session | Traces a single workflow/session run. Tracing only — NOT used for filtering. |
| `app_id` | Optional | Application | Application-level defaults. Platform-only (mem0 Cloud). |
| `metadata` | Optional | Custom | Custom JSON for rich context (tags, source, etc.) |

### How scoping interacts

- `agent_id` filters **within** a `user_id`. Searches without `agent_id` return memories from all agents for that user.
- This enables both **isolation** (agent-scoped search: include `agent_id`) and **sharing** (cross-agent search: omit `agent_id`).
- `run_id` is for tracing/auditing only — it does NOT filter search results.
- `app_id` provides a tenant/application-level scope on mem0 Platform.

---

## 6. Multi-Agent: Shared Collection vs Separate Collection

### The definitive answer: SHARED COLLECTION

**mem0 uses a single shared vector collection for all agents. `agent_id` is a metadata field on each memory record, used as a filter during search — NOT a separate collection per agent.**

This is confirmed from the source code across all supported vector stores:

#### Evidence from source code

**Redis** (`mem0/vector_stores/redis.py`):
```python
DEFAULT_FIELDS = [
    {"name": "memory_id", "type": "tag"},
    {"name": "hash", "type": "tag"},
    {"name": "agent_id", "type": "tag"},     # ← metadata field, not collection
    {"name": "run_id", "type": "tag"},
    {"name": "user_id", "type": "tag"},
    {"name": "memory", "type": "text"},
    {"name": "metadata", "type": "text"},
]
```

**Weaviate** (`mem0/vector_stores/weaviate.py`):
```python
# agent_id is a Property on the collection schema
wvcc.Property(name="agent_id", data_type=wvcc.DataType.TEXT),

# Search filters on the property within the shared collection
if value and key in ["user_id", "agent_id", "run_id"]:
    filter_conditions.append(Filter.by_property(key).equal(value))
```

**OpenSearch** (`mem0/vector_stores/opensearch.py`):
```python
# agent_id is a keyword field in the index mapping
"metadata": {
    "type": "object",
    "properties": {
        "user_id": {"type": "keyword"},
        "agent_id": {"type": "keyword"},     # ← keyword field, not separate index
        "run_id": {"type": "keyword"},
    },
}

# Search uses a term filter on the shared index
for key in ["user_id", "run_id", "agent_id"]:
    value = filters.get(key)
    if value:
        filter_clauses.append({"term": {f"payload.{key}.keyword": value}})
```

**Azure AI Search** (`mem0/vector_stores/azure_ai_search.py`):
```python
# agent_id is a filterable field on the index
SimpleField(name="agent_id", type=SearchFieldDataType.String, filterable=True),
```

**Qdrant** (`mem0/vector_stores/qdrant.py`):
```python
# Creates payload indexes on the collection (metadata indexes, not separate collections)
common_fields = ["user_id", "agent_id", "run_id", "actor_id"]
for field in common_fields:
    self.client.create_payload_index(
        collection_name=self.collection_name,
        field_name=field,
        ...
    )
```

**Milvus** (`mem0/vector_stores/milvus.py`):
```python
def _create_filter(self, filters: dict):
    """Prepare filters for efficient query.
    Args:
        filters (dict): filters [user_id, agent_id, run_id]
    """
    # Builds filter expressions on the shared collection
```

**All vector stores** take a `collection_name` parameter at initialization — this is a single collection for the entire mem0 instance. You do not create separate collections per agent.

### What this means for multi-agent deployments

| Question | Answer |
|---|---|
| Do agents share the same collection? | **Yes.** One collection per mem0 instance. |
| Does each agent get its own collection? | **No.** `agent_id` is a metadata field, not a collection name. |
| Can I create separate collections per agent? | Technically yes, by running separate mem0 instances with different `collection_name` configs. But this is NOT how mem0 is designed to work. |
| How is isolation achieved? | Via `agent_id` as a metadata filter on search queries. |
| Can agents share memories? | Yes — omit `agent_id` in the search filter to search across all agents. |
| Is there a performance impact? | Payload indexes on `agent_id` (Qdrant, Redis) ensure filtered searches are fast. All supported stores index `agent_id` for efficient filtering. |

### CRITICAL: Open-Source vs Platform Storage Behavior

**The open-source self-hosted mem0 and the mem0 Platform (api.mem0.ai) have DIFFERENT storage behaviors for multi-entity writes.** This is the most important gotcha for multi-agent deployments.

| Aspect | Open-Source (self-hosted) | Platform (api.mem0.ai) |
|--------|---------------------------|------------------------|
| How `agent_id` is stored | Payload metadata on the **same** vector record as `user_id` | **Separate record per entity** — a write with both `user_id` + `agent_id` creates two records |
| AND filter (`user_id` + `agent_id`) | ✅ **Works** — returns memories matching both on the same record | ❌ **Returns empty** — entities are stored as separate records, so no single record has both |
| OR filter | ✅ Works | ✅ Works (recommended for cross-scope queries) |
| Query pattern | `filters={"user_id": "alice", "agent_id": "bot"}` (AND, implicit) | `filters={"OR": [{"user_id": "alice"}, {"agent_id": "bot"}]}` (explicit OR) |

**Source**: [entity-scoped-memory docs](https://docs.mem0.ai/platform/features/entity-scoped-memory) and `skills/mem0/references/architecture.md` in the repo (SHA `8d6b7c1d`):

> "Platform writes that include both `user_id` and `agent_id` (or other combinations) are persisted as separate records per entity so we can enforce privacy boundaries. Each record carries exactly one primary entity, which is why `{"AND": [{"user_id": ...}, {"agent_id": ...}]}` never returns results."

### ⚠️ Contradiction in the Official Blog Post

The [multi-agent blog post](https://mem0.ai/blog/multi-agent-memory-systems) (March 2026) shows this code using the Platform client:

```python
client = MemoryClient(api_key="your-api-key")
client.add("Customer upgraded to Pro plan", user_id="cust_123", agent_id="billing_agent")
billing_results = client.search("What plan?",
    filters={"AND": [{"user_id": "cust_123"}, {"agent_id": "billing_agent"}]})
```

**This contradicts the Platform's own documentation** — the AND filter would return **empty results** on the Platform because entities are stored as separate records. The pattern shown would only work on **open-source mem0** where `agent_id` is payload metadata on the same record. This appears to be a documentation error in the blog post.

### The Safest Cross-Platform Pattern

Query **one scope at a time** and merge results in your application code. This works identically on both open-source and Platform:

```python
def get_agent_context(user_id, agent_id, query):
    user_mems = mem0.search(query, filters={"user_id": user_id})
    agent_mems = mem0.search(query, filters={"agent_id": agent_id})
    return merge_and_dedupe(user_mems, agent_mems)
```

### ChatDev's OR-Filter Pattern (Platform-Correct)

The [ChatDev integration](https://docs.mem0.ai/integrations/chatdev) uses the **correct** Platform pattern — dual-scope with OR:

```yaml
memory:
  - name: shared_store
    type: mem0
    config:
      api_key: ${MEM0_API_KEY}
      user_id: alice              # stores user preferences
      agent_id: support-bot       # stores agent-learned context
```

> "When both `user_id` and `agent_id` are configured, Mem0 uses an OR filter to search across both scopes in a single query."

---

## 7. Best Practices for Multi-Agent mem0

### 7.1 Three architecture patterns (from mem0's official blog)

Source: [How to Design Multi-Agent Memory Systems for Production](https://mem0.ai/blog/multi-agent-memory-systems) by Taranjeet Singh (mem0 Co-Founder & CEO), 2026-03-03.

| Pattern | Structure | Best for | Trade-off |
|---|---|---|---|
| **Centralized** | Single shared store, all agents R/W | Small teams (<5 agents), simple orchestration | Strong consistency, but bottleneck past N agents |
| **Distributed** | Each agent owns private memory, selective sync | Large-scale, privacy-sensitive | Better isolation, but sync/consistency is painful |
| **Hybrid** | Private + shared tiers (what mem0 implements) | Production multi-agent workflows | Configurable per use case; scoping decisions are hard to change later |

**mem0 implements the hybrid pattern** through its four scoping dimensions. From the blog:

> "Mem0 implements multi-level memory scoping through four dimensions: user_id for personal memories, agent_id for agent-specific context, run_id for session isolation, and app_id for application-level defaults."

### 7.2 Recommended scoping strategies

| Scenario | `user_id` | `agent_id` | Behavior |
|---|---|---|---|
| **Agent isolation** (each agent has private memory) | Set per end-user | Set per agent | Agent-A only sees its own memories; Agent-B only sees its own |
| **Shared blackboard** (all agents share memory) | Set per end-user | Omit (or same for all agents) | All agents see all memories for that user |
| **Hybrid** (private + shared) | Set per end-user | Set per agent for writes; omit for cross-agent reads | Agents write privately, but can search across all agents when needed |
| **Cross-user isolation** | Different per user | N/A | Users never see each other's memories |

### 7.3 The hybrid pattern in practice

The recommended approach for most multi-agent systems:

1. **Each agent writes with its own `agent_id`** — prevents context pollution (billing agent doesn't see support tickets)
2. **Cross-agent search omits `agent_id`** — when an agent needs to see what other agents know about the user
3. **`user_id` is always set** — ensures per-user isolation
4. **`run_id` for tracing** — pass the workflow/session ID for audit trails (not filtering)

From the blog:

> "The agent_id scoping prevents context pollution, so the billing agent never sees raw support tickets and the support agent never sees payment method details. But because both agents share the same user_id, they can both contribute to the same customer's memory when needed."

### 7.4 When NOT to use shared memory

From the blog:

> "Not every multi-agent workflow needs shared memory. If agents are doing one-off, isolated tasks, persistent memory is overhead. Multi-agent memory becomes essential once agents must collaborate on the same evolving state, persist decisions across steps/sessions, or scale in parallel without duplicating and contradicting work."

### 7.5 Hard isolation: `mem0 init --agent` (Platform)

For cases requiring **complete isolation** between agents (compliance, multi-tenant, different trust levels), the mem0 Platform supports provisioning a fully isolated account per agent:

```bash
mem0 init --agent --agent-caller "your-tool-name" --json
```

From the [single-command blog post](https://mem0.ai/blog/how-to-enable-memory-in-your-agentic-stack-with-a-single-command):

> "Each call creates a standalone shadow account with its own isolated (Organization, Project, APIKey) trio. No email address, no usable password. Two calls on two machines produce two fully independent accounts."

This is the strongest form of agent isolation — separate org, project, and API key per agent. No memory sharing is possible between isolated accounts. Use this when agents serve different tenants or when regulatory compliance requires hard data boundaries.

### 7.6 Common failure modes without shared memory

From the blog (citing Cemri et al., who analyzed 200+ execution traces across 7 frameworks):

- **36.9% of multi-agent failures** come from inter-agent misalignment (agents ignoring, duplicating, or contradicting each other's work)
- **Work duplication**: agents independently call the same APIs because they can't see each other's results
- **Inconsistent state**: customer-facing agent says "order shipped" while fulfillment agent shows "processing"
- **Communication overhead**: agents pass full conversation histories to each other (token costs scale linearly with conversation length)
- **Cascade failures**: one agent hallucinates, downstream agents treat it as ground truth

> "Interventions through improved prompting and orchestration yielded only modest accuracy gains of 14 to 15 percentage points. So better base models alone will not fix these problems, because the failures are structural."

### 7.6 Key warnings

From the blog:

> "Scoping decisions you make early (which agent_ids map to which memory partitions) can be hard to restructure later as your system grows. So you also need to think carefully about what gets stored as a memory versus what stays ephemeral in the context window, because over-storing creates noise that degrades retrieval quality over time."

**Design your scoping strategy before writing your first agent.** The three questions to answer:
1. Where does the shared state live?
2. Which agents can see what?
3. What happens when two agents disagree about a fact?

### 7.7 Context contamination

From the AWS Bedrock + Strands + mem0 article:

> "Mem0 supports complex multi-agent systems with agent-specific memory: Multiple users shared the same agent, causing context contamination."

This highlights the importance of always setting `user_id` — without it, multiple users' memories mix together in the shared collection.

---

## 8. Framework Integration Patterns

### 8.1 LlamaIndex (official mem0 cookbook)

Source: [Multi-Agent Collaboration](https://docs.mem0.ai/cookbooks/frameworks/llamaindex-multiagent)

The official LlamaIndex example uses the **shared blackboard pattern** — both agents share the same memory instance with no `agent_id`:

```python
class MultiAgentLearningSystem:
    def __init__(self, student_id: str):
        # Memory context for this student — no agent_id set!
        self.memory_context = {"user_id": student_id, "app": "learning_assistant"}
        self.memory = Mem0Memory.from_client(context=self.memory_context)

    def _setup_agents(self):
        # TutorAgent and PracticeAgent — both share the same memory
        self.workflow = AgentWorkflow(
            agents=[self.tutor_agent, self.practice_agent],
            root_agent=self.tutor_agent.name,
        )

    async def start_learning_session(self, topic, student_message):
        # Both agents share the same memory instance
        response = await self.workflow.run(
            user_msg=request,
            memory=self.memory  # ← shared across all agents
        )
```

Key quote from the docs:

> "multi-agent memory is automatically shared! Shared memory prevents duplication and ensures consistency"

This is the simplest pattern: one `user_id`, no `agent_id`, all agents see all memories. Works well for collaborative agents that need full shared context.

### 8.2 CrewAI

CrewAI has native mem0 integration via `memory_config`. **All agents in a crew share the same `user_id` — no per-agent `agent_id` is set.** This is the centralized/shared-blackboard pattern:

```python
crew = Crew(
    agents=[agent_a, agent_b],
    tasks=[task_a, task_b],
    memory=True,
    memory_config={
        "provider": "mem0",
        "config": {"user_id": "crew_user_1"},  # single shared user_id, no agent_id
    }
)
```

From the [CrewAI memory blog post](https://mem0.ai/blog/crewai-memory-production-setup-with-mem0):

> "user_id is the only required parameter. It scopes all memory to a specific user, which is the single change that solved my multi-user isolation problem."

**Known gotcha**: The CrewAI `memory_config={"provider": "mem0"}` has a bug where `UserMemory` is set to `None` unless you also include `"user_memory": {}` in the config. (Source: [community.crewai.com](https://community.crewai.com/t/mem0-integration-with-crew-ai/4951))

**Also**: Use separate `project_id` values for staging and production to prevent test data from leaking into real user responses.

### 8.3 LangGraph

LangGraph integration is manual — you wire mem0 calls into graph nodes:

```python
from mem0 import Memory

memory = Memory.from_config(config)

def memory_node(state):
    # Search for relevant memories before the agent runs
    results = memory.search(
        query=state["messages"][-1].content,
        filters={"user_id": state["user_id"], "agent_id": "research_agent"},
        top_k=5
    )
    state["prior_context"] = results

def persist_node(state):
    # Persist after the agent responds
    memory.add(
        messages=state["messages"],
        user_id=state["user_id"],
        agent_id="research_agent"
    )
```

Each LangGraph agent node uses its own `agent_id` for writes, and can search with or without `agent_id` for cross-agent reads.

### 8.4 AutoGen

AutoGen uses a **shared `MemoryClient` with a single `USER_ID`** across all agents. Both the chatbot agent and the manager agent share the same memory scope — no `agent_id` is used:

```python
USER_ID = "alice"
memory_client = MemoryClient()

# Chatbot agent retrieves memories
relevant_memories = memory_client.search(question, filters={"user_id": USER_ID})

# Manager agent (escalation) retrieves the SAME memories — same scope
relevant_memories = memory_client.search(question, filters={"user_id": USER_ID})
```

From the [AutoGen integration docs](https://docs.mem0.ai/integrations/autogen):

> "AutoGen integration follows the same structure, but uses `ConversableAgent.generate_reply` in the generation step, and supports multi-agent escalation by having the manager agent also retrieve memories before generation."

**Key finding**: No per-agent isolation — both chatbot and manager share `user_id="alice"`. The manager agent is a manual escalation path, not an automatic handoff with separate memory scope.

### 8.5 AWS Strands Agents SDK

From the AWS blog:

> "The AWS Strands Agents SDK includes Mem0 natively, using ElastiCache for vector storage and Neptune Analytics for graph memory."

This is a managed integration — mem0 is built into the Strands SDK with AWS-native backends.

### 8.6 Pattern summary across frameworks

| Framework | Integration type | Scoping pattern | `agent_id` used? | Reality |
|---|---|---|---|---|
| **LlamaIndex** | Official cookbook (`llama-index-memory-mem0`) | Shared blackboard | ❌ Not set | All agents share one `Mem0Memory` instance with `user_id` + `app` only |
| **CrewAI** | Native (`memory_config`) | Shared `user_id` | ❌ Not set | All agents in a crew share one `user_id`; no per-agent `agent_id` |
| **LangGraph** | Manual (node wiring) | `user_id` scoped | ❌ Not set (typically) | Application-driven; `agent_id` can be added manually per node |
| **AutoGen** | Manual wrapper | Shared `user_id` | ❌ Not set | Both chatbot and manager agents share the same `USER_ID` scope |
| **ChatDev** | Config-based | Dual-scope OR filter | ✅ Set | Uses OR filter to search across `user_id` + `agent_id` scopes (Platform-correct) |
| **AWS Strands** | Native SDK | Managed | Handled by SDK | Uses ElastiCache + Neptune Analytics |

**Key insight**: The dominant pattern across frameworks is **shared `user_id` with no `agent_id`** — the centralized/shared-blackboard pattern. The `agent_id` scoping described in the blog post is the **recommended** pattern, but most framework integrations don't implement it yet. If you want per-agent isolation, you must wire `agent_id` manually.

### 8.7 Gotchas and Limitations

1. **Platform AND-filter silent failure**: `{"AND": [{"user_id": ...}, {"agent_id": ...}]}` returns empty on Platform. Use `OR` or separate queries. (Open-source works fine with AND.)
2. **Wildcard `*` excludes null**: `{"user_id": "*"}` matches only non-null values, not memories where `user_id` was never set.
3. **Async processing**: Memories process asynchronously. Wait 2-3s after `add()` before searching, or the memory may not be retrievable yet.
4. **Scoping decisions are hard to change**: "Scoping decisions you make early (which agent_ids map to which memory partitions) can be hard to restructure later as your system grows." — design your scoping strategy upfront.
5. **Over-storing creates noise**: "Over-storing creates noise that degrades retrieval quality over time." — be selective about what gets persisted as memory vs. what stays ephemeral.
6. **CrewAI bug**: `memory_config={"provider": "mem0"}` has a bug where `UserMemory` is set to `None` unless you also include `"user_memory": {}` in the config.
7. **Context contamination**: If multiple users share the same agent without proper `user_id` scoping, their memories mix together. Always set `user_id`.

---

## 9. Dify Integration

### 9.1 Dify orchestration model

- **Workflow**: single run, START → nodes → END
- **Chatflow**: conversation-layer workflow, triggered each turn, ends with Answer node, has `conversation_id`
- **Agent Node**: embeds an Agent Strategy (Function Calling or ReAct), gives LLM autonomous tool-calling
- **Multi-agent**: Dify has no native multi-agent orchestrator. You compose multiple Agent nodes in a Workflow/Chatflow (serial, parallel, or router pattern). Up to 50 nodes per path.
- **Native memory**: `TokenBufferMemory` — session-scoped, resets across sessions. Window Size setting (hard limits: 2000 tokens, 500 messages).
- **Custom Tool**: paste an OpenAPI spec; Dify parses it into callable tools. Auth via API key header.
- **MCP Tool**: Dify can connect to external MCP servers over HTTP transport (Tools → MCP → Add Server).

### 9.2 Five integration paths for mem0 + Dify

| Path | How it works | Best for |
|---|---|---|
| **Dify Custom Tool (OpenAPI)** → mem0 REST | Paste mem0's auto-generated OpenAPI spec into Dify; curate to agent-safe endpoints | Self-hosted mem0 REST; per-agent API keys |
| **Dify MCP Tool** → mem0 cloud MCP | Connect Dify to `https://mcp.mem0.ai/mcp`; 11 tools auto-imported | mem0 Cloud users; zero plugin code |
| **Dify HTTP Request node** → mem0 REST | Call mem0 REST endpoints directly in the workflow | Maximum control, no tool abstraction |
| **Dify Plugin** (community) | Install `beersoccer/mem0ai` (self-hosted, 12 tools) or `Feversun/dify-plugin-mem0` (cloud, 8 tools) from Marketplace | Pre-built integration, least effort |
| **mem0 OpenAI-compat proxy** | Point Dify's model provider at mem0's proxy | Niche — Dify model providers are for LLMs, not memory |

### 9.3 Community Dify plugins

| Plugin | Mode | Tools | Notes |
|---|---|---|---|
| `beersoccer/mem0ai` (v0.3.1) | Self-hosted (runs mem0 inside plugin container) | 12 (add, search, get, update, delete, extract_long_term, forget, etc.) | Async mode, access-log-driven forgetting, most feature-rich |
| `Feversun/dify-plugin-mem0` | Cloud (mem0 Platform API v2) | 8 | Simpler, uses managed mem0 |
| `yevanchen/mem0` | Cloud | Basic add/retrieve | Referenced by mem0's own docs page |
| `sysam68/mem0_dify_plugin` | Self-hosted (fork of beersoccer) | Same as beersoccer | Fork |

**Note**: The `beersoccer/mem0ai` plugin runs mem0 as a library INSIDE the Dify plugin container — it does NOT connect to an external mem0 REST server. It IS the mem0 instance. If you want a central REST server shared by multiple services, use the Custom Tool or HTTP Request path instead.

### 9.4 mem0's `marketplace.json` is NOT a Dify plugin

The mem0 repo root has a `marketplace.json` file, but it references `./integrations/mem0-plugin/` which is a **coding-agent plugin** (for Cursor, Codex, opencode) — NOT a Dify plugin. mem0 ships no official Dify plugin. The Dify ecosystem is entirely community-built.

---

## 10. Dify Multi-Agent Setup Workflow (Reference)

For a self-hosted mem0 REST server + self-hosted Dify with multiple agents sharing one mem0 instance:

### 10.1 Deploy mem0 REST server

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

### 10.2 Bootstrap per-agent API keys

```bash
# 1. Register admin
curl -X POST http://localhost:8888/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@mem0.local", "password": "..."}'

# 2. Login → get JWT
JWT=$(curl -s -X POST http://localhost:8888/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@mem0.local", "password": "..."}' | jq -r .access_token)

# 3. Create one API key per agent
for AGENT in agent_a agent_b agent_c; do
  curl -X POST http://localhost:8888/api-keys \
    -H "Authorization: Bearer ${JWT}" \
    -H "Content-Type: application/json" \
    -d "{\"name\": \"${AGENT}\"}"
done
# Each returns: {"api_key": "m0sk_...", "name": "agent_a"}
```

### 10.3 Create Dify Custom Tool

1. Fetch OpenAPI spec: `curl http://mem0:8888/openapi.json -o mem0-openapi.json`
2. Curate to agent-safe endpoints (remove `/auth/*`, `/api-keys`, `/configure`, `/reset`, `/requests`, `/entities`)
3. In Dify: **Tools → Custom → Create Custom Tool** → paste curated YAML
4. Set auth: `X-API-Key` header, one tool instance per agent (Agent-A uses `m0sk_A`, Agent-B uses `m0sk_B`)

### 10.4 Wire per-agent Chatflow

```
START
  → HTTP: POST /search (with agent_id filter)     [search prior memories]
  → Code: format search results into context string
  → Agent Node (Function Calling, with mem0 tools) [LLM generates response]
  → HTTP: POST /memories (with agent_id)           [persist this turn]
  → ANSWER
```

Each agent's Chatflow:
- Sets `agent_id` as a constant (e.g., `"agent_a"`)
- Uses its own API key (`m0sk_A`)
- Passes `user_id` from the end-user (Dify's `sys.user_id`)
- Search includes `agent_id` filter for isolation, or omits it for cross-agent sharing

### 10.5 Multi-agent scoping patterns in Dify

| Pattern | How to wire it |
|---|---|
| **Agent isolation** | Each Chatflow's search HTTP node includes `"agent_id": "{{agent_id}}"` in the filters |
| **Shared blackboard** | Omit `agent_id` from the search filters — all agents see all memories for that `user_id` |
| **Hybrid (recommended)** | Search WITH `agent_id` by default (isolation); add a second search node WITHOUT `agent_id` when cross-agent context is needed |

---

## 11. References

### mem0 source code
- Repository: https://github.com/mem0ai/mem0
- Server source (SHA `8d6b7c1d`): `server/main.py` — REST endpoints verified from source
- Vector store implementations: `mem0/vector_stores/*.py` — `agent_id` handling verified across 8+ stores

### mem0 documentation
- Main docs: https://docs.mem0.ai
- Dify integration: https://docs.mem0.ai/integrations/dify
- LlamaIndex multi-agent cookbook: https://docs.mem0.ai/cookbooks/frameworks/llamaindex-multiagent
- OpenAI compatibility: https://docs.mem0.ai/open-source/features/openai_compatibility

### mem0 blog
- Multi-agent memory systems: https://mem0.ai/blog/multi-agent-memory-systems (Taranjeet Singh, 2026-03-03)
- Dify agent memory guide: https://mem0.ai/blog/dify-agent-memory-configuration-guide

### Dify
- Dify docs: https://docs.dify.ai
- Agent Node blog: https://dify.ai/blog/dify-agent-node-introduction-when-workflows-learn-autonomous-reasoning
- Dify Marketplace (beersoccer/mem0ai): https://marketplace.dify.ai/plugin/beersoccer/mem0ai

### mem0 MCP
- Cloud MCP server: https://mcp.mem0.ai/mcp (11 tools: add_memory, search_memories, get_memories, get_memory, update_memory, delete_memory, delete_all_memories, delete_entities, list_entities, list_events, get_event_status)
- Archived self-hosted MCP repo: https://github.com/mem0ai/mem0-mcp (archived, replaced by cloud)

### Research citations
- Cemri et al.: analysis of 200+ execution traces across 7 multi-agent frameworks, 36.9% inter-agent misalignment failure rate
- Bazeley (MongoDB blog): "memory engineering" concept
- Yu and Zhao: three-layer agent memory model (I/O, cache, memory)
- Rezazadeh et al.: Collaborative Memory paper (distributed memory with bipartite access graphs, 90% accuracy at 61% reduced resource usage)
- Wegner (1980s): transactive memory systems (who knows what)
