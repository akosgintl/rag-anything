# 08 — Governance, Evaluation, Risk, and Quality

## 1. Governance goals

Document classification can drive legal, financial, HR, compliance, and customer-impacting workflows. The governance model must make the system safe, auditable, and improvable.

Governance must cover:

- Taxonomy ownership.
- Label quality.
- Model evaluation.
- Confidence thresholds.
- Human review policy.
- Security and privacy.
- Auditability.
- Drift and incident response.
- Model promotion and rollback.

## 2. Taxonomy governance

### 2.1 Why taxonomy is critical

A classifier cannot be better than the class definitions. Many document classification failures are taxonomy failures, not model failures.

Common taxonomy problems:

- Two classes overlap.
- Business users use different names for the same class.
- A class is defined by downstream workflow rather than document content.
- Too many rare subclasses are introduced too early.
- Unknown classes are forced into known classes.

### 2.2 Taxonomy change process

```mermaid
flowchart LR
    A[New class request] --> B[Business review]
    B --> C[Examples and negatives collected]
    C --> D[Confusable class analysis]
    D --> E[Risk and routing policy]
    E --> F[Taxonomy version update]
    F --> G[Labeling and training]
    G --> H[Evaluation]
    H --> I[Production promotion]
```

### 2.3 Class definition template

Each class should have:

- Class id.
- Display name.
- Description.
- Inclusion criteria.
- Exclusion criteria.
- Positive examples.
- Negative examples.
- Confusable classes.
- Required evidence.
- Risk level.
- Auto-route threshold.
- Review threshold.
- Downstream route.
- Owner.

## 3. Label governance

### 3.1 Label quality levels

| Quality | Description | Training use |
|---|---|---|
| Bronze | historical/weak labels | exploratory only |
| Silver | human-reviewed once | training allowed with caution |
| Gold | expert-reviewed or adjudicated | train/evaluate |
| Platinum | expert consensus, high-risk set | final test and compliance evidence |

### 3.2 Label review process

- Use double review for high-risk or ambiguous classes.
- Track reviewer disagreement.
- Create adjudication workflow for conflicts.
- Sample accepted model predictions for quality control.
- Audit historical labels before training.
- Keep label source and label version.

## 4. Evaluation strategy

### 4.1 Test set structure

A good test set has multiple partitions:

| Partition | Purpose |
|---|---|
| Random holdout | general performance |
| Time-based holdout | future-like performance |
| Source holdout | source generalization |
| Template holdout | new template robustness |
| Low-quality scan set | OCR/visual robustness |
| Confusable class set | boundary errors |
| Unknown/OOD set | unknown handling |
| High-risk class set | compliance confidence |
| Packet set | split + classify behavior |

### 4.2 Classification metrics

Report:

- Overall accuracy.
- Macro F1.
- Weighted F1.
- Per-class precision, recall, F1.
- Top-2 / top-3 accuracy.
- Confusion matrix.
- False auto-route rate.
- Human review deflection rate.

### 4.3 Business-oriented metrics

| Metric | Meaning |
|---|---|
| Straight-through processing rate | percent auto-routed without review |
| False auto-route rate | auto-routed documents later found wrong |
| Review correction rate | percent reviewed cases changed by human |
| Time to decision | ingestion to decision latency |
| Cost per page | OCR + ML + infra + review cost |
| Downstream success rate | extraction/workflow accepted result |
| Unknown discovery rate | true new classes/templates found |

### 4.4 Segmentation metrics

For packets:

- Boundary precision.
- Boundary recall.
- Boundary F1.
- Exact segment match.
- Over-split rate.
- Under-split rate.
- Downstream extraction failure due to split error.

## 5. Confidence and calibration governance

### 5.1 Why calibration matters

A model can be accurate but overconfident. Automation decisions need calibrated probabilities, not just high raw scores.

### 5.2 Required reports

- Reliability curve.
- Expected Calibration Error.
- Brier score.
- Accuracy by confidence bucket.
- Class-specific threshold simulation.
- False auto-route simulation.

### 5.3 Threshold approval

Thresholds should be approved jointly by:

- Business owner.
- Operations owner.
- ML owner.
- Risk/compliance owner for sensitive classes.

Thresholds should be versioned and attached to every decision.

## 6. Human review governance

### 6.1 Review policy

Human review is required when:

- Confidence below class auto-route threshold.
- Margin is low.
- Model disagreement is high.
- OOD score is high.
- Document is high-risk or critical.
- Scan quality is poor.
- Packet split is uncertain.
- Source is untrusted.
- Random QA sample is selected.

### 6.2 Reviewer permissions

- Restrict by tenant, process, and document sensitivity.
- Hide/redact sensitive fields where feasible.
- Log every view and edit.
- Require second reviewer for high-risk corrections.

### 6.3 Review quality metrics

- Reviewer throughput.
- Reviewer disagreement rate.
- Adjudication overturn rate.
- Average review time by class.
- Correction rate by model version.
- SLA breach count.

## 7. Security and privacy

### 7.1 Security controls

| Control | Requirement |
|---|---|
| Encryption at rest | raw, normalized, OCR, features, predictions |
| Encryption in transit | all service calls |
| IAM | least privilege service identities |
| Tenant isolation | per-tenant access policy and data partitioning |
| Secrets | managed secret store only |
| Malware scanning | before parsing |
| Audit logging | every state transition and review action |
| Data minimization | no unnecessary copies or logs |
| Retention | lifecycle policies per class/source |

### 7.2 PII and sensitive data

OCR text can contain more sensitive data than the original file because it is searchable. Treat OCR/layout JSON as sensitive.

Controls:

- Redact logs.
- Avoid storing full OCR in application logs.
- Restrict search index access.
- Apply retention to derived artifacts.
- Restrict training eligibility for sensitive classes.
- Review external VLM/LLM use with privacy/compliance.

## 8. Auditability

Every decision must be reconstructable.

Minimum audit fields:

- Raw artifact hash.
- Package id and source context.
- Page generation parameters.
- OCR/layout provider and version.
- Model ids and versions.
- Rule pack version.
- Taxonomy version.
- Policy version and thresholds.
- Candidate predictions.
- Final decision.
- Review result if any.
- Downstream route result.

## 9. Model risk controls

### 9.1 Model card

Every promoted model should have a model card with:

- Intended use.
- Not intended use.
- Training data summary.
- Evaluation data summary.
- Metrics by class and source.
- Known limitations.
- Confusable classes.
- Calibration quality.
- Threshold recommendation.
- Security/privacy considerations.
- Approval status.

### 9.2 Promotion gates

A model can move to production only if:

- It beats or matches the incumbent on primary metrics.
- It does not regress high-risk classes beyond tolerance.
- Calibration is acceptable.
- False auto-route simulation is acceptable.
- Shadow test does not show unexpected drift.
- Rollback is possible.
- Model card is approved.

## 10. Drift monitoring

### 10.1 Drift types

| Drift | Signal |
|---|---|
| Class distribution drift | incoming class proportions change |
| Source drift | new scanner/vendor/email source |
| Template drift | layout embeddings shift |
| OCR quality drift | OCR confidence drops |
| Confidence drift | model confidence distribution changes |
| Review drift | reviewer correction rate changes |
| Unknown drift | OOD/unknown queue grows |

### 10.2 Drift response

1. Identify affected class/source/template.
2. Temporarily raise review requirements if needed.
3. Sample and label affected documents.
4. Update taxonomy/rules/model as needed.
5. Run targeted evaluation.
6. Promote fix through staging/canary.

## 11. Incident management

### 11.1 Misrouting incident

Steps:

1. Stop auto-routing for affected class/source.
2. Identify affected documents by decision version/time window.
3. Reprocess or review affected documents.
4. Notify downstream owners.
5. Root-cause error: taxonomy, OCR, model, policy, route mapping, or review.
6. Add regression test.
7. Update model/rule/policy.

### 11.2 Data exposure incident

Steps:

1. Disable affected access path.
2. Preserve audit logs.
3. Identify accessed artifacts and users/services.
4. Rotate credentials if needed.
5. Notify security/privacy owners.
6. Review logs and data retention.
7. Patch IAM/logging/search access.

## 12. Validation checklist before auto-routing a class

- [ ] Class definition approved.
- [ ] At least one strong evaluation set exists.
- [ ] Confusable classes tested.
- [ ] Threshold simulation completed.
- [ ] False auto-route rate acceptable.
- [ ] Review UI supports this class.
- [ ] Downstream route tested.
- [ ] Rollback plan exists.
- [ ] Monitoring by class is live.
- [ ] Business owner signs off.

## 13. Recommended KPI targets

These are starting points, not universal guarantees.

| KPI | Initial target |
|---|---:|
| High-confidence precision for auto-routed docs | 98–99%+ for medium/high-risk classes |
| Review correction rate | decreasing over time, monitored by class |
| Unknown/OOD rate | expected during early rollout, should stabilize |
| Packet split exact match | depends on packet complexity; track separately |
| Time to decision | near-real-time for small docs, async SLA for large packets |
| Audit completeness | 100% of final decisions |
| Model version traceability | 100% of predictions/decisions |

## 14. Compliance-friendly design decisions

- Store every model output separately from the final decision.
- Store thresholds used at decision time.
- Use immutable artifact references.
- Keep raw files and OCR/layout artifacts versioned.
- Human corrections should never overwrite original predictions silently.
- Use review queues for high-risk classes even when confidence is high if policy requires it.
- Keep prompt/model versions for any LLM/VLM involvement.
- Implement data retention on both raw and derived data.
