# 09 — APIs, Events, and Contracts

## 1. API design principles

- APIs must be idempotent.
- Large files should be stored in object storage; APIs pass metadata/URIs.
- Every state change should emit an event.
- All events should include schema version, tenant ID, correlation ID, and timestamps.
- Internal provider formats must be converted into canonical contracts.
- Final output should be stable even if providers change.

## 2. Core identifiers

| Identifier | Meaning |
|---|---|
| `tenant_id` | Customer/business partition. |
| `document_packet_id` | Original uploaded package/file. |
| `physical_page_id` | Rendered page from original package. |
| `logical_document_id` | Classified/split business document. |
| `classification_run_id` | Upstream classification run. |
| `schema_id` | Extraction schema selected by document class. |
| `recipe_id` | Runtime extraction recipe. |
| `extraction_run_id` | Single extraction attempt. |
| `field_id` / `field_path` | Field within selected schema. |
| `evidence_id` | Evidence crop/span/region supporting a field. |
| `review_task_id` | Human review task. |
| `correction_id` | Human correction. |

## 3. API endpoints

### 3.1 Submit document packet

`POST /v1/document-packets`

Request:

```json
{
  "tenant_id": "tenant_a",
  "source_type": "api_upload",
  "external_reference": "case-12345",
  "file_name": "invoice.pdf",
  "mime_type": "application/pdf",
  "file_base64": "...",
  "class_hint": null,
  "priority": "normal",
  "callback_url": "https://example.com/callbacks/docai"
}
```

Response:

```json
{
  "document_packet_id": "pkt_2026_000001",
  "status": "received",
  "tracking_url": "/v1/document-packets/pkt_2026_000001"
}
```

Large-file alternative:

1. `POST /v1/document-packets/upload-url`
2. client uploads directly to object storage,
3. `POST /v1/document-packets/from-uri`.

### 3.2 Receive classification result from upstream system

`POST /v1/classification-results`

Request:

```json
{
  "tenant_id": "tenant_a",
  "document_packet_id": "pkt_2026_000001",
  "classification_run_id": "clsrun_123",
  "logical_documents": [
    {
      "logical_document_id": "ldoc_0001",
      "page_numbers": [1, 2],
      "selected_class": "invoice.v1",
      "confidence": 0.94,
      "candidate_classes": [
        {"class_id": "invoice.v1", "confidence": 0.94},
        {"class_id": "receipt.v1", "confidence": 0.04}
      ],
      "requires_review": false
    }
  ]
}
```

Response:

```json
{
  "accepted": true,
  "created_extraction_jobs": [
    {"logical_document_id": "ldoc_0001", "job_id": "job_001"}
  ]
}
```

### 3.3 Get extraction status

`GET /v1/logical-documents/{logical_document_id}/extraction-status`

Response:

```json
{
  "logical_document_id": "ldoc_0001",
  "class_id": "invoice.v1",
  "status": "needs_review",
  "extraction_run_id": "extrun_2026_000001",
  "review_tasks_open": 2,
  "updated_at": "2026-06-08T10:20:00Z"
}
```

### 3.4 Get final extraction result

`GET /v1/logical-documents/{logical_document_id}/extraction-result`

Response:

```json
{
  "logical_document_id": "ldoc_0001",
  "document_packet_id": "pkt_2026_000001",
  "class_id": "invoice.v1",
  "schema_id": "invoice.schema.v1",
  "status": "accepted_after_review",
  "fields": {},
  "tables": [],
  "document_validations": [],
  "review_summary": {
    "review_required": true,
    "review_completed": true,
    "corrected_fields": 1
  }
}
```

### 3.5 List review tasks

`GET /v1/review-tasks?tenant_id=tenant_a&status=pending`

Response:

```json
{
  "items": [
    {
      "review_task_id": "rev_001",
      "logical_document_id": "ldoc_0001",
      "class_id": "invoice.v1",
      "field_path": "totals.total_gross",
      "priority": "high",
      "reason": "High-risk field below threshold",
      "created_at": "2026-06-08T10:20:00Z"
    }
  ]
}
```

### 3.6 Submit human correction

`POST /v1/review-tasks/{review_task_id}/corrections`

Request:

```json
{
  "action": "correct",
  "new_value": {
    "raw": "1245.00 EUR",
    "normalized": {"amount": "1245.00", "currency": "EUR"},
    "type": "money"
  },
  "presence": "present",
  "comment": "Corrected from visual crop."
}
```

Response:

```json
{
  "review_task_id": "rev_001",
  "status": "completed",
  "correction_id": "corr_001"
}
```

## 4. Event envelope

All events should use a common envelope.

```json
{
  "event_id": "evt_001",
  "event_type": "ExtractionAccepted",
  "event_version": "1.0",
  "tenant_id": "tenant_a",
  "correlation_id": "corr_doc_001",
  "causation_id": "evt_previous",
  "occurred_at": "2026-06-08T10:30:00Z",
  "payload": {}
}
```

## 5. Event types

Recommended event types:

| Event | Producer | Consumer |
|---|---|---|
| `DocumentPacketCreated` | Ingestion API | normalizer/renderer |
| `DocumentPacketRendered` | Renderer | classifier, extraction preparation |
| `LogicalDocumentClassified` | Classification system | extraction orchestrator |
| `ExtractionRunCreated` | Orchestrator | OCR/extraction workers |
| `OcrLayoutCompleted` | OCR service | extraction workers |
| `ExtractionCandidatesCreated` | Extractor | validator |
| `ExtractionValidationCompleted` | Validator | review router/publisher |
| `ReviewTaskCreated` | Review router | review UI |
| `ReviewTaskCompleted` | Review UI | finalizer |
| `ExtractionAccepted` | Finalizer | downstream publisher |
| `ExtractionPublished` | Publisher | audit/monitoring |
| `ExtractionFailed` | Any stage | operator/dead-letter handler |

## 6. Event examples

### 6.1 `ExtractionRunCreated`

```json
{
  "event_type": "ExtractionRunCreated",
  "event_version": "1.0",
  "tenant_id": "tenant_a",
  "payload": {
    "extraction_run_id": "extrun_2026_000001",
    "logical_document_id": "ldoc_0001",
    "document_packet_id": "pkt_2026_000001",
    "class_id": "invoice.v1",
    "schema_id": "invoice.schema.v1",
    "recipe_id": "invoice.hybrid.v1"
  }
}
```

### 6.2 `ExtractionValidationCompleted`

```json
{
  "event_type": "ExtractionValidationCompleted",
  "event_version": "1.0",
  "payload": {
    "extraction_run_id": "extrun_2026_000001",
    "logical_document_id": "ldoc_0001",
    "status": "needs_review",
    "field_summary": {
      "total_fields": 42,
      "accepted_fields": 39,
      "review_fields": 3,
      "failed_fields": 0
    },
    "document_validations": [
      {"rule_id": "totals_reconciliation", "status": "passed"}
    ]
  }
}
```

### 6.3 `ExtractionAccepted`

```json
{
  "event_type": "ExtractionAccepted",
  "event_version": "1.0",
  "payload": {
    "logical_document_id": "ldoc_0001",
    "class_id": "invoice.v1",
    "schema_id": "invoice.schema.v1",
    "status": "accepted_after_review",
    "final_record_uri": "s3://doc-ai/final/tenant_a/ldoc_0001/extraction.json",
    "review_summary": {
      "corrected_fields": 1,
      "reviewer_count": 1
    }
  }
}
```

## 7. Idempotency

Use idempotency keys for:

- document submission,
- classification result submission,
- extraction job creation,
- review correction submission,
- downstream publishing.

Example key:

```text
tenant_id + external_reference + file_sha256
```

For extraction runs:

```text
logical_document_id + class_id + schema_id + recipe_id + pipeline_version
```

## 8. Error contract

```json
{
  "error": {
    "code": "SCHEMA_NOT_FOUND",
    "message": "No active extraction schema found for class invoice.v3.",
    "retryable": false,
    "details": {
      "class_id": "invoice.v3"
    }
  }
}
```

Recommended error codes:

- `UNSUPPORTED_FILE_TYPE`,
- `ENCRYPTED_FILE`,
- `RENDERING_FAILED`,
- `CLASSIFICATION_RESULT_INVALID`,
- `SCHEMA_NOT_FOUND`,
- `EXTRACTION_TIMEOUT`,
- `STRUCTURED_OUTPUT_INVALID`,
- `VALIDATION_FAILED`,
- `REVIEW_REQUIRED`,
- `DOWNSTREAM_PUBLISH_FAILED`.

## 9. Provider adapter contract

Every provider adapter should return canonical output.

```python
class OcrLayoutProvider:
    def analyze(self, pages: list[str]) -> OcrLayoutResult:
        ...

class StructuredExtractorProvider:
    def extract(
        self,
        schema: dict,
        pages: list[str],
        ocr_context: dict | None,
        prompt: str,
    ) -> dict:
        ...

class ParserProvider:
    def parse(self, parser_name: str, text_or_image: object) -> dict:
        ...
```

Provider-specific details stay behind adapters.

## 10. Downstream publishing contract

Downstream publisher should support:

- full result delivery,
- business-object-specific delivery,
- partial delivery after review,
- retry and dead-letter,
- delivery receipt.

Example:

```json
{
  "delivery_id": "del_001",
  "target_system": "erp_invoice_import",
  "logical_document_id": "ldoc_0001",
  "status": "delivered",
  "attempt": 1,
  "delivered_at": "2026-06-08T10:31:00Z"
}
```

