# 06 — Implementation Plan

## 1. Delivery strategy

Build the solution in increments. The goal is not to build the most advanced model first. The goal is to build a reliable classification product with audit, review, and feedback loops, then improve model sophistication safely.

Recommended delivery phases:

1. Discovery and taxonomy design.
2. Data foundation and ingestion.
3. OCR/layout and normalization.
4. Baseline classifier and review workflow.
5. Layout-aware classifier and packet splitting.
6. Calibration, thresholds, and decision policy.
7. MLOps, monitoring, and retraining.
8. Cloud hardening and production rollout.
9. Advanced VLM/LLM fallback and active learning.

## 2. Phase 0 — Discovery and scoping

### Goals

- Define business scope.
- Identify document classes.
- Understand source channels.
- Define risk levels and auto-routing policy.
- Collect sample documents.

### Tasks

- Interview business owners and operations users.
- Inventory input channels and downstream workflows.
- Create initial class taxonomy.
- Identify confusable classes.
- Define success metrics.
- Define privacy/retention constraints.
- Estimate volume, page counts, SLA, and peak load.
- Collect representative examples.

### Deliverables

- Class taxonomy v0.1.
- Source system inventory.
- Risk and routing matrix.
- Data availability report.
- Non-functional requirement list.

### Acceptance criteria

- At least the top 10–20 classes are clearly defined.
- Each class has examples and negative examples.
- Business owners agree on what should be auto-routed vs reviewed.
- Security/privacy constraints are documented.

## 3. Phase 1 — Data foundation

### Goals

- Create a reliable ingestion and storage foundation.
- Make every document traceable.
- Prepare for reprocessing and audit.

### Tasks

- Implement `DocumentPackage` registry table.
- Implement object store structure.
- Implement ingestion API.
- Implement idempotency and hash-based duplicate detection.
- Implement malware scan adapter.
- Implement event bus topics.
- Implement status state machine.
- Add basic admin/status API.

### Deliverables

- Running ingestion service.
- Raw object storage.
- Registry database.
- Event contracts.
- Basic status UI or API.

### Acceptance criteria

- A document can be submitted and traced by `package_id`.
- Raw file is stored immutably.
- Duplicate and malware scan results are recorded.
- Processing events are persisted.

## 4. Phase 2 — Normalization and OCR/layout

### Goals

- Convert all supported file formats into canonical pages.
- Extract text and layout in a provider-neutral format.

### Tasks

- Implement normalizer for PDF, image, TIFF, Office files, and email body if required.
- Render page images and thumbnails.
- Add page quality analyzer.
- Integrate OCR/layout provider.
- Map provider output to canonical schema.
- Store OCR/layout JSON artifacts.
- Index OCR text for search/debug.

### Deliverables

- Normalization service.
- OCR/layout adapter.
- Page and OCR artifact schema.
- Reviewable page thumbnails.

### Acceptance criteria

- Supported inputs produce page records and page images.
- OCR/layout output contains text, confidence, and bounding boxes.
- Poor quality and blank pages are detected.
- The pipeline can be replayed for a package.

## 5. Phase 3 — Baseline classification MVP

### Goals

- Build a first measurable classifier.
- Route only safe high-confidence cases.
- Send uncertain cases to review.

### Tasks

- Create training dataset manifest from available labels.
- Implement rule/metadata classifier.
- Implement TF-IDF + logistic regression or similar text classifier.
- Implement candidate prediction schema.
- Implement simple fusion.
- Implement thresholds by class/risk.
- Implement review queue creation.
- Implement manual correction UI/API.
- Store review labels.

### Deliverables

- Baseline classifier service.
- Decision policy v0.1.
- Review task workflow.
- Initial evaluation report.

### Acceptance criteria

- System produces final `ClassificationDecision` objects.
- Uncertain cases go to review.
- Reviewer corrections create training labels.
- Baseline metrics are known by class.

## 6. Phase 4 — Layout-aware classification

### Goals

- Improve accuracy for visually rich documents.
- Add page-level classification.
- Support document packet splitting.

### Tasks

- Prepare layout-aware model input from OCR tokens and boxes.
- Fine-tune or configure layout-aware model.
- Build page classifier.
- Implement sequence-based packet splitter.
- Add segment-level classification.
- Store page and segment predictions separately.
- Compare layout-aware model against baseline.

### Deliverables

- Page classifier.
- Segmenter.
- Layout-aware document classifier.
- Evaluation report with confusion matrix.

### Acceptance criteria

- Packet splitting works on representative multi-document PDFs.
- Layout-aware classifier improves target metrics on structured classes.
- Mis-split and misclassify examples are reviewable and labeled.

## 7. Phase 5 — Calibration and policy hardening

### Goals

- Make confidence usable for automation.
- Reduce false auto-routes.
- Implement class-specific thresholds.

### Tasks

- Create held-out calibration set.
- Apply temperature scaling or isotonic calibration.
- Evaluate reliability curves.
- Define thresholds by class/risk/source.
- Add OOD score and model disagreement triggers.
- Implement policy simulation on historical data.
- Add manual-only classes.

### Deliverables

- Calibrated prediction output.
- Threshold policy file.
- Calibration report.
- Business policy simulation.

### Acceptance criteria

- Accuracy by confidence bucket is measured.
- False auto-route rate is within agreed limit.
- High-risk classes cannot bypass policy.
- Policy changes are versioned.

## 8. Phase 6 — Production MLOps

### Goals

- Make training and deployment reproducible.
- Monitor model quality and drift.
- Enable safe model promotion.

### Tasks

- Implement dataset snapshot builder.
- Implement training pipeline.
- Implement model registry integration.
- Add model card generation.
- Add staging, shadow, canary, production stages.
- Implement online monitoring dashboards.
- Add alerting for drift and review-rate spikes.
- Add reprocessing capability.

### Deliverables

- Automated training pipeline.
- Model registry and promotion workflow.
- Monitoring dashboards.
- Drift alerts.

### Acceptance criteria

- Any production decision can be tied to model/rule/taxonomy/policy version.
- New model versions can be shadow tested.
- Production can roll back to previous model.
- Drift and review rate are visible.

## 9. Phase 7 — Advanced modern capabilities

### Goals

- Add cost-aware VLM/LLM fallback.
- Improve unknown-class handling.
- Accelerate learning from review feedback.

### Tasks

- Implement VLM/LLM classifier adapter with strict JSON schema.
- Add cost and rate-limit controls.
- Add active learning sample selector.
- Add nearest-neighbor similarity search for reviewer evidence.
- Add unknown-class clustering dashboard.
- Add reviewer copilot evidence summaries.
- Add synthetic/weak label workflow with human verification.

### Deliverables

- VLM fallback service.
- Active learning pipeline.
- Novelty queue.
- Reviewer assist features.

### Acceptance criteria

- VLM fallback runs only when policy allows it.
- Cost per page remains within budget.
- Unknown documents are clustered and triaged.
- Active learning improves target classes over time.

## 10. Work breakdown structure

### Epic A — Platform foundation

| Story | Description |
|---|---|
| A1 | Create registry schema and state machine. |
| A2 | Implement object storage abstraction. |
| A3 | Implement event bus contracts. |
| A4 | Implement ingestion API. |
| A5 | Implement connector framework. |
| A6 | Implement malware scan adapter. |

### Epic B — Document processing

| Story | Description |
|---|---|
| B1 | Render PDF/image/TIFF to pages. |
| B2 | Convert Office files to normalized PDF/pages. |
| B3 | Generate thumbnails. |
| B4 | Analyze page quality. |
| B5 | Integrate OCR/layout provider. |
| B6 | Map OCR/layout to canonical schema. |

### Epic C — Classification

| Story | Description |
|---|---|
| C1 | Implement taxonomy service. |
| C2 | Implement rule classifier. |
| C3 | Train baseline text classifier. |
| C4 | Implement layout-aware classifier. |
| C5 | Implement page classifier. |
| C6 | Implement packet segmenter. |
| C7 | Implement fusion/calibration. |
| C8 | Implement VLM fallback adapter. |

### Epic D — Review and feedback

| Story | Description |
|---|---|
| D1 | Create review task service. |
| D2 | Build review UI with thumbnails. |
| D3 | Add OCR/layout overlay. |
| D4 | Add class correction and split correction. |
| D5 | Store review result and training label. |
| D6 | Add active learning queue. |

### Epic E — MLOps and operations

| Story | Description |
|---|---|
| E1 | Dataset snapshot pipeline. |
| E2 | Training job pipeline. |
| E3 | Evaluation and model card. |
| E4 | Model registry. |
| E5 | Shadow/canary deployment. |
| E6 | Monitoring dashboards. |
| E7 | Drift alerts. |

### Epic F — Integration

| Story | Description |
|---|---|
| F1 | Downstream routing adapter. |
| F2 | ECM metadata writeback. |
| F3 | Case/workflow system integration. |
| F4 | Extraction processor routing. |
| F5 | Reprocessing API. |

## 11. Suggested timeline

### 12-week delivery plan

| Week | Focus | Outcome |
|---:|---|---|
| 1 | Discovery, taxonomy, architecture | agreed scope and classes |
| 2 | Data model, storage, registry | core schema and object layout |
| 3 | Ingestion API, event bus | documents can enter system |
| 4 | Normalization and page rendering | canonical pages available |
| 5 | OCR/layout integration | text/layout artifacts available |
| 6 | Baseline rules/text classifier | first classification output |
| 7 | Review workflow | uncertain docs can be corrected |
| 8 | Layout-aware model | stronger structured classification |
| 9 | Packet splitting | multi-document PDFs supported |
| 10 | Calibration and thresholds | confidence-based decisions |
| 11 | Monitoring and MLOps | dashboards and model registry |
| 12 | Pilot rollout | controlled production pilot |

### 6-month production hardening roadmap

| Month | Focus |
|---:|---|
| 1 | MVP platform and baseline classifier. |
| 2 | Review UI and feedback loop. |
| 3 | Layout-aware classifier and packet splitting. |
| 4 | Calibration, MLOps, monitoring, model registry. |
| 5 | Production integration, security hardening, performance tuning. |
| 6 | VLM fallback, active learning, unknown-class management, multi-cloud adapters. |

## 12. MVP scope recommendation

### Include in MVP

- PDF/image ingestion.
- Raw object storage.
- Registry and status API.
- OCR/layout extraction.
- 10–20 document classes.
- Rule + text classifier.
- Layout-aware classifier if labeled data is available.
- Human review queue.
- Final decision object.
- Simple downstream export.
- Evaluation report.

### Exclude from MVP unless required

- Full multi-cloud deployment.
- VLM fallback for every document.
- Fully automated retraining.
- Complex active learning.
- Advanced explainability dashboards.
- Multi-language optimization beyond current data.
- Fine-grained extraction after classification.

## 13. Acceptance criteria by capability

| Capability | Acceptance criterion |
|---|---|
| Ingestion | Every file receives `package_id`, raw object ref, and status. |
| Normalization | Supported files produce pages and thumbnails. |
| OCR/layout | OCR tokens include text, bbox, confidence, and page id. |
| Classification | Every segment receives top-k predictions and final decision. |
| Confidence | Decision records include thresholds and calibrated confidence. |
| Review | Reviewer can correct class and split; correction is stored. |
| Audit | Decision can be reconstructed with model/rule/taxonomy versions. |
| Routing | Auto-routed documents reach the correct downstream endpoint. |
| Monitoring | Latency, error rate, review rate, and class distribution are visible. |
| MLOps | Model versions are registered and rollbacks are possible. |

## 14. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| Taxonomy ambiguity | Define class descriptions, examples, negatives, and confusable classes. |
| Poor label quality | Human review, adjudication, label audits. |
| OCR quality issues | Quality analyzer, visual model fallback, rescan workflow. |
| Model overconfidence | Calibration, thresholds, review triggers. |
| New templates/classes | OOD detection, novelty queue, active learning. |
| Mis-splitting packets | Separate page classification and segmentation evaluation. |
| Cost blow-up from VLMs | Cost-aware routing, fallback only. |
| Vendor lock-in | Canonical data model and adapter layer. |
| Compliance exposure | PII handling, encryption, retention, audit, access controls. |
| Silent model degradation | Drift monitoring and reviewer correction metrics. |

## 15. Definition of done for production pilot

A production pilot is ready when:

- At least 80–90% of input volume is represented in the taxonomy.
- Test set covers classes, source systems, scanners, languages, and quality levels.
- Auto-route threshold policy is approved by business/risk owners.
- Review UI is operational.
- False auto-route rate is measured and acceptable.
- Model and policy versions are logged in decisions.
- Monitoring and alerts are live.
- Rollback procedure is documented and tested.
- Security review is complete.
- Downstream routing has a reconciliation report.

## 16. Team roles

| Role | Responsibility |
|---|---|
| Product owner | business scope, taxonomy approval, routing rules |
| Document AI architect | end-to-end design and trade-offs |
| Backend engineer | ingestion, registry, APIs, events |
| Data engineer | artifact stores, dataset snapshots, pipelines |
| ML engineer | model training, evaluation, calibration |
| MLOps engineer | serving, registry, deployment, monitoring |
| Frontend engineer | review UI |
| Security engineer | IAM, encryption, data handling |
| Business reviewers | label validation and feedback |
| Integration engineer | ECM/workflow/downstream adapters |
