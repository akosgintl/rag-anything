# Modern Document Content / Field Extraction System Design (2026)

This design pack describes a modern document content extraction platform for invoices, forms, ID documents, handwritten documents, printed documents, scanned PDFs, and multi-document packets.

The design assumes the upstream **document classification / splitting solution already exists**. Classification determines the logical document boundaries and document class. Extraction then uses the class to select a field schema, extraction model, validation rules, confidence thresholds, and human review workflow.

## Files

| File | Purpose |
|---|---|
| `00_plan.md` | Delivery plan, assumptions, milestones, and implementation sequence. |
| `01_reference_architecture.md` | Main architecture, component relationships, and Mermaid diagrams. |
| `02_component_model.md` | Detailed component catalogue and responsibilities. |
| `03_data_model.md` | Canonical data model that should flow through the system. |
| `04_end_to_end_data_flow.md` | Step-by-step data flow from ingestion to downstream publishing. |
| `05_class_based_schema_registry.md` | How fields depend on document class; schema registry, versions, prompts, validators. |
| `06_extraction_patterns.md` | Extraction strategies for invoices, forms, IDs, handwriting, tables, and long documents. |
| `07_validation_quality_human_review.md` | Validation, confidence, evidence, reconciliation, review queues, auditability. |
| `08_deployment_and_operations.md` | Docker/dev, production, cloud-provider mapping, observability, security, scaling. |
| `09_api_events_and_contracts.md` | APIs, event contracts, JSON examples, idempotency, state machine. |
| `10_implementation_backlog.md` | Practical phased backlog and acceptance criteria. |

## Design principles

1. **Class-first extraction**: never use one universal extraction prompt for all documents. Route by document class and schema version.
2. **Schema-constrained output**: extraction outputs must conform to a typed schema with value, confidence, evidence, bounding region, normalization status, and validation status.
3. **Evidence over belief**: every extracted field should point to page, region, OCR span, or visual crop evidence.
4. **Hybrid extraction**: combine OCR/layout, VLM, LLM, deterministic parsers, table extraction, barcode/MRZ parsers, and rule engines.
5. **Human review by exception**: route only uncertain, high-risk, or rule-failing fields to review.
6. **Version everything**: document class, schema, prompt, model, validator, extraction run, and human correction.
7. **Auditable, replayable pipeline**: every stage stores inputs, outputs, metadata, timings, and model versions.
8. **Cloud portable core**: keep canonical contracts cloud-neutral; implement adapters for AWS, Azure, GCP, and local open-source deployments.

## Main source references used while preparing the design

- Azure AI Document Intelligence model overview and custom extraction/classification/composed models: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/model-overview?view=doc-intel-4.0.0
- Azure AI Document Intelligence docs landing page: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/?view=doc-intel-4.0.0
- Google Document AI custom extractor mechanisms: https://docs.cloud.google.com/document-ai/docs/ce-mechanisms
- Google Document AI custom splitter: https://docs.cloud.google.com/document-ai/docs/custom-splitter
- AWS Bedrock Data Automation user guide: https://docs.aws.amazon.com/bedrock/latest/userguide/bda.html
- AWS human review with Bedrock Data Automation and SageMaker AI: https://aws.amazon.com/blogs/machine-learning/process-multi-page-documents-with-human-review-using-amazon-bedrock-data-automation-and-amazon-sagemaker-ai/
- vLLM structured outputs: https://docs.vllm.ai/en/stable/features/structured_outputs/
- vLLM OpenAI-compatible server: https://docs.vllm.ai/en/v0.14.0/serving/openai_compatible_server/
- Operationalizing Document AI: Microservice Architecture for OCR and LLM Pipelines in Production, arXiv 2026: https://arxiv.org/abs/2605.18818
- Schema-constrained AI extraction and provenance-aware extraction, arXiv 2026: https://arxiv.org/abs/2601.14267

