+++
title = "Three Silent Data Loss Bugs We Caught in Testing (and Nearly Shipped)"
date = "2025-08-10"
draft = false
tags = ["testing", "ml infrastructure", "data validation", "production reliability"]
summary = "A document extraction pipeline with three separate bugs that would have silently corrupted data in production, all caught in testing. None threw errors. All looked correct."
+++

# Three Silent Data Loss Bugs We Caught in Testing (and Nearly Shipped)

We built a document extraction pipeline that processed structured data from unstructured PDF documents. PDFs to extracted records to downstream systems (various integrations depending on tenant).

The pipeline worked. Extracted thousands of records per batch, normalized them, pushed them downstream. No errors. No failures. Nearly broke in three separate, silent ways.

None of these bugs would surface in code review. None threw exceptions. All three produced output that looked correct and passed schema validation. The only thing that caught them: building measurement into the system.

---

## The Architecture

Step Functions Express Workflow with seven Lambda stages:
1. S3 ingestion, format detection
2. PDF page splitting (MapState, max 20 concurrent)
3. AWS Textract OCR per page
4. Claude LLM extraction via Bedrock
5. Schema normalization + validation
6. RDS atomic write (job + extracted records)
7. Downstream API push

All stateless Lambdas. RDS for job state and extracted records. Configuration driven by tenant_id, extraction schema, validation rules, prompt version, downstream API endpoint. Same code runs for different tenants with different schemas (Tenant A: 11 fields, Tenant B: 10 different fields).

---

## Bug #1: Silent Context Window Truncation

Large document with 150 records extracted successfully. Schema validation passed. Got back 80 records.

No errors. No timeouts. No Lambda failures. The LLM just stopped extracting mid-document.

Caught this because I'd built a completeness check: input_record_count vs. output_record_count. Mismatch immediately visible.

**Why it happened**

Claude Sonnet 3.0 had ~20-page context window (~50k tokens for most PDFs). Beyond that, silently degraded. Didn't refuse. Didn't error. Just stopped.

Root cause: no model versioning. Comment in prod code: "use main branch for production" based on Slack discussion. No formal deployment, no version strategy.

**The fix**

Switch to Claude 3.5 Sonnet (~100-page capacity). Add model versioning to requirements.txt. Add to Step Functions context. Make validation include completeness checks.

The real fix: completeness is a validation concern, not an accuracy concern. Most systems validate that extracted values are correct (is salary a number? is date formatted right?). Fewer validate that you got everything you asked for.

When extracting from variable-length documents, completeness is as critical as correctness.

---

## Bug #2: Semantic Misinterpretation

Customer pushback on numeric field data. Flagged in their review queue as implausible.

Extracted annual amount: $5,500. Actual data: monthly amount of $5,500.

Another record: rate extracted in wrong unit. Another: aggregated total confused with base value.

**Why it matters**

The extraction was syntactically correct. Value was a number. In reasonable range. Confidence score 0.78. Downstream system accepted it. Processed it.

If we'd shipped this, silent corruption of downstream records in production systems.

**Why it happened**

Training data was clean, well-formatted. Real customer documents were messy. Values expressed multiple ways: "$5.5k/month," "$28/hr," aggregates mixed with base. The LLM made reasonable guesses based on available context. Got them wrong.

More fundamentally: numeric field extraction isn't straightforward. It's a semantic concept with multiple expressions. Without metadata (which unit? aggregate or base? which category?), the LLM inferred.

**The fix**

Three layers:

1. Expand training data: collect real customer documents with known ground truth values. Rebuild extraction prompt with common expression variants.

2. Confidence as a feature: return `{"value": 5500, "value_confidence": 0.47, "value_interpretation": "MONTHLY"}` instead of just the number. That 0.47 on monthly interpretation was the signal.

3. Validation bounds that respect context: if value is implausible for the entity's domain/category, send to manual review.

**The lesson**

Lab accuracy != production reliability.

Our model showed high accuracy on test data. In production, "accuracy" meant inferring intent from ambiguous, variably-formatted text. Real data is messier, more varied, more ambiguous than any test set you build.

Confidence signals aren't optional instrumentation. They're product requirements. You can't safely deploy an AI system without visibility into where it's uncertain.

---

## Bug #3: Invisible Incompleteness

Three months in, quarterly review of record counts for a large customer. Consistently missing 10-15% of records.

Investigated every layer:
- OCR was complete
- LLM extraction looked fine (spot checks)
- Schema validation passed
- No RDS write failures

The records just weren't there.

**Why it happened**

We shipped without record-level confidence metrics. Had LLM-level confidence (does the system understand the document?). Not record-level confidence (does the system think this specific record is extracted correctly?).

The LLM was silently dropping records it wasn't confident about. Not flagging them as INVALID. Not logging them. Just not including them in the JSON array.

We only noticed when comparing input to output counts.

**The fix**

First: add confidence-per-record to the validation layer. Distinguish between:
- Low-confidence extracted record: send to customer review
- Missing record: investigate (OCR failed on that page? LLM not recognizing certain record types?)

Second: we'd built this for one tenant only. When we tried to expand to a different tenant with a different domain, different schemas would have hidden the same problem again.

So we refactored validation and extraction to be tenant-configurable. Extraction schema, validation rules, confidence thresholds, downstream API URLs, all RDS configuration, not code.

The LLM extraction Lambda didn't know which tenant's data it was processing. It read the tenant's configuration and applied it.

This made failures visible: each tenant config included required_confidence_threshold, so we could surface when records failed that threshold. Later we could compare: why is Tenant B extracting at 92% completeness while Tenant A is at 98%? Turns out certain field types are expressed more variably in source documents than others. That fed back into prompt engineering.

---

## What Caught These

Standard testing would have missed all three.

Static test data doesn't capture 20-page PDFs or real value expression variance. Unit tests don't catch context window truncation. Integration tests can pass while your system loses 10% of records.

What actually caught them:

1. **Test data matching production variance**, large documents, real field formats, real record counts. Tedious to set up. Easy to skip. Do it anyway.

2. **Completeness as a validation concept**, not just syntactic correctness (is it a number?), but semantic completeness (did we get everything we asked for?).

3. **Confidence signals as product requirements**, not instrumentation, not metrics. Requirements. The system needs to know what it doesn't know.

4. **Architecture that makes behavior configurable**, hardcoded validation rules hide domain-specific problems. Configurability makes them visible.

5. **Measurement over prevention**, failures weren't caught by preventing them. They were caught by measuring outcomes against expectations.

---

## What Mattered

Testing discipline didn't come from comprehensive coverage or edge case thinking.

It came from:
- Designing for completeness as a core validation concept
- Making absence of signals visible, not silent
- Choosing architecture that exposes blind spots
- Measuring every change against ground truth

The pipeline was designed so that failures couldn't hide.

That's the discipline.

---