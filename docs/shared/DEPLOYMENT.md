# Deployment (shared)

The two `IMPLEMENTATION.md` docs describe the **local-development stack** — what runs on your
laptop in one `docker-compose` (a single Supabase/Postgres backbone plus stateless processing
adapters and the online models). **This doc describes how each of those components deploys beyond
the laptop**, and *why* each default was chosen.

Two rules make every row here a config change rather than a rewrite:

1. **Everything stays behind a port.** Each component is named by the port it implements
   (`ledger`, `index_writer`, `search`, `embedder`, `reranker`, `llm`, …) — defined in
   [retrieval/IMPLEMENTATION.md §3](../retrieval/IMPLEMENTATION.md),
   [ingestion/IMPLEMENTATION.md §3](../ingestion/IMPLEMENTATION.md), and the shared
   [`DATA_MODEL.md`](./DATA_MODEL.md). Swapping a deployment target = swapping one adapter.
2. **Open standards are the escape hatch.** We commit to **Postgres wire**, the **S3 API**,
   **RESP** (Redis protocol), **OpenTelemetry**, and the **OpenAI-compatible LLM API**. Any vendor
   that speaks these is interchangeable.

**Consolidate locally, split for production.** Start with the one backbone; move a single port to a
dedicated or managed service *only when that concern outgrows Postgres* — never as an upfront tax.

---

## 1. Deployment matrix

Per component: the self-host / CLI option, a cloud-agnostic (portable, vendor-neutral) managed
option, and the native managed service on each of the three major clouds. `—` = no meaningful
managed equivalent (run the self-host option, or it isn't a hosted concern).

| Component (port) | CLI / self-host | Cloud-agnostic | AWS | GCP | Azure |
|------------------|-----------------|----------------|-----|-----|-------|
| Relational ledger (`ledger`) | Postgres | Neon, Supabase, CockroachDB | RDS / Aurora PostgreSQL | Cloud SQL / AlloyDB | Azure DB for PostgreSQL |
| Vector store (`index_writer` / `search`) | pgvector + pgvectorscale, Qdrant, Milvus | Qdrant Cloud, Zilliz, Pinecone | Aurora pgvector / OpenSearch k-NN | Vertex Vector Search / AlloyDB AI | Azure AI Search |
| Keyword / BM25 (`index_writer` / `search`) | ParadeDB `pg_search`, OpenSearch | Elastic Cloud | OpenSearch Service | Vertex AI Search / Elastic on GKE | Azure AI Search |
| Blob store (`document_store`) | MinIO, SeaweedFS | Cloudflare R2, Backblaze B2 | S3 | GCS | Azure Blob |
| Cache (`dedup` / cache) | Redis, Valkey, Dragonfly | Upstash | ElastiCache | Memorystore | Azure Cache for Redis |
| Job queue / orchestration | native Python + `pgmq`, Prefect, Temporal, Dagster | Prefect Cloud, Temporal Cloud | SQS + Step Functions / MWAA | Cloud Tasks + Workflows / Composer | Azure Queue + Logic Apps / Data Factory |
| Secrets | Vault, OpenBao, Infisical | Doppler, Infisical Cloud | Secrets Manager | Secret Manager | Key Vault |
| Doc parse / OCR (`extractor` / `ocr`) | Docling + Tesseract; poppler / MuPDF / gs | LlamaParse, Mistral OCR, Reducto | Textract | Document AI | Azure AI Document Intelligence |
| Media preprocessing | ffmpeg, libvips, yt-dlp | — | MediaConvert | Transcoder API | Azure Media Services |
| Web fetch (`web_fetcher`) | Firecrawl self-host, Playwright, Crawl4AI | Firecrawl Cloud, Jina Reader, Tavily, Apify | — | — | — |
| Transcript / ASR (`transcript`) | yt-dlp + faster-whisper | Gemini (native URL), Deepgram, AssemblyAI | Transcribe | Speech-to-Text | Azure AI Speech |
| Captioning / VLM (`vision_captioner`) | Qwen2.5-VL via vLLM | Gemini, Claude, GPT (via LiteLLM) | Bedrock (Claude) | Vertex (Gemini) | Azure OpenAI (GPT-4o) |
| Text embeddings (`embedder`) | BGE / E5 via HF TEI | Voyage, Cohere, Jina, OpenAI | Bedrock (Titan / Cohere) | Vertex text-embedding | Azure OpenAI embeddings |
| Multimodal embeddings (`embedder`) | Jina-CLIP / SigLIP via HF TEI | Cohere Embed v4, Voyage multimodal | Bedrock (Titan multimodal) | Vertex multimodalembedding | Azure AI Vision |
| Reranker (`reranker`) | bge-reranker via HF TEI | Cohere Rerank, Voyage, Jina | Bedrock Rerank | Vertex Ranking API | Azure AI Search semantic ranker |
| LLM serving (`llm`) | vLLM, SGLang, Ollama | Claude, GPT, Gemini (via LiteLLM) | Bedrock | Vertex | Azure OpenAI |
| Telemetry (`telemetry`) | OTel + Grafana LGTM, Jaeger | Grafana Cloud, Honeycomb, Datadog | CloudWatch + X-Ray | Cloud Trace + Monitoring | Azure Monitor |
| LLM trace / eval | Opik, Langfuse, Arize Phoenix | Opik Cloud, Langfuse Cloud, LangSmith | Bedrock evals | Vertex AI eval | Azure AI eval |

---

## 2. Rationale per choice

### Backbone — consolidate, then split one port at a time

One Postgres engine legitimately covers ledger + vector + BM25 + cache + queue, which is why local
dev is a single stack:

- **Vector** — `pgvector` + `pgvectorscale` benchmarks at roughly **11× Qdrant's QPS at 50M vectors**
  and is "more than sufficient for ~80% of RAG"; plain `pgvector` is fine below ~5M. **Split to
  Qdrant** (or Milvus/Weaviate) only when you cross ~5–50M vectors, rebuild indexes frequently, or
  have strict tail-latency SLAs — Qdrant wins on index-build time and single-query p99.
- **Keyword / BM25** — **ParadeDB `pg_search`** brings true BM25 + RRF into Postgres and matches
  Elasticsearch relevance with no separate cluster, JVM, or sync pipeline. **Split to
  OpenSearch / Elasticsearch** for very high search QPS or complex faceting. (Managed-Postgres BM25
  support varies by provider, so the BM25-in-PG route means self-host ParadeDB or a provider that
  ships it.)
- Splitting is a one-adapter change because the read/write paths are the `index_writer` /
  `VectorSearchPort` / `KeywordSearchPort` ports, not Postgres-specific code.

### Storage, ledger, cache — ride the open protocols

- **Blob** — the **S3 API** is the contract; MinIO/SeaweedFS locally, any of S3 / GCS / Azure Blob /
  R2 in the cloud, all via `boto3` + an endpoint URL. R2 is attractive for zero egress fees.
- **Ledger** — plain **Postgres wire** (`asyncpg`, Alembic migrations); Neon/Supabase/RDS/Cloud SQL
  are drop-in. Avoid provider-only features (RDS IAM auth, Cloud SQL extensions) to stay portable.
- **Cache** — **RESP** is the contract. Prefer **Valkey** (the BSD-3 Linux Foundation fork) over
  Redis to dodge the Redis licensing change; ElastiCache/Memorystore/Upstash are all RESP-compatible.
  In local dev, an in-proc LRU + Postgres is enough — add a cache server only when QPS demands it.

### Orchestration — native Python → Prefect → Temporal

Deliberately staged; do not add an orchestrator before you need one:

- **Start native** (`asyncio` + **`pgmq`** + the Postgres **ledger**). This already gives
  at-least-once delivery (re-drive from the queue), retries/backoff (visibility timeout + ledger
  status), per-source quarantine, and bounded concurrency. The whole phased build ships on this —
  no extra moving part, which is the point.
- **Adopt Prefect** when you productionize **scheduled recrawls and backfills** and want a
  retries/observability UI, concurrency limits, and caching without hand-rolling them. Prefect has
  the best Python-native ergonomics (decorators, low ceremony) and the cleanest local→cloud path; it
  fits a task/flow-shaped pipeline better than Dagster's asset graph here.
- **Reach for Temporal** only when workflows become genuinely **long-running and durable** — running
  for hours/days, needing to survive a process crash mid-flight, exactly-once steps, or
  human-in-the-loop pauses. Its durable execution / replay-and-resume is the differentiator and the
  reason to pay its operational cost; batch ingestion usually never needs it.

All three sit behind the same queue/orchestrator port, so the progression is additive.

### Secrets & observability — vendor-neutral contracts

- **Secrets** — a thin `get_secret(key)` wrapper hides the provider. Self-host Vault / **OpenBao**
  (the OSS Vault fork) / Infisical; in the cloud use the native manager (Secrets Manager / Secret
  Manager / Key Vault). Never import a provider SDK outside the adapter.
- **Telemetry** — **OpenTelemetry** is the instrumentation standard; the backend
  (Grafana stack / Honeycomb / Datadog / a cloud-native APM) is swapped via exporters with no code
  change. Use OTel semantic conventions; avoid vendor-specific span attributes.

### Document & media processing — Docling first, native CLI for the cheap path

- **Docling** (with its **built-in Tesseract** OCR engine) is the default for layout, tables, and
  reading order. Swap the OCR engine it drives (PaddleOCR / Surya / RapidOCR) per language, or send
  hard pages to cloud OCR (Document AI / Textract / Azure AI Document Intelligence) when accuracy or
  throughput demands it.
- The **native-CLI lane** — poppler (`pdftotext`/`pdfimages`/`pdftoppm`), MuPDF (`mutool`),
  Ghostscript (`gs`), **libvips**, **ffmpeg** (audio for ASR, keyframes for captioning), **yt-dlp** —
  are single binaries, multithreaded, and embarrassingly parallel. Fan out per file with GNU
  `parallel` / `xargs -P $(nproc)` in dev, the job queue at scale. Use them for the cheap/fast
  extraction and preprocessing that feeds OCR/captioning/ASR.

### Models — the parity invariant and one LLM swap point

- **Embeddings** run behind HF **TEI** locally. The **parity invariant** is load-bearing: ingestion
  and query *must* use the identical embedder (model + version + pooling), so changing the embedder
  means a re-embed / blue-green reindex, not an in-place swap. The composition root fails fast on
  mismatch.
- **LLM** stays online via **LiteLLM** — a single OpenAI-compatible swap point across Claude / GPT /
  Gemini (and self-host vLLM / SGLang when you want open weights). Role-route a small utility model
  vs. the answer model behind the same `LLMPort`.
- **YouTube transcription** defaults to **Gemini native URL transcription** (the model transcribes
  the video from the URL directly) — no download, no separate ASR service, and it handles
  caption-less videos. yt-dlp + faster-whisper is the fully-offline fallback; cloud STT
  (Transcribe / Speech-to-Text / Azure AI Speech) is the managed alternative.

### LLM trace / eval — Opik (recommended), Langfuse as the alternative

Both are OSS, self-hostable, sit behind the same tracing/eval port, and **both support prompt
versioning** — so this is not a one-way door. We default to **Opik (Comet)** because:

- it is already wired in via `track_genai_client` (first-class `google-genai` integration),
- it is **eval-first** — built-in LLM-as-judge metrics (hallucination, relevance, moderation) that
  pair directly with the RAGAS / DeepEval harness, and
- it includes prompt versioning, so the in-repo versioned `prompts/` artifacts have a home.

**Choose Langfuse instead** if you want the larger ecosystem / integration surface or a more
"platform"-style product. Either way, traces are emitted over **OpenTelemetry** and metrics are
RAGAS/DeepEval libraries, so the evaluation data is portable.

---

## 3. Topology notes

- **Stateless workers.** Both the ingestion workers and the query service are stateless and scale
  horizontally; all durable state is the backbone (or its split-out managed services).
- **Reindex / migration.** Embedder or `schema_version` upgrades run as a **blue-green backfill from
  the stored normalized-document blobs** (no re-fetch), validated on the golden set before cut-over —
  never an in-place swap, because the parity invariant forbids it.
- **Cost centers behind gates.** ASR/VLM/embedding/LLM calls run behind caches and rate limits; the
  ledger guarantees at-most-once processing per unchanged unit.
