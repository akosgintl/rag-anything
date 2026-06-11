# 10 — Decision Log and Backlog

This file captures recommended architecture decisions and a prioritized backlog.

## 1. Architecture Decision Records

### ADR-001 — Use graph/state-machine orchestration

**Decision:** Use a graph/state-machine orchestrator for the agent runtime.

**Reason:** Agentic RAG needs explicit steps, loops, budgets, retries, validation, and traceable decisions. A single prompt-based chain is too opaque for production.

**Consequences:** Slightly more engineering upfront, much better debuggability and control.

### ADR-002 — Keep provider adapters behind internal interfaces

**Decision:** The core agent should not directly depend on AWS, Azure, or GCP SDKs.

**Reason:** The user intent includes development, production, and multi-cloud deployment. Provider-specific logic should live in adapters.

**Consequences:** More abstraction work, but much less lock-in.

### ADR-003 — Use hybrid retrieval as default

**Decision:** Default retrieval strategy should be hybrid search: semantic vector + lexical keyword, with reranking where needed.

**Reason:** Vector-only retrieval misses exact terms, IDs, error codes, legal clauses, and product names. Keyword-only retrieval misses paraphrases.

**Consequences:** Requires either a combined search service or result merging logic.

### ADR-004 — Preserve ACLs at chunk level

**Decision:** Every chunk and retrieval result must include tenant and ACL metadata.

**Reason:** RAG systems can leak data if vector/search indexes bypass original source permissions.

**Consequences:** Ingestion and retrieval become more complex, but this is mandatory for enterprise use.

### ADR-005 — Treat retrieved content as untrusted

**Decision:** Retrieved documents are evidence, not instructions.

**Reason:** Prompt injection can live inside documents, webpages, tickets, emails, or PDFs.

**Consequences:** Evidence wrappers, injection detectors, and validation are required.

### ADR-006 — Store traces for every production run

**Decision:** Every run should emit structured traces with model/tool/retrieval events.

**Reason:** Agentic systems are hard to debug from final answers alone.

**Consequences:** Need redaction, retention, and cost controls for telemetry.

### ADR-007 — Make evaluation part of CI/CD

**Decision:** Retrieval, answer, trajectory, and safety evals should gate releases.

**Reason:** Prompt/model/retrieval changes can silently regress quality.

**Consequences:** Requires maintained eval datasets and scoring infrastructure.

### ADR-008 — First release is read-only

**Decision:** Initial production version should not execute write actions autonomously.

**Reason:** Read-only RAG already provides value and is much safer.

**Consequences:** Action tools can be introduced later with approval workflows.

## 2. Prioritized backlog

### P0 — Foundation

- Create repo structure.
- Implement Docker Compose.
- Implement schemas.
- Implement metadata migrations.
- Implement local ingestion.
- Implement hybrid retrieval.
- Implement basic graph agent.
- Implement cited answer.
- Implement traces.

### P1 — Production quality

- Evidence sufficiency grader.
- Claim/citation validator.
- Prompt injection tests.
- ACL enforcement tests.
- Eval runner.
- CI gate.
- Cloud pilot deployment.
- Dashboards and alerts.

### P2 — Advanced retrieval

- Graph extraction.
- Graph query tool.
- Table-aware retrieval.
- Source-neighbor expansion.
- Query rewrite variants.
- Cross-source conflict detection.
- Entity-aware reranking.

### P3 — Cloud-native adapters

- AWS Bedrock Knowledge Bases adapter.
- Azure AI Search agentic retrieval adapter.
- Azure AI Search MCP endpoint adapter.
- GCP RAG Engine adapter.
- Provider cost comparison dashboard.

### P4 — Advanced agent features

- Deep research mode.
- Human approval workflow.
- Read-only business API tools.
- Sandboxed code execution.
- Report generation.
- Multi-agent review pattern.

## 3. Open questions

| Question | Why it matters | Suggested default |
|---|---|---|
| First cloud provider? | Determines first production adapter | Choose customer/platform-native cloud |
| First corpus? | Quality depends on real documents | Architecture docs + ADRs + runbooks |
| Need PDF/OCR immediately? | Impacts ingestion complexity | Start text/markdown, add PDF next |
| Need graph retrieval in MVP? | Adds complexity | No, unless domain is heavily relational |
| Need MCP in MVP? | Useful but not mandatory | Design for it, add after retrieval MVP |
| Need local LLM? | Privacy/cost/dev experience | Optional profile with Ollama/vLLM |
| Need multi-tenancy in MVP? | Security architecture | Yes, at least tenant ID and ACL model |

## 4. Risk register

| Risk | Impact | Mitigation |
|---|---|---|
| Retrieval quality poor | wrong answers | hybrid retrieval, reranking, evals |
| Hallucinated answers | trust loss | evidence grading, claim validation, citations |
| ACL leakage | severe security issue | chunk-level ACL, tests, audit |
| Prompt injection | unsafe behavior | treat docs as untrusted, detectors, validation |
| Cost blow-up | budget issue | model gateway, budgets, caching |
| Cloud lock-in | portability issue | adapters, canonical schemas, eval portability |
| Observability gap | hard to debug | OTel from start |
| Ingestion failures | stale/missing knowledge | manifests, retry, quarantine |
| Too much scope | late delivery | read-only MVP, one corpus, one cloud pilot |

## 5. Recommended first milestone definition

“Architecture assistant MVP”:

- source corpus: architecture docs and ADRs;
- input: architecture questions;
- output: cited markdown answer;
- deployment: Docker local;
- retrieval: hybrid;
- security: tenant + ACL filter;
- observability: trace per run;
- eval: 30 cases.

Example MVP demo question:

> “Compare the current architecture with the target cloud architecture and list the top migration risks.”

Expected answer:

- retrieves current architecture;
- retrieves target cloud architecture;
- retrieves security constraints;
- identifies differences;
- cites each major claim;
- marks assumptions;
- stores trace.

## 6. Definition of done

### Feature done

- code implemented;
- unit tests pass;
- integration tests pass;
- trace coverage added;
- schema documented;
- eval case added if behavior affects quality;
- security implications reviewed.

### Release done

- eval gate passes;
- dashboards updated;
- runbook updated;
- rollback path tested;
- prompt/model/retrieval versions recorded;
- canary plan approved.
