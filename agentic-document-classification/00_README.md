# Modern Document Classification Solution — 2026 Design Pack

This package designs a modern enterprise document classification solution suitable for scanned PDFs, native PDFs, Office files, email attachments, images, and multi-document packets. The design assumes production requirements: auditability, confidence-based automation, human review, model lifecycle management, cloud portability, and integration with downstream extraction / case-management / ECM systems.

## What the system should do

The system receives an unknown document or document packet and decides:

1. **What each page is** — page-level class, page quality, language, layout traits.
2. **Where documents start and end** — document splitting for packets containing multiple documents.
3. **What each document is** — canonical document class, sub-class, business routing class.
4. **How confident the system is** — calibrated confidence, uncertainty, explanation, and risk decision.
5. **What should happen next** — auto-route, send to extractor, ask for human review, reject, or quarantine.

The intended output is not only a label. It is a **classification decision object** with evidence, lineage, confidence, thresholds, and routing instructions.

## File map

| File | Purpose |
|---|---|
| `01_architecture.md` | End-to-end architecture, main components, relationships, and Mermaid diagrams. |
| `02_components.md` | Detailed component catalog with responsibilities, inputs, outputs, and design decisions. |
| `03_data_model.md` | Canonical data model and JSON examples for documents, pages, layout, predictions, decisions, reviews, labels, and events. |
| `04_data_flow.md` | Runtime flow from ingestion to routing, plus training / feedback / monitoring flows. |
| `05_model_strategy.md` | Model architecture choices: rules, text classifiers, layout-aware models, OCR-free models, VLM/LLM fallback, ensembles, calibration. |
| `06_implementation_plan.md` | Step-by-step implementation plan, milestones, backlog, acceptance criteria, and MVP-to-production roadmap. |
| `07_deployment_ops.md` | Docker-based development, production Kubernetes, and AWS/Azure/GCP deployment mapping. |
| `08_governance_evaluation.md` | Security, privacy, audit, risk controls, evaluation metrics, thresholds, test sets, and monitoring. |
| `09_references.md` | Source list and reading notes. |

## Recommended architecture in one paragraph

Build a **hybrid, event-driven Intelligent Document Processing classifier**. Persist raw documents immutably, normalize them into pages, extract text/layout/images, run multiple classification strategies, fuse their results, calibrate confidence, apply policy thresholds, and route only high-confidence low-risk cases automatically. Use human review for uncertain, novel, sensitive, or high-impact documents. Feed reviewer decisions back into a versioned training dataset and retrain/evaluate models in controlled promotion stages.

## Design principles

- **Classification is a decision, not just a model output.** Store predictions separately from business decisions.
- **Page-level classification comes before document-level classification.** Many enterprise inputs are packets.
- **Use layout and image information, not OCR text alone.** Modern document type classification benefits from text, layout, visual structure, and metadata.
- **Make uncertainty first-class.** Confidence, margin, entropy, OOD score, evidence, and threshold policy should all be explicit.
- **Route by risk.** A low-value marketing flyer can be auto-routed at lower certainty than a legal notice, insurance claim, or KYC document.
- **Keep raw, normalized, extracted, and predicted data separate.** This supports reprocessing and audit.
- **Design for human feedback from day one.** Review queues, label corrections, and active learning are core system components.
- **Avoid vendor lock-in at the domain model layer.** Cloud OCR/document AI providers can be adapters behind the same canonical schema.

## Suggested first MVP

Start with 10–20 document classes, 500–2,000 labeled examples if available, and a strict review policy. Use OCR + text classifier + layout-aware classifier + rule layer. Add VLM/LLM classification only as fallback or second-opinion until cost, latency, and hallucination risk are measured.

## Target outputs

Every processed package should produce:

- `DocumentPackage` record
- immutable raw object reference
- page records with rendered image references
- OCR/layout result
- page-level predictions
- document segment predictions
- document-level decision
- route action
- audit event chain
- review task when thresholds are not met

## Sources consulted

The design is intentionally vendor-neutral, but it reflects current 2025–2026 capabilities from major cloud and research sources:

- Microsoft Azure AI Document Intelligence custom classifier: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-classifier?view=doc-intel-4.0.0
- Google Cloud Document AI custom classifier: https://docs.cloud.google.com/document-ai/docs/custom-classifier
- Amazon Textract overview: https://aws.amazon.com/textract/
- Amazon Textract Analyze Lending classification/extraction: https://docs.aws.amazon.com/textract/latest/dg/lending-document-classification-extraction.html
- Amazon Comprehend custom classification: https://docs.aws.amazon.com/comprehend/latest/dg/how-document-classification.html
- AWS Bedrock Data Automation / IDP concepts: https://docs.aws.amazon.com/bedrock/latest/userguide/bda.html
- LayoutLMv3 paper: https://arxiv.org/abs/2204.08387
- Donut OCR-free Document Understanding Transformer paper: https://arxiv.org/abs/2111.15664
- 2026 multimodal document type classification comparison: https://arxiv.org/abs/2606.02162
- RVL-CDIP dataset: https://adamharley.com/rvl-cdip/

