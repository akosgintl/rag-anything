# 01 — Architecture Overview

## 1. What “agentic RAG” means in this design

Classic RAG is usually a fixed pipeline:

```text
user question -> embed question -> top-k retrieval -> prompt with chunks -> answer
```

Agentic RAG makes retrieval and answer generation part of a controlled decision loop:

```text
user task -> understand intent -> plan -> choose tools -> retrieve -> grade evidence -> refine search if needed -> answer with citations -> validate -> trace/evaluate
```

The important shift is not “more agents”. The important shift is that the system has explicit state, tool-use decisions, retrieval strategy, evidence quality checks, and observable execution traces.

A good 2026 agentic RAG system should therefore be designed as a **stateful, policy-controlled, observable workflow**, not as a single prompt wrapped around a vector database.

## 2. Main architecture layers

| Layer | Responsibility | Typical modules |
|---|---|---|
| Experience layer | User-facing chat, API, embedded assistant, admin UI | Web app, API gateway, chat UI, feedback UI |
| Identity and policy layer | Tenant isolation, user permissions, data access, rate limits | OIDC/SAML, RBAC/ABAC, policy engine, DLP, content filters |
| Agent orchestration layer | Planning, routing, memory, tool loops, state transitions | LangGraph-style graph runtime, planner, router, critic, state store |
| Retrieval layer | Find relevant evidence across knowledge sources | Hybrid search, vector search, BM25, graph traversal, SQL, MCP tools |
| Knowledge layer | Stores and indexes prepared knowledge | Object store, vector index, search index, graph DB, metadata DB |
| Ingestion layer | Load, parse, enrich, chunk, embed, index, version data | Connectors, OCR, parser, chunker, entity extractor, embedding worker |
| Generation layer | Compose cited answers from evidence | LLM gateway, prompt templates, answer composer, claim validator |
| Evaluation layer | Measure retrieval, answer, trajectory, and safety quality | Golden sets, trace evals, judge models, human review, regression tests |
| Observability layer | Debug and operate the system | OpenTelemetry traces, logs, metrics, dashboards, audit records |
| Platform layer | Runtime, networking, secrets, storage, CI/CD | Docker locally; Kubernetes/serverless/managed cloud in production |

## 3. Core components

### 3.1 API Gateway / Backend-for-Frontend

Accepts user requests and normalizes them into a canonical `AgentRunRequest`. It should not contain deep business logic. Its job is to:

- authenticate the caller;
- assign `tenant_id`, `user_id`, `session_id`, and `request_id`;
- enforce coarse-grained rate limits;
- stream partial responses where needed;
- pass a clean request envelope to the agent runtime.

### 3.2 Identity, tenancy, and authorization

This is a first-class component, not an afterthought. RAG systems are dangerous when they centralize enterprise content into a vector store without preserving original access control.

Minimum requirement:

- every document, chunk, entity, relation, and tool result carries ACL metadata;
- every retrieval query includes `tenant_id`, `user_id`, and `permission_context`;
- retrieval results are filtered before they are exposed to the model;
- privileged tools require scoped service identities and policy checks.

### 3.3 Agent Orchestrator

The orchestrator is the core runtime. Use a graph/state-machine style, because agentic RAG needs explicit transitions and failure handling.

Typical nodes:

1. `classify_intent`
2. `load_session_memory`
3. `plan_retrieval`
4. `route_tools`
5. `execute_retrieval`
6. `rerank_and_compress`
7. `grade_evidence`
8. `rewrite_or_expand_query`
9. `compose_answer`
10. `validate_claims`
11. `apply_policy`
12. `finalize_response`
13. `log_eval_candidate`

The orchestrator should enforce loop budgets:

```yaml
max_total_steps: 12
max_retrieval_rounds: 3
max_tool_calls_per_round: 8
max_tokens_evidence: 12000
max_latency_ms_interactive: 12000
max_cost_usd_per_run: 0.25
```

### 3.4 Planner / Router

The planner converts the user task into one or more retrieval/action intents.

It should answer:

- Is retrieval required?
- Which knowledge domains are relevant?
- Is the query simple, multi-hop, comparative, temporal, numerical, procedural, or policy-heavy?
- Should the system use semantic search, keyword search, hybrid search, graph traversal, SQL, or live API calls?
- Is the user asking for an answer, a document, a workflow execution, or a decision recommendation?

Planner output should be structured, not free text.

```json
{
  "task_type": "comparative_question",
  "requires_retrieval": true,
  "subqueries": [
    {"id": "q1", "query": "current production architecture", "source_hints": ["architecture_docs"]},
    {"id": "q2", "query": "deployment constraints", "source_hints": ["platform_docs"]}
  ],
  "retrieval_strategy": "hybrid_then_rerank",
  "answer_policy": "cite_all_nontrivial_claims"
}
```

### 3.5 Retrieval Tool Router

The retrieval router exposes retrieval capabilities as tools. Keep the interface stable even when the underlying provider changes.

Recommended retrieval tools:

| Tool | Use when |
|---|---|
| `hybrid_search` | Most enterprise text search; combines semantic and lexical recall |
| `vector_search` | Conceptual similarity, paraphrases, multilingual semantic search |
| `keyword_search` | Exact names, IDs, error messages, legal clauses, product codes |
| `graph_search` | Entity relationships, dependency maps, lineage, multi-hop reasoning |
| `sql_query` | Structured, numeric, transactional, reporting questions |
| `source_read` | Read full source section after finding a chunk |
| `mcp_retrieve` | Use external knowledge services via MCP-compatible endpoints |
| `web_or_external_search` | Only if the product requires public/current data |

### 3.6 Evidence Processor

This layer turns raw retrieval results into an `EvidenceBundle` that the answer composer can trust.

Responsibilities:

- deduplicate near-identical chunks;
- resolve chunk-to-source links;
- preserve page/section anchors;
- rerank by relevance and authority;
- compress long evidence without losing citation anchors;
- flag stale, low-confidence, or permission-sensitive evidence;
- group evidence by subquery and claim candidate.

### 3.7 Evidence Sufficiency Grader

Before answering, the system should ask: “Do we have enough evidence?”

The grader should produce:

```json
{
  "sufficient": false,
  "score": 0.62,
  "missing_aspects": ["production deployment constraints", "tenant isolation requirement"],
  "recommended_next_action": "expand_query",
  "query_rewrite": "tenant isolation requirements production RAG deployment"
}
```

This is where agentic RAG becomes self-correcting. If evidence is weak, the system loops back into retrieval rather than hallucinating.

### 3.8 Answer Composer

The composer should not receive arbitrary system internals. It should receive:

- the user question;
- a compact task plan;
- evidence bundles with source IDs;
- constraints and response style;
- citation rules;
- safety/policy reminders.

It should output a structured answer draft with claims and citations before final rendering.

### 3.9 Claim Validator

Validates whether each factual claim is supported by evidence. It does not need to be perfect, but it should catch obvious unsupported statements.

Validation checks:

- unsupported factual claim;
- citation points to irrelevant evidence;
- answer contradicts source evidence;
- answer exposes hidden/system/policy content;
- answer violates response contract;
- answer exceeds allowed confidence.

### 3.10 Memory

Use memory carefully. Separate these memory types:

| Memory type | Purpose | Storage |
|---|---|---|
| Session memory | Current conversation state | Redis/Postgres/session service |
| User preference memory | Stable user preferences | governed profile store |
| Task memory | Intermediate plan/tool/evidence state | trace/state DB |
| Domain memory | Enterprise knowledge | knowledge layer, not chat memory |
| Evaluation memory | Failures and improvement signals | eval datasets and trace lake |

Do not mix user preferences with retrieved enterprise evidence. Do not let untrusted retrieved text write directly into long-term memory.

## 4. Recommended baseline architecture

For a serious, portable build, use this baseline:

- **API**: FastAPI or similar async API.
- **Agent runtime**: graph/state-machine orchestration, LangGraph-style.
- **Model gateway**: one internal abstraction for OpenAI/Azure OpenAI/Bedrock/Gemini/local models.
- **Retrieval**: hybrid search as default, vector-only as secondary, keyword-only for exact recall, graph optional for multi-hop domains.
- **Metadata and state**: Postgres.
- **Local vector**: Qdrant or pgvector.
- **Local object store**: MinIO.
- **Async jobs**: Redis Queue/Celery/Temporal/NATS, depending on complexity.
- **Observability**: OpenTelemetry collector, traces, logs, metrics.
- **Security**: OIDC, policy engine, per-tenant ACL filters, secrets manager.

## 5. “Modern” capabilities to design for from day one

### Multi-query retrieval

Complex questions should be decomposed into focused subqueries. This is especially useful for comparison, temporal, policy, architecture, and multi-document questions.

### Hybrid search

Dense vector retrieval alone is often insufficient. Use hybrid retrieval for higher recall: lexical search catches exact terms, IDs, and names; vector search catches paraphrases and conceptual matches.

### Source reading

Chunk retrieval should be followed by source reading when exactness matters. This avoids answering from a tiny chunk that lacks context.

### Evidence sufficiency loop

If evidence is not enough, the system should rewrite the query, broaden/narrow filters, read source neighbors, or ask for clarification only when truly necessary.

### Structured traces

Every model call, tool call, retrieval query, rerank, decision, and validation result should be traceable. Production agent debugging is almost impossible without this.

### Provider-native accelerators

Use cloud-native RAG services where they fit:

- AWS: Bedrock Agents + Knowledge Bases.
- Azure: Azure AI Search agentic retrieval / knowledge base / MCP endpoint.
- GCP: Gemini Enterprise Agent Platform + RAG Engine + Agent Runtime.

But keep the core contracts portable.

## 6. Non-goals for the first implementation

Avoid these in the first version:

- fully autonomous action-taking without human approval;
- unbounded web browsing;
- direct write access to business systems;
- hidden chain-of-thought storage as an audit artifact;
- one giant prompt that handles routing, retrieval, generation, and validation;
- vector store without ACL filtering;
- production deployment before evaluation datasets exist.
