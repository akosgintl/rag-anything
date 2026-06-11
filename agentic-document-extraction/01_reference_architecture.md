# 01 — Reference Architecture

## 1. Architecture overview

The system is a class-driven, schema-constrained, evidence-preserving document extraction platform.

It is composed of five large zones:

1. **Input and document preparation** — ingestion, file normalization, page rendering, image quality checks.
2. **Classification/splitting integration** — consume upstream class decisions and logical document boundaries.
3. **Extraction orchestration** — select schema, models, prompts, parsers, OCR/layout context, VLM calls, table extraction, deterministic parsers.
4. **Validation, confidence, review** — type checks, normalization, cross-field validation, confidence calibration, human review.
5. **Publishing, feedback, operations** — final canonical records, downstream APIs/events, training feedback, monitoring, evaluation.

```mermaid
flowchart TB
    subgraph Input[Input and preparation]
        A1[Upload API / scanner / email / batch import]
        A2[Object storage: raw documents]
        A3[File normalizer]
        A4[Page renderer]
        A5[Image quality and orientation]
    end

    subgraph Classify[Existing classification and splitting]
        B1[Page/document classifier]
        B2[Logical document splitter]
        B3[Class decision + confidence]
    end

    subgraph Registry[Control plane]
        C1[Document class registry]
        C2[Extraction schema registry]
        C3[Prompt/model recipe registry]
        C4[Validation rules registry]
        C5[Threshold policy registry]
    end

    subgraph Extract[Extraction runtime]
        D1[Extraction orchestrator]
        D2[OCR/layout service]
        D3[VLM extractor]
        D4[LLM structured extractor]
        D5[Table extractor]
        D6[Barcode/MRZ/parser service]
        D7[Field merger]
    end

    subgraph Quality[Quality and review]
        E1[Schema validator]
        E2[Normalizer]
        E3[Business rule validator]
        E4[Confidence/risk scorer]
        E5[Human review queue]
    end

    subgraph Output[Publishing and feedback]
        F1[Canonical extraction record]
        F2[Downstream event/API]
        F3[Operational database]
        F4[Data warehouse/lake]
        F5[Training/evaluation feedback]
    end

    A1 --> A2 --> A3 --> A4 --> A5 --> B1
    B1 --> B2 --> B3 --> D1
    Registry --> D1
    D1 --> D2
    D1 --> D3
    D1 --> D4
    D1 --> D5
    D1 --> D6
    D2 --> D7
    D3 --> D7
    D4 --> D7
    D5 --> D7
    D6 --> D7
    D7 --> E1 --> E2 --> E3 --> E4
    E4 -->|accepted| F1
    E4 -->|needs review| E5 --> F1
    F1 --> F2
    F1 --> F3
    F1 --> F4
    E5 --> F5
    F1 --> F5
```

## 2. Logical layers

### 2.1 Control plane

The control plane defines what the system is allowed to extract and how.

It contains:

- document class taxonomy,
- class aliases and versioning,
- field schemas,
- field descriptions,
- prompt templates,
- extraction recipes,
- validation policies,
- confidence thresholds,
- review policy,
- downstream mapping definitions.

This should be configuration-driven and reviewed like source code.

### 2.2 Data plane

The data plane processes documents.

It contains:

- ingestion APIs,
- storage adapters,
- queues,
- OCR/layout workers,
- VLM/LLM workers,
- extraction workers,
- validation workers,
- review task creation,
- publishing workers.

### 2.3 Feedback plane

The feedback plane turns operational usage into measurable improvement.

It contains:

- human corrections,
- false positives/false negatives,
- field-level accuracy metrics,
- class-level accuracy metrics,
- golden datasets,
- regression test suites,
- prompt/model/schema experiments,
- active learning candidate selection.

## 3. Component relationship to classification

Classification is not just a label; it is an extraction contract selector.

```mermaid
sequenceDiagram
    participant Ingest as Ingestion
    participant Prep as Preparation
    participant Cls as Classifier/Splitter
    participant Reg as Schema Registry
    participant Orch as Extraction Orchestrator
    participant Ext as Extractors
    participant Val as Validation
    participant Rev as Review
    participant Pub as Publisher

    Ingest->>Prep: raw document
    Prep->>Cls: page images + OCR/layout hints
    Cls-->>Orch: logical docs + class + confidence
    Orch->>Reg: get schema/recipe for class + version
    Reg-->>Orch: fields, descriptions, validators, thresholds
    Orch->>Ext: extract according to class recipe
    Ext-->>Orch: candidate fields + evidence
    Orch->>Val: validate, normalize, reconcile
    Val-->>Rev: low-confidence/rule-failing fields
    Rev-->>Val: corrected values
    Val-->>Pub: accepted canonical extraction record
```

## 4. Recommended runtime decomposition

| Service | Scaling pattern | State | Notes |
|---|---:|---|---|
| Ingestion API | horizontal CPU | stateless | Handles uploads, callbacks, idempotency. |
| Document store | managed storage | stateful | Raw files, rendered pages, evidence crops. |
| Job orchestrator | horizontal CPU | DB-backed | Owns state machine and retries. |
| OCR/layout workers | CPU/GPU depending provider | stateless/cache | Often high-latency and reusable. |
| VLM workers | GPU | stateless | Batch page images, control concurrency. |
| LLM structured workers | GPU/API | stateless | Use schema-constrained JSON output. |
| Parser workers | CPU | stateless | MRZ, barcode, QR, checksums, regex, arithmetic. |
| Validator workers | CPU | stateless | Deterministic rules and score aggregation. |
| Review API/UI | horizontal CPU | DB-backed | Human correction and audit trail. |
| Publisher | horizontal CPU | DB/event-backed | Downstream mappings and delivery guarantees. |
| Evaluation service | batch/async | stateful | Golden set tests, drift metrics, replay. |

## 5. Recommended storage model

```text
s3://doc-ai/raw/{tenant}/{yyyy}/{mm}/{document_packet_id}/input.pdf
s3://doc-ai/rendered/{tenant}/{document_packet_id}/page-0001.png
s3://doc-ai/ocr/{tenant}/{document_packet_id}/ocr.json
s3://doc-ai/classification/{tenant}/{document_packet_id}/classification.json
s3://doc-ai/extraction-runs/{tenant}/{run_id}/candidates.json
s3://doc-ai/evidence/{tenant}/{run_id}/{field_id}/crop.png
s3://doc-ai/final/{tenant}/{logical_document_id}/extraction.json
```

A relational database or document database stores metadata, current state, indexes, review tasks, and downstream delivery status. Object storage stores large immutable artifacts.

## 6. Why not one end-to-end model only?

A single VLM prompt can work for demos, but production extraction needs:

- repeatability,
- schema versioning,
- deterministic validation,
- evidence traceability,
- per-field confidence,
- route-specific cost control,
- low-risk human review,
- reprocessing and audit.

Therefore, the recommended architecture is hybrid: OCR/layout + VLM/LLM + deterministic parsers + validators + review.

## 7. Architecture variants

### Variant A — OCR-first, LLM extraction

Best for high-volume printed documents where OCR is reliable.

Flow:

```text
PDF/image -> OCR/layout -> class schema -> LLM over OCR text/layout -> validators -> review
```

### Variant B — VLM-first extraction

Best for visually complex documents, handwritten forms, IDs, tables, and documents where OCR quality is weak.

Flow:

```text
page image -> VLM with schema + field descriptions -> evidence crops -> validators -> review
```

### Variant C — Hybrid ensemble

Best for high-risk documents.

Flow:

```text
OCR/layout + VLM + parser outputs -> field-level candidate merger -> validators -> review
```

### Variant D — Cloud-native managed IDP

Best when platform speed matters more than local model control.

Flow:

```text
Azure / AWS / GCP managed extractor -> canonical adapter -> validation -> review -> downstream
```

## 8. Recommended default

Use **Variant C** for the platform design, with per-class recipes choosing cheaper/simpler variants where enough.

Examples:

| Class | Default recipe |
|---|---|
| `invoice.v1` | OCR/layout + VLM/LLM + arithmetic validator + VAT/IBAN parsers. |
| `generic_form.v1` | OCR/layout + field-anchor extraction + VLM fallback for handwritten fields. |
| `id_document.v1` | VLM + OCR + MRZ/barcode parser + text consistency checks. |
| `receipt.v1` | OCR + table/line-item parser + VLM fallback. |
| `contract.v1` | OCR/layout + section chunking + schema-constrained LLM extraction. |

