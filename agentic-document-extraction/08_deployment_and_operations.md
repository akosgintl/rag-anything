# 08 — Deployment and Operations

## 1. Deployment goals

The platform should support:

- local Docker development,
- production Kubernetes deployment,
- cloud-native AWS, Azure, and GCP variants,
- local/open-source VLM/LLM serving such as vLLM,
- managed OCR/document AI providers,
- replayable and observable operations.

## 2. Environment architecture

```mermaid
flowchart TB
    subgraph Dev[Development]
        D1[Docker Compose]
        D2[Local object storage]
        D3[PostgreSQL]
        D4[Queue]
        D5[vLLM / local model]
    end

    subgraph Prod[Production]
        P1[API Gateway / Ingress]
        P2[Kubernetes services]
        P3[Managed object storage]
        P4[Managed DB]
        P5[Queue / event bus]
        P6[GPU inference pool]
        P7[Managed OCR/Document AI adapters]
        P8[Observability]
    end

    Dev --> Prod
```

## 3. Local Docker development stack

Recommended services:

| Service | Suggested local tool |
|---|---|
| API/orchestrator | FastAPI / Python service |
| Worker queue | Redis Queue, Celery, Dramatiq, or Temporal local |
| Metadata DB | PostgreSQL |
| Object storage | MinIO |
| OCR/layout | Tesseract/PaddleOCR/Docling or mocked provider adapter |
| VLM/LLM server | vLLM OpenAI-compatible server |
| Review UI | lightweight React/Next.js or Streamlit prototype |
| Observability | OpenTelemetry + Prometheus + Grafana |

Example Docker Compose services:

```yaml
services:
  api:
    build: ./services/api
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/docai
      OBJECT_STORE_ENDPOINT: http://minio:9000
      QUEUE_URL: redis://redis:6379
  worker:
    build: ./services/worker
    depends_on: [postgres, redis, minio]
  postgres:
    image: postgres:16
  redis:
    image: redis:7
  minio:
    image: minio/minio
  vlm:
    image: vllm/vllm-openai:latest
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
```

## 4. Production runtime pattern

Separate CPU-bound and GPU-bound work.

CPU-bound:

- ingestion,
- orchestration,
- file normalization,
- validation,
- publishing,
- review API.

GPU/API-bound:

- VLM calls,
- local LLM calls,
- specialized OCR models,
- embedding or visual models if used.

Recommended queue separation:

```text
queue.ingestion
queue.rendering
queue.ocr_layout
queue.classification_result
queue.extraction_cpu
queue.extraction_gpu
queue.validation
queue.review
queue.publishing
```

## 5. Scaling strategy

| Component | Scale by | Notes |
|---|---|---|
| Ingestion API | request count | Stateless. |
| Renderer | page count | CPU/memory heavy. |
| OCR/layout | page count and provider latency | Cache outputs; avoid rerun. |
| VLM/LLM | GPU tokens/images and model concurrency | Use batching and strict timeouts. |
| Validators | document count | Cheap CPU. |
| Review UI/API | reviewer count | Needs low latency. |
| Publisher | event backlog | Idempotent retry. |

## 6. Cloud provider mapping

### 6.1 AWS mapping

| Capability | AWS service option |
|---|---|
| Object storage | S3 |
| Events | EventBridge, SQS, SNS |
| Orchestration | Step Functions, ECS/EKS, Lambda |
| OCR/layout | Amazon Textract |
| Generative document extraction | Amazon Bedrock Data Automation, Bedrock models |
| Human review | SageMaker Ground Truth / Augmented AI pattern |
| Database | DynamoDB, Aurora PostgreSQL |
| Observability | CloudWatch, X-Ray, OpenTelemetry |
| Secrets | Secrets Manager, KMS |

### 6.2 Azure mapping

| Capability | Azure service option |
|---|---|
| Object storage | Azure Blob Storage |
| Events | Event Grid, Service Bus |
| Orchestration | Durable Functions, Container Apps, AKS |
| OCR/layout/extraction/classification | Azure AI Document Intelligence, Azure AI Content Understanding |
| VLM/LLM | Azure AI Foundry / Azure OpenAI / custom containers |
| Human review | Custom review app, Power Platform, or internal workflow |
| Database | Azure SQL, Cosmos DB, PostgreSQL Flexible Server |
| Observability | Azure Monitor, Application Insights |
| Secrets | Key Vault |

### 6.3 GCP mapping

| Capability | GCP service option |
|---|---|
| Object storage | Cloud Storage |
| Events | Pub/Sub, Eventarc |
| Orchestration | Workflows, Cloud Run Jobs, GKE |
| OCR/layout/classification/extraction | Document AI, Custom Splitter, Custom Extractor, Layout Parser |
| VLM/LLM | Vertex AI Gemini, custom model endpoints |
| Human review | Human-in-the-loop via custom app / Vertex AI workflows |
| Database | Cloud SQL, Firestore, BigQuery |
| Observability | Cloud Monitoring, Cloud Trace |
| Secrets | Secret Manager, Cloud KMS |

### 6.4 Local / open-source mapping

| Capability | Option |
|---|---|
| Object storage | MinIO, filesystem for dev |
| Events | Kafka, Redpanda, RabbitMQ, Redis Streams |
| Orchestration | Temporal, Prefect, Celery, Dagster, custom workers |
| OCR/layout | Tesseract, PaddleOCR, Docling, Unstructured, PyMuPDF, layout models |
| VLM/LLM | vLLM OpenAI-compatible server, Ollama for prototypes, custom Triton servers |
| Validation | Python/Pydantic/rule engine |
| Database | PostgreSQL, MongoDB, Elasticsearch/OpenSearch |
| Observability | OpenTelemetry, Prometheus, Grafana, Loki |
| Review UI | Custom web app |

## 7. Observability

### 7.1 Tracing

Trace every document through stages:

```text
document_packet_id -> logical_document_id -> extraction_run_id -> field_id -> evidence_id
```

Use OpenTelemetry trace IDs across services.

### 7.2 Metrics

System metrics:

- documents per minute,
- pages per minute,
- queue lag,
- OCR latency,
- VLM latency,
- validation latency,
- review queue age,
- publish latency,
- cost per page/document,
- GPU utilization,
- provider error rate.

Quality metrics:

- field confidence distribution,
- validation failure rate,
- review rate,
- correction rate,
- downstream rejection rate,
- class/schema/model version comparison.

### 7.3 Logs

Logs should include IDs but not sensitive field values unless explicitly allowed.

Recommended log fields:

- tenant ID,
- document packet ID,
- logical document ID,
- extraction run ID,
- stage,
- schema ID,
- recipe ID,
- model ID,
- duration,
- status,
- error code.

## 8. Security and compliance

### 8.1 Data protection

- encrypt raw and derived artifacts at rest,
- encrypt in transit,
- separate tenant data,
- use least-privilege IAM,
- avoid storing prompts with sensitive values in logs,
- redact sensitive fields from debug output,
- define retention policy for raw documents and evidence crops.

### 8.2 Access control

Roles:

- ingestion client,
- system operator,
- schema administrator,
- reviewer,
- auditor,
- downstream consumer.

Permissions:

- reviewers only see assigned documents,
- schema changes require approval,
- production model changes require release process,
- audit logs are append-only.

### 8.3 Privacy for ID documents

Extra safeguards:

- stricter access control,
- shorter retention for evidence crops where possible,
- no biometric inference,
- explicit purpose limitation,
- audit every view/correction/download.

## 9. Release and versioning

Version everything:

- pipeline version,
- schema version,
- prompt version,
- model version,
- OCR provider version,
- validator version,
- review UI version,
- downstream mapping version.

Release process:

```text
change proposal -> offline regression -> golden-set comparison -> canary traffic -> monitoring -> full rollout or rollback
```

## 10. Performance considerations

Important production lessons:

- OCR/layout may dominate latency for many pipelines.
- GPU-bound model inference should be independently scalable.
- Rendering, OCR, VLM, and validation should run asynchronously.
- Cache reusable artifacts such as rendered pages and OCR output.
- Avoid calling VLMs for fields that deterministic parsers can extract reliably.
- Use page selection to avoid sending all pages to large models when not needed.

## 11. Reliability design

Use:

- idempotency keys,
- retry with exponential backoff,
- dead-letter queues,
- stage-level timeout,
- per-provider circuit breakers,
- fallback providers,
- replay by document/run ID,
- immutable artifacts.

## 12. Disaster recovery

Minimum:

- raw document storage replicated/backed up,
- metadata DB backup and point-in-time recovery,
- schema registry in Git,
- model/prompt/validator versions reproducible,
- ability to replay accepted documents from raw inputs.

## 13. Production readiness checklist

- [ ] Schema registry exists and is versioned.
- [ ] Extraction jobs are idempotent.
- [ ] Raw files and rendered pages are immutable.
- [ ] OCR/layout outputs are cached.
- [ ] VLM/LLM calls use structured output where supported.
- [ ] Every field has evidence.
- [ ] Validation rules are separated from prompts.
- [ ] Review tasks are field-level where possible.
- [ ] Audit log is append-only.
- [ ] Golden-set regression exists.
- [ ] Metrics exist per class, field, schema, and model version.
- [ ] Sensitive logs are redacted.
- [ ] Downstream publishing is idempotent.

