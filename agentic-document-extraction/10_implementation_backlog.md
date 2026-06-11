# 10 — Implementation Backlog

## 1. Recommended delivery approach

Build the system in vertical slices. Each slice should process real documents from ingestion to accepted output, even if the first versions use simple extractors.

Avoid building all infrastructure first and all intelligence later. Instead:

1. define contracts,
2. implement one class end-to-end,
3. add validation and review,
4. add more classes,
5. optimize cost/latency/quality.

## 2. Milestone 0 — Project setup

### Stories

- Create repository structure.
- Create Docker Compose development environment.
- Create object storage, DB, queue, and basic API service.
- Create schema registry folder and loader.
- Define coding standards and artifact naming.

### Acceptance criteria

- developer can run local stack with one command,
- API health check works,
- schema registry can load one class/schema/recipe,
- object storage and DB connection work,
- automated tests run in CI.

## 3. Milestone 1 — Canonical contracts

### Stories

- Implement Pydantic models for document packet, physical page, logical document, extraction run, extracted field, evidence, validation result, review task.
- Implement JSON Schema generation for extraction schemas.
- Implement event envelope and event publisher abstraction.
- Implement idempotency key handling.

### Acceptance criteria

- contracts validate sample JSON,
- invalid extraction output is rejected with readable errors,
- event examples can be serialized/deserialized,
- schema ID and class ID are mandatory everywhere.

## 4. Milestone 2 — Ingestion and rendering

### Stories

- Implement document upload API.
- Store raw document immutably.
- Render PDF pages to images.
- Store page images and metadata.
- Calculate basic page quality metrics.

### Acceptance criteria

- PDF with multiple pages creates physical page records,
- image-only input is accepted,
- unsupported/encrypted file gets controlled error,
- rendered pages are reusable by later stages.

## 5. Milestone 3 — Classification integration

### Stories

- Implement endpoint to receive classification result from previous system.
- Validate logical document page ranges.
- Map classifier labels to canonical class IDs.
- Create extraction jobs from logical documents.
- Route low-confidence classification to review/triage.

### Acceptance criteria

- one packet can create multiple logical documents,
- each logical document has selected class and page refs,
- unsupported class does not start extraction,
- class confidence threshold is enforced.

## 6. Milestone 4 — OCR/layout baseline

### Stories

- Implement OCR/layout provider interface.
- Add one local or cloud OCR provider.
- Store OCR words/lines/blocks/tables with bboxes.
- Build simple evidence lookup by page and text span.

### Acceptance criteria

- OCR output is provider-neutral,
- bboxes are normalized,
- OCR output can be cached and reused,
- failed OCR can be retried or routed to fallback.

## 7. Milestone 5 — Invoice extraction MVP

### Stories

- Define `invoice.v1` class.
- Define `invoice.schema.v1`.
- Define invoice prompt and recipe.
- Implement VLM/LLM structured-output adapter.
- Extract header, parties, totals, and simple line items.
- Implement invoice validators.

### Acceptance criteria

- sample invoices produce valid canonical extraction records,
- required fields are explicit even when missing,
- totals validation works,
- low-confidence total fields trigger review,
- output can be published only after accepted status.

## 8. Milestone 6 — Generic form MVP

### Stories

- Define `generic_form.v1` schema.
- Implement anchor-based field extraction.
- Implement checkbox/selection mark extraction.
- Add VLM fallback for handwritten fields.
- Add conditional required-field validation.

### Acceptance criteria

- printed labels and handwritten values can be extracted,
- checkbox group output is structured,
- ambiguous handwriting creates review tasks,
- review corrections update final record.

## 9. Milestone 7 — ID document MVP

### Stories

- Define ID document classes: `id_card.generic.v1`, `passport.generic.v1`, optionally country-specific subclasses.
- Define ID schema.
- Implement OCR/VLM visible text extraction.
- Implement MRZ parser if applicable.
- Implement visible text vs MRZ consistency checks.
- Add strict security/audit controls.

### Acceptance criteria

- textual ID fields are extracted with evidence,
- MRZ fields are parsed and compared when present,
- inconsistency triggers review,
- no face matching or legal authenticity claim is produced,
- all reviewer access is audited.

## 10. Milestone 8 — Review workflow

### Stories

- Create review task table/API.
- Create simple review UI.
- Show page image, crop, proposed value, alternatives, validation errors.
- Support accept/correct/mark unreadable/not found.
- Write correction and audit entries.

### Acceptance criteria

- review tasks are automatically created by validation policy,
- reviewer can correct fields,
- correction appears in final record,
- audit log shows who changed what and when,
- completed review can trigger final publishing.

## 11. Milestone 9 — Publishing and downstream integration

### Stories

- Implement final record store.
- Implement downstream mapping config.
- Implement callback/event publisher.
- Implement retry/dead-letter handling.
- Add delivery status.

### Acceptance criteria

- accepted invoice can be mapped to downstream payload,
- failed delivery is retried,
- duplicate delivery does not create duplicate business record,
- downstream rejection is captured for feedback.

## 12. Milestone 10 — Evaluation harness

### Stories

- Create golden dataset format.
- Add field-level evaluation metrics.
- Add document-level metrics.
- Add regression comparison between schema/model/prompt versions.
- Add report generation.

### Acceptance criteria

- every release can be tested on golden set,
- field accuracy and review rate are visible per class,
- failed examples are easy to inspect,
- schema/model/prompt changes cannot be promoted without regression report.

## 13. Milestone 11 — Production hardening

### Stories

- Add OpenTelemetry tracing.
- Add Prometheus/Grafana dashboards.
- Add secrets management.
- Add RBAC for reviewer/admin/auditor.
- Add object retention policies.
- Add queue-specific scaling.
- Add dead-letter replay tooling.

### Acceptance criteria

- operator can trace any document by ID,
- no sensitive values appear in normal logs,
- GPU queue and CPU queue can scale independently,
- failed jobs can be replayed,
- retention policy is configurable.

## 14. Milestone 12 — Cloud adapters

### Stories

- Add AWS adapter for S3/SQS/Step Functions/Textract/Bedrock Data Automation pattern.
- Add Azure adapter for Blob/Service Bus/Document Intelligence/Content Understanding pattern.
- Add GCP adapter for Cloud Storage/PubSub/Document AI pattern.
- Keep canonical model unchanged.

### Acceptance criteria

- provider-specific output converts to canonical extraction model,
- same validators and review policy run across providers,
- same downstream contract is used regardless of provider,
- provider can be selected by class recipe.

## 15. Backlog by engineering category

Using the earlier engineering taxonomy:

| Category | Work items |
|---|---|
| Prompt engineering | Class-specific prompts, field descriptions, null-over-guessing instructions, retry prompts, output-format prompts. |
| Context engineering | OCR/layout context selection, page selection, evidence snippets, schema descriptions, few-shot examples, retrieval of class rules. |
| AI engineering | OCR/VLM/LLM model selection, structured outputs, candidate merging, confidence calibration, evaluation, fine-tuning. |
| Harness engineering | APIs, queues, orchestration, storage, review UI, validation service, observability, deployment, security. |

## 16. Definition of done for a document class

A document class is production-ready when:

- class definition exists,
- schema exists and is versioned,
- extraction recipe exists,
- prompt/template exists if using LLM/VLM,
- validators exist,
- thresholds exist,
- review UI instructions exist,
- downstream mapping exists,
- golden set exists,
- metrics dashboard exists,
- regression tests pass,
- rollout plan exists.

## 17. MVP sprint plan

### Sprint 1

- contracts,
- schema registry loader,
- ingestion skeleton,
- storage layout,
- sample classification result.

### Sprint 2

- rendering,
- OCR/layout adapter,
- logical document extraction job creation,
- invoice schema draft.

### Sprint 3

- invoice VLM/LLM extractor,
- structured output validation,
- invoice field wrappers and evidence.

### Sprint 4

- invoice validators,
- review task creation,
- simple review UI/API.

### Sprint 5

- generic form extraction,
- checkbox/handwriting support,
- review corrections.

### Sprint 6

- ID document extraction,
- MRZ parsing,
- text consistency checks,
- security hardening.

### Sprint 7

- downstream publishing,
- evaluation harness,
- dashboards.

### Sprint 8

- cloud/local adapters,
- performance tuning,
- production release candidate.

## 18. Open design questions

These should be resolved during implementation:

1. Which document classes are highest volume and highest business value?
2. What is the acceptable human review rate per class?
3. What is the required SLA per document type?
4. Which fields are legally or financially high risk?
5. Which cloud/provider constraints apply per environment?
6. How long should raw documents and evidence crops be retained?
7. What downstream systems require exact field mappings?
8. Which languages and handwriting styles must be supported first?
9. Is active learning/fine-tuning allowed on reviewed documents?
10. Which fields must never be logged or exported to training?

