# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

This repository is **design-stage**: it currently contains only architecture specifications under
[docs/](docs/), no source code. There is no `pyproject.toml`/`setup.py`, build, lint, or test
tooling yet — the `.gitignore` is Python-oriented, and the docs target Python (typed) as the
default language. When you implement code, you are creating the structure the docs describe; favor
matching the layout and conventions in the IMPLEMENTATION.md files over inventing new ones.

## What this is

Two cooperating RAG systems built on **Clean Architecture**, documented in parallel:

- **Retrieval / query side** ([docs/retrieval/](docs/retrieval/)) — a query-adaptive *agentic* RAG
  system. A router sends each query down a cheap deterministic *fast path* or a fully agentic
  *deliberate path* (transform → plan → retrieve → fuse → rerank → grade → loop → generate →
  critique). Orchestrates three retrievers (dense text, BM25, multimodal) behind one
  `RetrieverTool` abstraction.
- **Ingestion / producer side** ([docs/ingestion/](docs/ingestion/)) — the offline/batch pipeline
  that feeds the three stores the query side reads. Acquires heterogeneous sources (YouTube, web
  pages incl. auth-gated, documents/PDF), normalizes everything to a single `NormalizedDocument`
  (canonical Markdown + media manifest + metadata + provenance), then chunks → enriches → embeds →
  indexes.

Each side has four docs: `README.md` (concept), `ARCHITECTURE.md` (ports, domain model, control
flow, trade-offs), `IMPLEMENTATION.md` (stack, phased build plan, module layout), `EVALUATION.md`
(quality harness). Read the relevant `ARCHITECTURE.md` before implementing any port or stage — it
specifies the exact entities, interfaces, and defaults.

## Architecture rules that must hold across all code

These are load-bearing invariants, not style preferences. Most edits will be wrong if they break one.

1. **Dependency rule (inward only).** Layers are `domain` → `application` (use cases + ports +
   policies) → `adapters` → `infrastructure`. `domain` imports nothing outward; `application`
   imports only `domain`; **only `adapters`/`infrastructure` may import a vendor SDK**. No vendor
   SDK import belongs in `domain` or `application` — this is meant to be CI-enforced (import-linter).

2. **Ports over implementations.** The core depends only on interfaces. Every external capability
   (LLM, vector DB, BM25, reranker, fusion, embedder, transcriber, crawler, PDF parser, captioner,
   OCR, index writer, ledger) is a port with swappable adapters. Adding a vendor = new adapter +
   one config branch. Adding a retrieval modality = register one more `RetrieverTool`. Adding a
   source = register one more `SourceConnector`.

3. **One composition root.** All concrete wiring lives in a single `infrastructure/container.py`
   driven by declarative config (YAML). It is the only code that branches on `provider`.

4. **The embedder parity invariant.** Ingestion-time text/multimodal embedders **must** equal
   query-time embedders (same model + version + pooling). The query side and ingestion side
   *share the same embedder ports and `Chunk`/`Metadata`/`Provenance` domain types* — import them
   from the shared package, do not re-declare. The composition root **fails fast** on mismatch.
   A mismatch silently destroys retrieval quality, so this is enforced, not documented.

5. **Provenance is mandatory.** No chunk is indexed without a resolvable `Anchor` back to its
   source (PDF page, video timestamp, doc heading). Citations on the query side depend on it; a
   chunk that can't be traced home is a defect caught in eval.

6. **Fakes from day one.** Every port ships an in-memory fake so the whole system runs in tests
   with zero network. A `ClockPort` keeps tests deterministic.

## Keystone concepts to keep in mind

- **`RetrieverTool` (query side)** unifies dense / BM25 / multimodal so the agent treats them
  identically. Default behavior is hybrid: run several in parallel and fuse (RRF, k≈60), then
  rerank the top-K with a cross-encoder ("retrieve wide, rerank narrow").
- **`NormalizedDocument` (ingestion side)** is the single representation every source converges on;
  everything downstream is source-blind. This is what lets a new source type slot in with zero
  downstream changes.
- **Triple-indexing of images.** Every image becomes both a `MediaAsset` and a `Chunk(modality=IMAGE)`,
  made retrievable three ways: caption+OCR text → BM25 and text vector store; pixels → multimodal
  image vector store.
- **Bounded agent.** The query-side agent is a *bounded state machine* governed by router /
  iteration / budget policies (max iterations, token/tool/latency ceilings) — never an open-ended
  loop. On budget exhaustion it finalizes in a degraded-but-honest mode.
- **Idempotent & incremental ingestion.** An ingestion ledger (content-hash keyed) skips unchanged
  sources and upserts by deterministic `chunk_id`; failures quarantine per-source without aborting
  the batch.

## Planned module layout

When building, follow the layouts in [docs/retrieval/IMPLEMENTATION.md §3](docs/retrieval/IMPLEMENTATION.md)
and [docs/ingestion/IMPLEMENTATION.md §3](docs/ingestion/IMPLEMENTATION.md). Both use
`domain/` → `application/{ports,usecases,policies,...}` → `adapters/` (SDK imports here) →
`infrastructure/` (container, config, entrypoint), with `tests/{fakes,unit,integration,eval,fixtures}`.

Build order is phased and each phase is independently shippable (walking skeleton on fakes first,
then swap in one real adapter per port). See §4 of each IMPLEMENTATION.md for the phase gates.

## Default stack (all swappable via config)

Python (typed); Qdrant (vector), OpenSearch (BM25), BGE/E5 (text embeddings), Jina-CLIP/SigLIP
(multimodal), Cohere/bge-reranker (rerank), RRF (fusion), Redis (cache), OpenTelemetry (telemetry).
Ingestion adds: Firecrawl (web fetch + auth + crawl), Docling (PDF→markdown), Tesseract (OCR), a
VLM (captioning), Whisper (ASR fallback for YouTube), Postgres (ledger), object storage (blobs).
Any framework (LangGraph/LlamaIndex/Haystack) must stay behind the application ports — never let
framework types leak into the domain.
