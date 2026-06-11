# 02 — Component Model

## 1. Component catalogue

The extraction system should be designed as a set of small, replaceable services. Each component has a clear input/output contract and can be tested independently.

## 2. Ingestion components

### 2.1 Ingestion API

Responsibilities:

- accept PDF, image, TIFF, DOCX-converted PDF, email attachment, or batch folder inputs,
- assign `document_packet_id`,
- calculate hash and deduplication key,
- store raw document immutably,
- create processing job,
- return job tracking ID.

Inputs:

- file bytes or object storage URI,
- tenant/customer ID,
- optional known document class,
- optional external business key,
- processing priority,
- callback URL or downstream topic.

Outputs:

- `DocumentPacketCreated` event,
- raw artifact metadata,
- initial job state.

### 2.2 File normalizer

Responsibilities:

- validate file type,
- remove unsupported container formats,
- convert office documents to PDF if needed,
- detect encrypted/password-protected files,
- detect malware using external scanner,
- ensure consistent page count and page numbering.

### 2.3 Page renderer

Responsibilities:

- render PDF pages to images,
- preserve page dimensions,
- generate thumbnails,
- generate high-resolution crops on demand,
- normalize orientation where safe.

Recommended default:

- render at 200–300 DPI for printed documents,
- use higher DPI only for small text, ID cards, or handwriting,
- store rendered page images once and reuse them across OCR, classification, extraction, and review.

### 2.4 Image quality analyzer

Responsibilities:

- blur detection,
- skew detection,
- low contrast detection,
- over/under exposure detection,
- cut-off page detection,
- unreadable handwriting detection heuristic,
- image orientation hints.

Outputs:

- `page_quality_score`,
- quality warnings,
- suggested preprocessing actions,
- review flags if the page is unusable.

## 3. Classification/splitting integration components

### 3.1 Classification consumer

Responsibilities:

- consume the previous classification design output,
- verify page ranges and logical document IDs,
- convert classifier-specific labels to canonical `document_class`,
- decide whether extraction can proceed or classification review is needed.

### 3.2 Logical document assembler

Responsibilities:

- group pages into logical documents,
- preserve packet/document/page relationships,
- create extraction job per logical document,
- pass selected document class and class candidates to the extraction orchestrator.

Example:

```json
{
  "document_packet_id": "pkt_2026_000001",
  "logical_document_id": "ldoc_0001",
  "document_class": "invoice.v1",
  "page_range": [1, 2],
  "class_confidence": 0.94,
  "candidate_classes": [
    {"class": "invoice.v1", "confidence": 0.94},
    {"class": "receipt.v1", "confidence": 0.04}
  ]
}
```

## 4. Control-plane components

### 4.1 Document class registry

Stores:

- class ID,
- human-readable name,
- parent class,
- version,
- allowed extractors,
- downstream mappings,
- risk level,
- sample documents,
- classification hints.

Example classes:

```text
invoice.v1
credit_note.v1
receipt.v1
id_card.hu.v1
passport.generic.v1
driving_license.eu.v1
claim_form.health.v1
bank_statement.v1
contract.service_agreement.v1
unknown.v1
```

### 4.2 Extraction schema registry

Stores:

- field names,
- types,
- occurrence rules,
- nested objects,
- field descriptions,
- examples,
- normalization rules,
- validators,
- review thresholds,
- evidence requirements.

### 4.3 Prompt/model recipe registry

Stores:

- model family,
- prompt template,
- output mode,
- page selection strategy,
- OCR context usage,
- image resolution policy,
- retry policy,
- fallback policy,
- cost budget.

### 4.4 Validation rule registry

Stores:

- rule ID,
- scope: field/document/table/cross-document,
- implementation type: regex, code, SQL, rule DSL, LLM-assisted,
- severity,
- review trigger,
- error message.

## 5. Extraction runtime components

### 5.1 Extraction orchestrator

Responsibilities:

- receive logical document and selected class,
- load schema and recipe,
- call OCR/layout if needed,
- call VLM/LLM/parsers according to recipe,
- merge candidates,
- create field evidence,
- run validation,
- route to review or publish.

The orchestrator should not contain class-specific logic in code. Class-specific behavior should come from the schema/recipe registry.

### 5.2 OCR/layout service

Responsibilities:

- extract words, lines, paragraphs, blocks,
- detect tables and cells,
- detect key-value candidates,
- preserve bounding boxes,
- produce reading order,
- support multiple providers.

Output should be provider-neutral:

```json
{
  "page_number": 1,
  "width": 2480,
  "height": 3508,
  "words": [],
  "lines": [],
  "blocks": [],
  "tables": [],
  "selection_marks": []
}
```

### 5.3 VLM extractor

Responsibilities:

- read page images directly,
- extract fields according to a schema,
- identify visual evidence regions,
- handle handwritten or visually ambiguous fields,
- return JSON-only schema-constrained output where supported.

Typical use cases:

- invoices with complex layout,
- handwritten forms,
- ID cards,
- tables with merged cells,
- low-OCR quality pages.

### 5.4 LLM structured extractor

Responsibilities:

- use OCR/layout text as context,
- fill class-specific schema,
- normalize values cautiously,
- explain missing/unreadable fields,
- preserve evidence references.

Use for:

- long documents,
- contracts,
- dense printed forms,
- extracting policy/compliance clauses,
- structured field extraction from reliable OCR.

### 5.5 Deterministic parser service

Responsibilities:

- parse MRZ,
- parse barcodes/QR codes,
- validate VAT IDs where possible,
- validate IBAN checksum,
- parse dates and currencies,
- calculate invoice totals,
- normalize country/currency/language codes.

### 5.6 Candidate merger

Responsibilities:

- combine OCR, VLM, LLM, and parser candidates,
- resolve conflicts,
- prefer deterministic parser for machine-readable fields,
- keep alternative candidates where needed,
- compute field-level final confidence.

Recommended priority order:

1. deterministic machine-readable source with checksum, such as MRZ/barcode/IBAN checksum,
2. exact OCR evidence with high confidence,
3. VLM extraction with visual evidence,
4. LLM extraction from OCR context,
5. heuristic fallback,
6. human review.

## 6. Validation components

### 6.1 Schema validator

Validates:

- required fields,
- field types,
- enum values,
- occurrence limits,
- nested object structure,
- JSON schema conformance.

### 6.2 Normalizer

Normalizes:

- dates,
- numbers,
- currencies,
- addresses,
- names where allowed,
- phone numbers,
- tax IDs,
- document numbers,
- line item quantities and amounts.

Always preserve the raw value.

### 6.3 Business rule validator

Examples:

- invoice line sums equal subtotal,
- subtotal plus tax equals total,
- due date is after invoice date,
- ID expiry date is after issue date,
- MRZ date of birth equals visual text date of birth,
- form checkbox value is one of allowed options,
- required signature field is present if the document class requires it.

### 6.4 Risk/confidence scorer

Responsibilities:

- combine model confidence, OCR confidence, evidence quality, validation status, class confidence, and field risk,
- classify field as accepted, warning, rejected, or review required,
- produce document-level status.

## 7. Human review components

### 7.1 Review task creator

Creates tasks with:

- field name,
- proposed value,
- alternatives,
- page image crop,
- OCR text near the field,
- validation errors,
- review instructions,
- risk level.

### 7.2 Review UI

Minimum required UI features:

- page viewer,
- field list,
- crop viewer,
- candidate alternatives,
- validation messages,
- keyboard-first correction,
- reviewer comments,
- submit/reject/mark unreadable.

### 7.3 Correction feedback service

Responsibilities:

- store reviewer corrections,
- attach corrections to schema/model/prompt versions,
- export training/evaluation datasets,
- identify recurring error patterns.

## 8. Publishing components

### 8.1 Canonical record store

Stores final extraction records with current status and full traceability.

### 8.2 Downstream adapter

Maps canonical output to:

- ERP invoice import,
- CRM/customer onboarding,
- claims management,
- case management,
- data warehouse,
- search/RAG index,
- audit archive.

### 8.3 Delivery guarantee

Use idempotent downstream publishing:

- stable `logical_document_id`,
- stable `extraction_run_id`,
- downstream delivery ID,
- retry counter,
- dead-letter queue,
- delivery status.

