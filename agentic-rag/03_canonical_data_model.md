# 03 — Canonical Data Model

This file defines the data that should flow through a production-grade agentic RAG system.

The goal is not to force one database schema. The goal is to create stable internal contracts so the system can run locally, on AWS, on Azure, or on GCP without rewriting the agent logic.

## 1. Design principles

1. **Every object is tenant-aware.** `tenant_id` is mandatory across runtime, ingestion, retrieval, traces, and evaluation.
2. **Every evidence item is source-addressable.** A chunk must point back to document, version, section, page, row, slide, timestamp, or source API response.
3. **Every retrieval result is permission-checked.** Do not rely on the LLM to ignore unauthorized content.
4. **Every answer claim should be traceable to evidence.** Final answers should have claim-level support where practical.
5. **Every agent run is reproducible enough for debugging.** Store prompt versions, model IDs, tool inputs, retrieval IDs, and output hashes.
6. **Every indexable item has lifecycle state.** Draft, active, deprecated, deleted, quarantined.

## 2. Runtime data model

### 2.1 AgentRunRequest

```json
{
  "request_id": "req_01",
  "tenant_id": "tenant_acme",
  "user_id": "user_123",
  "session_id": "sess_456",
  "input": {
    "type": "user_message",
    "text": "What changed in the target architecture?",
    "attachments": []
  },
  "runtime_options": {
    "stream": true,
    "max_latency_ms": 12000,
    "max_cost_usd": 0.25,
    "answer_format": "markdown",
    "citation_policy": "required"
  },
  "permission_context": {
    "roles": ["architect"],
    "groups": ["project-alpha-read"],
    "classification_ceiling": "confidential",
    "regions": ["eu"]
  },
  "client_context": {
    "app": "architecture-assistant",
    "locale": "en-US",
    "timezone": "Europe/Budapest"
  }
}
```

### 2.2 AgentRunState

This is the central object that the orchestrator updates through the graph.

```json
{
  "run_id": "run_01",
  "request_id": "req_01",
  "tenant_id": "tenant_acme",
  "user_id": "user_123",
  "session_id": "sess_456",
  "status": "running",
  "step_index": 4,
  "budgets": {
    "max_steps": 12,
    "max_retrieval_rounds": 3,
    "max_tool_calls": 20,
    "max_tokens_evidence": 12000,
    "max_cost_usd": 0.25
  },
  "messages": [],
  "task": {},
  "plan": {},
  "retrieval_requests": [],
  "tool_calls": [],
  "evidence_bundles": [],
  "answer_drafts": [],
  "validation_results": [],
  "policy_decisions": [],
  "trace_refs": []
}
```

### 2.3 TaskUnderstanding

```json
{
  "task_id": "task_01",
  "task_class": "architecture_question",
  "intent": "compare_and_recommend",
  "requires_retrieval": true,
  "requires_action": false,
  "sensitivity": "internal",
  "ambiguities": [],
  "output_requirements": {
    "format": "markdown",
    "include_citations": true,
    "include_assumptions": true
  }
}
```

### 2.4 Plan

```json
{
  "plan_id": "plan_01",
  "version": 1,
  "strategy": "multi_query_hybrid_rag",
  "steps": [
    {
      "step_id": "p1",
      "type": "retrieve",
      "goal": "Find current architecture baseline",
      "tool": "hybrid_search",
      "depends_on": []
    },
    {
      "step_id": "p2",
      "type": "retrieve",
      "goal": "Find target cloud deployment constraints",
      "tool": "hybrid_search",
      "depends_on": []
    },
    {
      "step_id": "p3",
      "type": "synthesize",
      "goal": "Produce comparison and implementation recommendation",
      "depends_on": ["p1", "p2"]
    }
  ],
  "stop_conditions": {
    "evidence_sufficiency_score_gte": 0.78,
    "max_retrieval_rounds": 3
  }
}
```

### 2.5 RetrievalRequest

```json
{
  "retrieval_request_id": "rr_01",
  "tenant_id": "tenant_acme",
  "run_id": "run_01",
  "strategy": "hybrid_then_rerank",
  "queries": [
    {
      "query_id": "q1",
      "text": "current architecture baseline components",
      "query_type": "hybrid",
      "source_hints": ["architecture_docs", "adr"],
      "filters": {
        "doc_status": "active",
        "classification_lte": "confidential"
      },
      "top_k": 30
    }
  ],
  "acl_context_ref": "aclctx_01",
  "rerank": {
    "enabled": true,
    "model": "reranker_default",
    "top_n": 8
  },
  "source_read": {
    "enabled": true,
    "neighbor_chunks": 1
  }
}
```

### 2.6 RetrievalResult

```json
{
  "retrieval_result_id": "res_01",
  "query_id": "q1",
  "items": [
    {
      "result_id": "r1",
      "source_id": "doc_001",
      "document_version_id": "docv_001_2026_05_01",
      "chunk_id": "chk_001_004",
      "score": 0.88,
      "score_components": {
        "vector": 0.82,
        "keyword": 0.71,
        "semantic_ranker": 0.91,
        "recency_boost": 0.04
      },
      "title": "Target Cloud Architecture",
      "snippet": "The platform uses a graph-based orchestrator...",
      "metadata": {
        "source_type": "markdown",
        "classification": "internal",
        "owner": "architecture-team",
        "last_modified": "2026-05-21"
      },
      "acl_verified": true
    }
  ]
}
```

### 2.7 EvidenceItem

A retrieval result becomes evidence only after filtering, deduplication, source linking, and optional source reading.

```json
{
  "evidence_id": "ev_01",
  "source_id": "doc_001",
  "document_version_id": "docv_001_2026_05_01",
  "chunk_ids": ["chk_001_004", "chk_001_005"],
  "text": "The production deployment uses a graph-based agent runtime...",
  "summary": "Target architecture uses graph orchestration and hybrid retrieval.",
  "citation": {
    "label": "Target Cloud Architecture, Runtime Topology",
    "uri": "internal://docs/doc_001#runtime-topology",
    "page": null,
    "section": "Runtime Topology"
  },
  "quality": {
    "relevance": 0.91,
    "authority": 0.86,
    "freshness": 0.95,
    "completeness": 0.78,
    "permission_confidence": 1.0
  },
  "risk_flags": []
}
```

### 2.8 EvidenceBundle

```json
{
  "evidence_bundle_id": "evb_01",
  "run_id": "run_01",
  "coverage": {
    "answered_subqueries": ["q1", "q2"],
    "missing_subqueries": [],
    "conflicts": []
  },
  "items": ["ev_01", "ev_02", "ev_03"],
  "sufficiency": {
    "score": 0.83,
    "sufficient": true,
    "rationale": "Evidence covers current architecture, target deployment, and security constraints."
  }
}
```

### 2.9 AnswerDraft

```json
{
  "draft_id": "draft_01",
  "run_id": "run_01",
  "claims": [
    {
      "claim_id": "c1",
      "text": "The target system should use a graph/state-machine orchestrator rather than a single prompt pipeline.",
      "supporting_evidence_ids": ["ev_01"],
      "confidence": 0.86
    }
  ],
  "rendered_markdown": "...",
  "citations": [
    {
      "citation_id": "cit_01",
      "evidence_id": "ev_01",
      "rendered": "Target Cloud Architecture, Runtime Topology"
    }
  ]
}
```

### 2.10 ValidationResult

```json
{
  "validation_id": "val_01",
  "draft_id": "draft_01",
  "valid": true,
  "scores": {
    "claim_support": 0.89,
    "citation_relevance": 0.84,
    "policy_compliance": 1.0,
    "answer_completeness": 0.81
  },
  "issues": []
}
```

### 2.11 FinalAnswer

```json
{
  "answer_id": "ans_01",
  "run_id": "run_01",
  "status": "completed",
  "text_markdown": "...",
  "citations": [],
  "confidence": {
    "overall": 0.84,
    "limitations": ["Cost estimates require provider-specific sizing."
    ]
  },
  "usage": {
    "input_tokens": 14200,
    "output_tokens": 1700,
    "tool_calls": 6,
    "latency_ms": 8900,
    "estimated_cost_usd": 0.12
  }
}
```

## 3. Ingestion data model

### 3.1 KnowledgeSource

```json
{
  "knowledge_source_id": "ks_arch_docs",
  "tenant_id": "tenant_acme",
  "name": "Architecture Documents",
  "source_type": "git_repo",
  "connector": {
    "type": "github",
    "config_ref": "secret_or_connection_ref"
  },
  "sync_policy": {
    "mode": "incremental",
    "schedule": "hourly",
    "delete_behavior": "soft_delete"
  },
  "default_classification": "internal",
  "owner_team": "architecture-team",
  "enabled": true
}
```

### 3.2 RawDocument

```json
{
  "document_id": "doc_001",
  "tenant_id": "tenant_acme",
  "knowledge_source_id": "ks_arch_docs",
  "external_id": "repo/path/target_architecture.md",
  "title": "Target Cloud Architecture",
  "mime_type": "text/markdown",
  "object_uri": "s3://bucket/raw/doc_001.md",
  "hash_sha256": "...",
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": "2026-05-21T10:00:00Z",
  "status": "active",
  "classification": "internal",
  "acl": {
    "allow_groups": ["architecture-readers"],
    "deny_groups": [],
    "owner": "architecture-team"
  }
}
```

### 3.3 DocumentVersion

```json
{
  "document_version_id": "docv_001_2026_05_21",
  "document_id": "doc_001",
  "version_label": "git:abc123",
  "content_hash": "...",
  "parser_version": "parser_2.1.0",
  "chunker_version": "chunker_semantic_1.3.0",
  "embedding_profile": "text-embedding-large-v1",
  "indexed_at": "2026-05-21T10:05:00Z",
  "status": "active"
}
```

### 3.4 ParsedBlock

Use parsed blocks before chunks. Blocks preserve document structure.

```json
{
  "block_id": "blk_001",
  "document_version_id": "docv_001_2026_05_21",
  "block_type": "heading|paragraph|table|figure|code|list|page",
  "text": "Runtime Topology",
  "position": {
    "page": 3,
    "section_path": ["Architecture", "Runtime Topology"],
    "start_char": 1450,
    "end_char": 1470
  },
  "metadata": {}
}
```

### 3.5 Chunk

```json
{
  "chunk_id": "chk_001_004",
  "document_version_id": "docv_001_2026_05_21",
  "tenant_id": "tenant_acme",
  "text": "The runtime topology consists of...",
  "chunk_index": 4,
  "token_count": 420,
  "source_block_ids": ["blk_010", "blk_011"],
  "neighbor_chunk_ids": {
    "previous": "chk_001_003",
    "next": "chk_001_005"
  },
  "metadata": {
    "title": "Target Cloud Architecture",
    "section": "Runtime Topology",
    "source_type": "markdown",
    "classification": "internal"
  },
  "acl": {
    "allow_groups": ["architecture-readers"]
  },
  "status": "active"
}
```

### 3.6 EmbeddingRecord

```json
{
  "embedding_id": "emb_001",
  "chunk_id": "chk_001_004",
  "embedding_profile": "text-embedding-large-v1",
  "dimension": 1536,
  "vector_ref": "vector_db://collection/chk_001_004",
  "created_at": "2026-05-21T10:07:00Z"
}
```

### 3.7 Entity and Relation

Graph extraction is optional but useful for architecture, code, policy, dependency, and enterprise knowledge.

```json
{
  "entity_id": "ent_001",
  "tenant_id": "tenant_acme",
  "name": "Agent Orchestrator",
  "type": "software_component",
  "aliases": ["Graph Runtime"],
  "source_evidence_ids": ["chk_001_004"],
  "confidence": 0.82
}
```

```json
{
  "relation_id": "rel_001",
  "tenant_id": "tenant_acme",
  "subject_entity_id": "ent_001",
  "predicate": "calls",
  "object_entity_id": "ent_002",
  "source_evidence_ids": ["chk_001_004"],
  "confidence": 0.79
}
```

## 4. Trace and observability data model

### 4.1 TraceEvent

```json
{
  "trace_id": "trace_abc",
  "span_id": "span_001",
  "parent_span_id": null,
  "run_id": "run_01",
  "tenant_id": "tenant_acme",
  "event_type": "model_call|tool_call|retrieval|rerank|validation|policy_decision",
  "name": "hybrid_search",
  "start_time": "2026-06-08T10:00:00Z",
  "end_time": "2026-06-08T10:00:01Z",
  "attributes": {
    "tool.name": "hybrid_search",
    "retrieval.top_k": 30,
    "retrieval.returned": 8,
    "model.name": null,
    "cost.usd": 0.002
  },
  "input_hash": "...",
  "output_hash": "...",
  "redaction_level": "standard"
}
```

### 4.2 AuditEvent

```json
{
  "audit_event_id": "audit_001",
  "tenant_id": "tenant_acme",
  "actor_user_id": "user_123",
  "action": "retrieve_document_chunk",
  "resource_type": "chunk",
  "resource_id": "chk_001_004",
  "decision": "allow",
  "policy_id": "policy_acl_v1",
  "timestamp": "2026-06-08T10:00:01Z"
}
```

## 5. Evaluation data model

### 5.1 EvalCase

```json
{
  "eval_case_id": "eval_001",
  "tenant_id": "tenant_acme",
  "dataset": "architecture_rag_regression",
  "question": "Which component owns query planning?",
  "expected_answer_points": [
    "Planner/router owns query planning",
    "Orchestrator executes graph transitions",
    "Retrieval router executes abstract retrieval tools"
  ],
  "required_source_ids": ["doc_001"],
  "forbidden_source_ids": [],
  "tags": ["architecture", "simple", "citation_required"]
}
```

### 5.2 EvalRun

```json
{
  "eval_run_id": "erun_001",
  "eval_case_id": "eval_001",
  "agent_version": "agent_0.4.0",
  "prompt_version": "answer_prompt_0.7.0",
  "retrieval_profile": "hybrid_rerank_v2",
  "scores": {
    "answer_correctness": 0.87,
    "faithfulness": 0.92,
    "citation_coverage": 0.88,
    "retrieval_recall": 0.83,
    "trajectory_efficiency": 0.76,
    "latency_ms": 7200,
    "cost_usd": 0.08
  },
  "failure_modes": []
}
```

## 6. Suggested database/storage mapping

| Data object | Local development | Production cloud-neutral | AWS | Azure | GCP |
|---|---|---|---|---|---|
| Raw documents | MinIO | Object storage | S3 | Blob / ADLS | Cloud Storage |
| Metadata | Postgres | Managed Postgres | Aurora/RDS | Azure Database for PostgreSQL | Cloud SQL / AlloyDB |
| Vectors | Qdrant/pgvector | Managed vector/search | OpenSearch Serverless, Aurora pgvector, S3 Vectors, Bedrock KB | Azure AI Search | RAG Engine/Spanner vector, Vertex AI Vector Search |
| Keyword index | OpenSearch/Meilisearch | Managed search | OpenSearch | Azure AI Search | Elasticsearch/OpenSearch on GKE or partner |
| Graph | Neo4j local | Managed graph | Neptune / Neo4j Aura | Neo4j Aura / Cosmos DB Gremlin where suitable | Neo4j Aura / Graph DB on GKE |
| Queue | Redis/NATS | Managed queue | SQS/EventBridge | Service Bus/Event Grid | Pub/Sub/Eventarc |
| Trace | OTel Collector + Jaeger | OTel + APM | CloudWatch/X-Ray/third-party | App Insights/Monitor | Cloud Trace/Logging/Monitoring |
| Eval store | Postgres + object store | Warehouse/lake | S3 + Athena/Redshift | ADLS + Synapse/Fabric | BigQuery + GCS |

## 7. Minimal Pydantic-style model sketch

```python
from pydantic import BaseModel, Field
from typing import Literal, Any

class PermissionContext(BaseModel):
    roles: list[str] = []
    groups: list[str] = []
    classification_ceiling: str = "internal"
    regions: list[str] = []

class AgentRunRequest(BaseModel):
    request_id: str
    tenant_id: str
    user_id: str
    session_id: str
    input: dict[str, Any]
    runtime_options: dict[str, Any] = {}
    permission_context: PermissionContext

class RetrievalQuery(BaseModel):
    query_id: str
    text: str
    query_type: Literal["keyword", "vector", "hybrid", "graph", "sql"]
    filters: dict[str, Any] = {}
    top_k: int = 20

class EvidenceCitation(BaseModel):
    label: str
    uri: str
    page: int | None = None
    section: str | None = None

class EvidenceItem(BaseModel):
    evidence_id: str
    source_id: str
    document_version_id: str | None = None
    chunk_ids: list[str] = []
    text: str
    citation: EvidenceCitation
    quality: dict[str, float] = {}
    risk_flags: list[str] = []

class EvidenceBundle(BaseModel):
    evidence_bundle_id: str
    run_id: str
    items: list[EvidenceItem]
    sufficiency: dict[str, Any]

class Claim(BaseModel):
    claim_id: str
    text: str
    supporting_evidence_ids: list[str]
    confidence: float = Field(ge=0, le=1)
```
