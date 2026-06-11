# 05 — Model Strategy for 2026 Document Classification

## 1. Recommended model philosophy

Do not design the solution around a single classifier. The best practical design is a **hybrid classifier architecture**:

```text
rules + metadata + OCR text + layout-aware model + visual model + optional VLM/LLM fallback + calibration + policy
```

The model output is not the business decision. The system should separate:

1. Raw candidate predictions.
2. Fused/calibrated prediction.
3. Policy-based classification decision.
4. Human-reviewed final label where applicable.

## 2. Why hybrid classification is needed

Different document classes expose different signals:

| Signal | Strong for | Weak for |
|---|---|---|
| Filename/source metadata | controlled intake channels | open upload, renamed files |
| OCR text | contracts, letters, reports, native PDFs | poor scans, handwriting, visually similar forms |
| Layout | forms, invoices, statements, applications | plain text documents |
| Visual appearance | templates, scans, IDs, statements | documents differing mainly by wording |
| Business rules | known templates, barcodes, fixed forms | novel classes and exceptions |
| VLM/LLM reasoning | ambiguous/rare/zero-shot cases | high-volume cheap classification if used alone |

## 3. Model categories

### 3.1 Deterministic rules

Rules should handle known high-precision cases.

Examples:

- Barcode `FORM-1003` ⇒ mortgage application.
- `W-2` visible in top title region ⇒ W-2 tax form.
- Sender + mailbox + filename pattern strongly suggests AP invoice.
- PDF metadata contains known template id.

Rule output should still be represented as a `PredictionCandidate` with evidence and confidence. Rules should not bypass audit.

### 3.2 Text classifier

Text classifiers consume OCR/native text.

Recommended progression:

1. Baseline: TF-IDF + linear classifier.
2. Better: sentence/document embeddings + gradient boosted classifier or shallow neural classifier.
3. Stronger: fine-tuned transformer encoder.
4. Cloud option: managed custom text classification.

Strengths:

- Fast.
- Cheap.
- Easier to explain with keywords and snippets.
- Works well for letters, emails, contracts, reports.

Weaknesses:

- Sensitive to OCR quality.
- Loses layout information.
- Confuses classes with similar text but different visual structure.

### 3.3 Layout-aware transformer

Layout-aware models combine OCR tokens, bounding boxes, and sometimes page images. This is usually the core model for enterprise documents.

Good candidates:

- LayoutLMv3-style models.
- LiLT-style multilingual layout models.
- DocFormer-style multimodal architectures.
- Custom spatial transformer over OCR tokens.

Strengths:

- Strong on forms and visually rich documents.
- Uses both text and where text appears.
- Good balance of accuracy and production cost.

Weaknesses:

- Depends on OCR quality.
- Token limits require careful truncation/page handling.
- Multi-page documents need aggregation strategy.

### 3.4 Visual / OCR-free model

OCR-free models classify from the page image directly or learn to decode structured output from images.

Examples:

- Donut-style encoder-decoder.
- ViT/Swin/ConvNeXt page image classifier.
- Multimodal image-text encoders.

Strengths:

- Avoids OCR error propagation.
- Captures visual style and form structure.
- Useful for poor scans and multilingual/handwritten content.

Weaknesses:

- Harder to inspect evidence.
- More GPU-intensive.
- Needs robust image preprocessing.

### 3.5 VLM / LLM classifier

Vision-language models and LLMs can classify documents with constrained prompts and schema output.

Use cases:

- Fallback for ambiguous cases.
- Rare classes where supervised data is limited.
- Initial zero-shot taxonomy exploration.
- Reviewer assistive explanation.
- Weak-label generation for active-learning candidates.

Production controls:

- Use allowed class list only.
- Force JSON schema output.
- Store model prompt and output version.
- Do not expose chain-of-thought; store concise evidence only.
- Apply cost budget and rate limits.
- Human review high-risk outputs.
- Do not let LLM output override deterministic safety policies.

Example prompt pattern:

```text
You are classifying an enterprise document. Choose exactly one class_id from the allowed taxonomy or UNKNOWN.
Use only visible evidence from the document text/layout.
Return JSON matching the schema. Do not invent missing fields.

Allowed classes:
- FIN.INVOICE.SUPPLIER: Supplier invoice requesting payment.
- FIN.CREDIT_NOTE: Credit note reducing an invoice amount.
- FIN.PURCHASE_ORDER: Purchase order issued by buyer.

Document evidence:
<ocr text snippets, layout summaries, page thumbnails if VLM>
```

Schema:

```json
{
  "class_id": "FIN.INVOICE.SUPPLIER|FIN.CREDIT_NOTE|FIN.PURCHASE_ORDER|UNKNOWN",
  "confidence": "low|medium|high",
  "evidence": [
    {"page": 1, "kind": "text", "value": "Invoice No.", "why_it_matters": "invoice indicator"}
  ],
  "confusable_classes": ["FIN.CREDIT_NOTE"],
  "needs_human_review": false
}
```

## 4. Page vs document modeling

### 4.1 Page-level model

Classifies each page independently or with local context.

Use cases:

- Identify blank/separator pages.
- Detect document boundaries.
- Classify pages in packets.
- Route pages to specialized extractors.

### 4.2 Document-level model

Classifies one `DocumentSegment`.

Aggregation strategies:

| Strategy | Description | Good for |
|---|---|---|
| First page only | classify from first page | forms with strong title/header |
| Majority vote | aggregate page predictions | homogeneous multi-page docs |
| Max evidence | strongest page drives class | docs where only first/last page carries type |
| Sequence model | model page order | packets and multi-page forms |
| Attention aggregation | learn page weights | complex documents |
| Hierarchical model | page encoder + document encoder | high accuracy multi-page classification |

Recommended baseline:

- Use first page + all-page text + page sequence predictions.
- Add hierarchical model after enough data is available.

## 5. Packet splitting model

For document packets, segmentation is critical. Mis-splitting causes downstream extraction errors even if page classification is good.

Model options:

1. Rule-based separator/blank/barcode splitter.
2. Page class transition model.
3. BIO sequence labeling over pages.
4. Pairwise adjacent-page same-document classifier.
5. VLM/LLM fallback for ambiguous packet boundaries.

Recommended output:

```json
{
  "page_boundary_predictions": [
    {"after_page": 1, "new_document_probability": 0.04},
    {"after_page": 2, "new_document_probability": 0.91}
  ],
  "segments": [
    {"pages": [1,2], "split_confidence": 0.94},
    {"pages": [3,4,5], "split_confidence": 0.89}
  ]
}
```

## 6. Fusion design

### 6.1 Candidate inputs

Each classifier produces `top_k` predictions and evidence.

Example candidates:

| Classifier | Class | Raw score |
|---|---|---:|
| rule_form_title | `FIN.INVOICE.SUPPLIER` | 0.99 |
| text_classifier_v3 | `FIN.INVOICE.SUPPLIER` | 0.88 |
| layoutlmv3_v5 | `FIN.INVOICE.SUPPLIER` | 0.96 |
| visual_vit_v2 | `FIN.INVOICE.SUPPLIER` | 0.91 |
| nearest_neighbor | `FIN.CREDIT_NOTE` | 0.64 |

### 6.2 Fusion features

- Model score per class.
- Model rank per class.
- Model agreement.
- Class-specific model reliability.
- OCR quality.
- Source channel.
- Historical class prior for the source.
- Top-2 margin.
- OOD score.
- Rule strength.

### 6.3 Fusion implementation options

| Option | Pros | Cons |
|---|---|---|
| Weighted average | simple, transparent | weak when models are miscalibrated |
| Logistic regression stacker | interpretable, strong baseline | needs validation data |
| Gradient boosted stacker | handles nonlinear interactions | less transparent |
| Bayesian fusion | principled uncertainty | more complex |
| Conformal prediction | produces prediction sets | requires careful calibration split |

Recommended MVP: weighted average + per-class thresholds.
Recommended production: logistic/GBM stacker + calibration + conformal prediction sets for review.

## 7. Confidence and uncertainty

Raw model confidence is usually not enough. Store multiple uncertainty signals.

| Signal | Meaning |
|---|---|
| `p_calibrated` | estimated probability after calibration |
| `margin` | top probability minus second probability |
| `entropy` | overall uncertainty across classes |
| `prediction_set_size` | how many labels remain plausible |
| `ood_score` | novelty/out-of-distribution estimate |
| `model_agreement` | agreement between independent classifiers |
| `quality_penalty` | reduction due to scan/OCR quality |

### 7.1 Threshold policy example

```yaml
threshold_policy:
  default:
    auto_route: 0.95
    review: 0.70
  by_risk:
    low:
      auto_route: 0.90
      review: 0.60
    medium:
      auto_route: 0.94
      review: 0.70
    high:
      auto_route: 0.98
      review: 0.85
    critical:
      auto_route: null
      review: 0.90
  additional_review_triggers:
    - margin_below: 0.15
    - ood_score_above: 0.30
    - model_agreement_below: 0.70
    - ocr_mean_confidence_below: 0.80
```

## 8. Training data strategy

### 8.1 Minimum useful data

For a serious enterprise classifier:

| Stage | Suggested data |
|---|---|
| Prototype | 20–50 examples/class if using foundation/VLM-assisted fallback |
| MVP | 100–300 examples/class where possible |
| Production | 500+ examples/class for important classes, plus negative/confusable examples |
| High-risk class | more data, expert review, and separate test set |

Managed cloud services may start with fewer examples for custom classifiers, but production reliability still depends on representative examples and a careful test set.

### 8.2 Label types

| Label type | Use |
|---|---|
| Gold human label | training and final evaluation |
| Expert adjudicated label | high-risk classes and test sets |
| Historical label | bootstrapping, must be audited for noise |
| Weak rule label | pretraining or active learning candidate |
| LLM-assisted label | triage, never treat as gold without validation |
| Synthetic example | robustness testing and low-data augmentation |

### 8.3 Dataset split

Avoid naive random splits. Prefer:

- Time-based holdout.
- Source-system holdout.
- Template-family holdout.
- Vendor/customer holdout where applicable.
- Scanner/site holdout.
- Confusable-class stress set.

## 9. Evaluation metrics

### 9.1 Classification metrics

- Accuracy.
- Macro F1.
- Weighted F1.
- Per-class precision/recall/F1.
- Top-k accuracy.
- Confusion matrix.
- False auto-route rate.
- Review deflection rate.

### 9.2 Segmentation metrics

- Boundary precision/recall/F1.
- Segment exact match.
- Page-to-document assignment accuracy.
- Over-split and under-split rate.

### 9.3 Calibration metrics

- Expected Calibration Error (ECE).
- Brier score.
- Reliability curves.
- Accuracy by confidence bucket.

### 9.4 Business metrics

- Manual review rate.
- Reviewer correction rate.
- Straight-through processing rate.
- Cost per page.
- Time to route.
- Downstream extraction success rate.
- Misroute incident rate.

## 10. Error taxonomy

Track errors using a consistent taxonomy:

| Error type | Example |
|---|---|
| OCR failure | text missing due to poor scan |
| Layout confusion | invoice vs purchase order with similar terms |
| Split error | contract pages split into two docs |
| Taxonomy ambiguity | class definitions overlap |
| Source drift | new template from supplier |
| Unknown class | new document type not in taxonomy |
| Rule conflict | metadata rule contradicts visual evidence |
| Reviewer error | incorrect human correction |
| Downstream mismatch | classified correctly but sent to wrong extraction profile |

## 11. Active learning

Select samples for review/training based on value:

- Low confidence.
- Low margin.
- High model disagreement.
- High OOD score.
- New source/template cluster.
- High business volume class with rising errors.
- Random sample for unbiased quality measurement.
- High-risk class sample for compliance QA.

Active learning loop:

```mermaid
flowchart LR
    A[Production predictions] --> B[Uncertainty scoring]
    B --> C[Sample selector]
    C --> D[Human review]
    D --> E[Gold labels]
    E --> F[Dataset snapshot]
    F --> G[Retrain + evaluate]
    G --> H[Deploy]
```

## 12. OOD and unknown-class handling

A production document classifier must admit when it does not know.

Methods:

- Maximum softmax probability threshold.
- Embedding distance from known training examples.
- Conformal prediction set too large.
- Autoencoder / density model over embeddings.
- High VLM uncertainty.
- Model disagreement.
- Novel cluster detection.

Unknown handling policy:

1. Route to novelty queue.
2. Reviewer assigns known class, unknown subtype, or new class candidate.
3. Product owner approves taxonomy change if needed.
4. Backfill labels and retrain.

## 13. Practical model roadmap

### Phase A — Baseline

- Rules.
- OCR text extraction.
- TF-IDF + linear classifier.
- Simple thresholds.
- Manual review.

### Phase B — Strong MVP

- Layout-aware model.
- Page classification.
- Packet splitting.
- Calibration.
- Per-class thresholds.
- Review feedback loop.

### Phase C — Advanced production

- Visual/OCR-free model.
- Fusion stacker.
- OOD detection.
- Active learning.
- Model monitoring.
- Shadow/canary deployments.

### Phase D — 2026 modern extension

- VLM fallback for ambiguous cases.
- LLM-assisted taxonomy maintenance.
- Reviewer copilot for evidence summaries.
- Semi-supervised class discovery.
- Cost-aware inference routing.
- Cross-cloud model adapters.

## 14. Recommended MVP model stack

For most enterprise use cases, start with:

| Layer | Technology pattern |
|---|---|
| OCR/layout | cloud OCR or strong open OCR adapter |
| Baseline classifier | TF-IDF + logistic regression |
| Primary classifier | LayoutLMv3-style layout-aware classifier |
| Visual backup | page-image ViT classifier or Donut-style model |
| Fusion | weighted/logistic stacker |
| Confidence | temperature scaling + per-class thresholds |
| Review | human UI with evidence overlays |
| Feedback | versioned label store + retraining pipeline |

## 15. Model anti-patterns

Avoid:

- LLM-only classification with no calibration.
- A single global confidence threshold.
- Evaluating only on clean public benchmark data.
- Ignoring packet splitting.
- Training on historical labels without noise analysis.
- Treating OCR confidence as document classification confidence.
- Promoting models without shadow comparison.
- Not storing model version and prompt version.
- Ignoring rejected/unknown documents during evaluation.
