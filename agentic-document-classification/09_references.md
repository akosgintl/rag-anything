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

## Reading notes

### Azure AI Document Intelligence custom classifier

Azure's custom classifier documentation is relevant because it supports page-level classification and multi-document/multi-instance scenarios. This reinforces the design choice that page-level classification and packet splitting should be first-class capabilities, not optional add-ons.

### Google Cloud Document AI custom classifier

Google's custom classifier documentation is relevant because it frames classification as a first step before extraction, which matches the recommended routing pattern: classify first, then send the document to the correct extractor or workflow.

### AWS Textract / Analyze Lending / Comprehend / Bedrock Data Automation

AWS materials are useful because they show the common enterprise IDP decomposition: classify, split, extract, validate, and route. Textract is an OCR/layout provider; Comprehend is a custom text classifier; Bedrock can be used for generative/VLM-based document processing where appropriate.

### LayoutLMv3

LayoutLMv3 is a strong reference architecture for OCR-dependent multimodal document AI because it combines text, layout, and image information. It is a good foundation for a layout-aware classifier.

### Donut

Donut is important because it represents OCR-free document understanding. OCR-free methods are useful when OCR is expensive, low quality, unavailable for the language, or error-prone.

### 2026 multimodal comparative analysis

The 2026 comparison is especially relevant because it evaluates specialized multimodal Transformers and LLM/VLM approaches for visually rich document type classification. Its conclusion supports a pragmatic design: use multimodal/layout-aware approaches as the production backbone and use LLM/VLM components selectively.

### RVL-CDIP

RVL-CDIP remains a common document image classification benchmark. It is useful for experimentation, but it should not replace enterprise-specific validation because real production documents differ by domain, source, language, scan quality, templates, and downstream business risk.
