# Domain 4: Prompt Engineering & Structured Output
## Claude Certified Architect (Foundations) — Study Guide
**Exam Weight: 20% | Primary Scenarios: CI/CD Code Review, Structured Data Extraction**

---

## Overview

Domain 4 is where the exam gets sneaky. Wrong answers sound like good engineering. Right answers require knowing **which technique applies to which specific problem**. Knowing all the techniques is not enough — you must know precisely when and why each one is deployed.

---

## Task Statement 4.1: Explicit Criteria

### The Core Principle

**Vague confidence-based instructions fail. Specific categorical criteria work.**

| ❌ Vague (Wrong) | ✅ Categorical (Right) |
|-----------------|----------------------|
| "Be conservative." | "Flag comments only when claimed behaviour contradicts actual code behaviour." |
| "Only report high-confidence findings." | "Report bugs and security vulnerabilities. Skip minor style preferences and local variable naming patterns." |
| "Avoid false positives." | "Only flag SQL queries where user input reaches the query without parameterisation." |

**Why vague instructions fail:** The model has no shared definition of "conservative" or "high-confidence." Each invocation interprets these differently. Categorical criteria leave no room for interpretation — either the condition is met or it is not.

### The False Positive Trust Problem

**The chain reaction:**
```
High false positives in Category X
    → Developers stop trusting Category X findings
    → Developers start ignoring ALL findings
    → Genuinely critical findings get dismissed
```

**The fix:**
1. Identify which category has the high false positive rate
2. **Temporarily disable** that specific category in the prompt
3. Trust in remaining categories is restored immediately
4. Iterate on prompts for the disabled category separately
5. Re-enable once false positive rate is acceptable

**Exam trap:** The exam presents "raise the confidence threshold" as a plausible fix. This is wrong — vague thresholds are the root cause, not insufficient threshold levels.

### Severity Calibration

**Wrong approach:** Prose descriptions of severity levels.
> "Critical: Very serious issues that could cause significant harm."

**Right approach:** Concrete code examples for each severity level.

```python
# CRITICAL — SQL injection: user input reaches query without parameterisation
query = f"SELECT * FROM users WHERE id = {user_input}"

# HIGH — Unhandled exception in payment path
def process_payment(amount):
    result = payment_gateway.charge(amount)  # No try/except

# MINOR — Inconsistent variable naming (skip this category)
userName = get_user()  # camelCase instead of snake_case
```

**Why code examples beat prose:** The model recognises code patterns precisely. Prose descriptions of severity are interpreted inconsistently across invocations. Code examples are unambiguous anchors.

### Check Questions
1. A code review agent flags 40% of findings as "high severity." Developers have started ignoring all findings. What is the single most effective first step?
2. Why is "only report high-confidence findings" an unreliable instruction?

---

## Task Statement 4.2: Few-Shot Prompting

### When to Deploy Few-Shot Examples

Deploy few-shot examples when you observe any of the following:

| Symptom | Root Cause | Fix |
|---------|------------|-----|
| Inconsistent output formatting across invocations | Model lacks a concrete format anchor | 2–4 formatted examples |
| Inconsistent judgment on ambiguous cases | Categorical criteria not enough for edge cases | Examples showing reasoning for boundary cases |
| Empty/null fields for information that exists in the document | Model uncertain about how to handle varied source formats | Examples showing extraction from each format variant |

### How to Construct Effective Examples

**Number:** 2–4 targeted examples. More is not better — 10 examples add noise without improving generalisation.

**What each example must show:**
1. The input (document excerpt, code snippet, etc.)
2. The output (extracted fields, findings, classification)
3. **The reasoning** — why this output was chosen over plausible alternatives

**The reasoning element is critical.** Without it, you get pattern-matching. With it, you get generalisation to novel cases the examples don't cover.

```
Example 1:
Input: "Revenue: $2.4M (see Appendix B for breakdown)"
Output: { "revenue": 2400000, "revenue_source": "appendix_reference", "revenue_confidence": "indirect" }
Reasoning: Value is stated but sourced externally. Mark as indirect rather than fabricating Appendix B contents.

Example 2:
Input: "The company generated approximately two million in annual recurring revenue"
Output: { "revenue": 2000000, "revenue_source": "inline_approximate", "revenue_confidence": "approximate" }
Reasoning: "Approximately" signals imprecision. Capture the stated figure but flag approximation.
```

### The Hallucination Reduction Effect

Few-shot examples showing correct handling of **varied document structures** dramatically improve extraction quality:

- Inline citations vs. bibliography references
- Narrative prose vs. structured tables
- Partial data vs. complete records
- Conflicting values across sections

Without these examples, the model defaults to a single assumed structure and fabricates values when the real document deviates from that assumption.

### The Exam's Key Distinction

| Problem | Wrong Fix | Right Fix |
|---------|-----------|-----------|
| Inconsistent formatting | Add more prose instructions | Few-shot examples with target format |
| Inconsistent judgment on edge cases | Stricter categorical criteria | Few-shot examples with reasoning for boundary cases |
| Empty fields for existing information | Relax required/optional field settings | Few-shot examples showing extraction from varied formats |

### Check Questions
1. A model correctly extracts financial data from structured tables but produces null values for the same data when it appears as narrative prose. What is the correct fix?
2. Why must few-shot examples include reasoning, not just input/output pairs?

---

## Task Statement 4.3: Structured Output with tool_use

### The Reliability Hierarchy

```
tool_use with JSON schema
    → Eliminates JSON syntax errors entirely
    → Model MUST conform to schema structure

Prompt-based JSON ("respond in JSON format")
    → Model CAN produce malformed JSON
    → No enforcement mechanism
```

**Always use tool_use for structured extraction in production.** Prompt-based JSON is a prototype technique.

### What tool_use Does NOT Prevent (Exam Trap)

This is heavily tested. `tool_use` guarantees **syntax**. It does **not** guarantee **semantics**.

| Error Type | Example | Prevented by tool_use? |
|------------|---------|----------------------|
| Malformed JSON | `{"name": "Acme,}` | ✅ Yes |
| Semantic errors | Line items sum to £980 but `total` field = £1,000 | ❌ No |
| Field placement errors | Tax amount placed in `discount` field | ❌ No |
| Fabrication | Model invents a value for a required field absent from source | ❌ No |

**The fabrication problem is the most dangerous.** When a required field has no source data, the model fills it rather than failing. This is why optional/nullable fields exist.

### tool_choice Options

| Value | Behaviour | Use When |
|-------|-----------|----------|
| `"auto"` | Default. Model may return text OR call a tool. | General conversation with optional tool use. |
| `"any"` | MUST call a tool. Model chooses which one. | Guaranteed structured output with unknown document types (you have multiple schemas). |
| `{"type": "tool", "name": "X"}` | MUST call tool X specifically. | Forcing a mandatory first step (e.g., always call `verify_account` before any transaction). |

### Schema Design Principles

**Prevent fabrication with optional/nullable fields:**
```json
{
  "company_name": { "type": "string" },
  "revenue": { "type": ["number", "null"], "description": "Annual revenue in GBP. Null if not stated." },
  "founded_year": { "type": ["integer", "null"], "description": "Year founded. Null if not stated." }
}
```

**Handle ambiguity with enum + "unclear":**
```json
{
  "sentiment": {
    "type": "string",
    "enum": ["positive", "negative", "neutral", "unclear"]
  }
}
```

**Handle extensibility with "other" + detail string:**
```json
{
  "issue_type": {
    "type": "string",
    "enum": ["bug", "security", "performance", "style", "other"]
  },
  "issue_type_detail": {
    "type": ["string", "null"],
    "description": "Required when issue_type is 'other'. Freeform description."
  }
}
```

**Format normalisation in prompts alongside schemas:**
Even with a strict schema, include normalisation rules:
```
All dates: ISO 8601 format (YYYY-MM-DD)
All monetary values: numeric, in pence/cents (no currency symbols)
All percentages: decimal (0.15, not 15%)
```

### Check Questions
1. A model using tool_use extracts invoice data. The line items sum to £450 but the `total` field contains £500. Which type of error is this and does tool_use prevent it?
2. When would you use `tool_choice: "any"` vs `{"type": "tool", "name": "X"}`?

---

## Task Statement 4.4: Validation-Retry Loops

### Retry-with-Error-Feedback Pattern

When extraction fails validation, do not simply retry the same prompt. Send back:
1. The **original document**
2. The **failed extraction output**
3. The **specific validation error**

```python
retry_prompt = f"""
Original document:
{original_document}

Your previous extraction:
{json.dumps(failed_extraction, indent=2)}

Validation error:
{validation_error}

Please re-extract, correcting the identified error.
"""
```

**Why this works:** The model uses the specific error as a correction signal. Blind retries without error context produce the same failure repeatedly.

### The Retry Effectiveness Boundary (Critical Exam Concept)

| Scenario | Retry Effective? | Reason |
|----------|-----------------|--------|
| Model placed a value in the wrong field | ✅ Yes | Structural error — model can self-correct with error feedback |
| Date formatted as MM/DD/YYYY instead of ISO 8601 | ✅ Yes | Format mismatch — fixable with error feedback |
| Required field is null but source genuinely lacks the information | ❌ No | Information does not exist — retry cannot create it |
| Model invented a plausible but wrong value | ❌ No (for that field) | Source lacks ground truth — schema fix needed (make field nullable) |

**The exam presents scenarios where retry is proposed for missing information.** The correct answer is: make the field optional/nullable in the schema, not retry.

### detected_pattern Fields

Add `detected_pattern` to structured findings to record **which specific code construct triggered the finding**:

```json
{
  "finding": "SQL injection risk",
  "severity": "critical",
  "detected_pattern": "f-string interpolation in database query",
  "file": "app/db.py",
  "line": 47
}
```

**Why this matters:**
- When developers dismiss findings, you know which patterns they rejected
- Enables systematic analysis: "developers always dismiss findings triggered by pattern X"
- Feeds back into prompt refinement with real data rather than guesses

### Self-Correction Flows

Build self-checking into the schema itself:

```json
{
  "line_items": [...],
  "stated_total": { "type": "number" },
  "calculated_total": { "type": "number", "description": "Sum of all line item amounts" },
  "total_discrepancy_detected": { "type": "boolean" }
}
```

Add conflict detection for inconsistent source data:
```json
{
  "revenue_section_1": { "type": ["number", "null"] },
  "revenue_section_2": { "type": ["number", "null"] },
  "revenue_conflict_detected": { "type": "boolean" }
}
```

These fields make the model an active participant in its own validation rather than a passive extractor.

### Check Questions
1. An extraction pipeline fails because the source document does not contain a `founded_year` value, but the schema requires it. Retries produce fabricated years. What is the correct fix?
2. What three elements must be included in a retry prompt for effective self-correction?

---

## Task Statement 4.5: Batch Processing

### Message Batches API — Core Facts

| Property | Value |
|----------|-------|
| Cost saving | 50% vs synchronous API |
| Processing window | Up to 24 hours |
| Latency SLA | None guaranteed |
| Multi-turn tool calling | ❌ Not supported within a single request |
| Request correlation | `custom_id` field |

### The Matching Rule (Exam's Most Tested Concept in 4.5)

```
Synchronous API → blocking workflows
Batch API       → latency-tolerant workflows
```

| Workflow | API | Reason |
|----------|-----|--------|
| Pre-merge code review check | Synchronous | Developer is waiting. Blocking. |
| Nightly security audit across 10,000 files | Batch | Overnight. No human waiting. |
| Real-time customer query classification | Synchronous | User-facing. Latency matters. |
| Weekly compliance report generation | Batch | Scheduled. Result needed by Monday morning. |
| Test generation for overnight CI run | Batch | Scheduled nightly build. |

**The exam's Q11 trap:** A manager proposes using Batch API for everything to save 50% on costs. The correct answer keeps blocking/developer-waiting workflows synchronous. The cost saving is irrelevant if developers are blocked.

### Batch Failure Handling

```
1. Submit batch
2. Collect results, identify failures by custom_id
3. Resubmit only failed documents with modifications:
   - Oversized documents: chunk into smaller pieces
   - Format issues: apply format normalisation
   - Schema validation failures: adjust prompt for that document type
4. Never resubmit the full batch — targeted resubmission only
```

**Prompt refinement before batching:**
- Extract a representative sample (10–20 documents) from the batch
- Iterate prompts on the sample until quality is acceptable
- **Then** submit the full batch
- Prompt refinement after full-batch failure is expensive and slow

### Check Questions
1. A team runs pre-commit hooks that invoke Claude for security checks. Engineers complain the hooks are too slow. Someone proposes switching to Batch API. Is this correct?
2. A batch of 500 documents has 47 failures. What is the correct resubmission strategy?

---

## Task Statement 4.6: Multi-Instance Review

### The Self-Review Limitation

A model reviewing its own output in the same session:
- Retains the reasoning context from generation
- Has implicit investment in the decisions it made
- Is less likely to question its own logic
- Misses subtle errors that an independent perspective would catch

**The fix: independent review instance.** A fresh session with no prior context applies genuine scrutiny.

This is identical to the principle in Domain 3 (3.6 CI/CD) — same concept, tested again in Domain 4 from a prompt engineering perspective.

### Multi-Pass Architecture

**The attention dilution problem:** Processing too many files in a single pass produces inconsistent depth. Some files get thorough analysis; others get surface-level treatment. Identical code patterns get flagged in one file and approved in another.

**The solution:**

```
Pass 1: Per-file local analysis
    → Consistent depth per file
    → Catches local issues: unused variables, type errors, null pointer risks
    → Each file analysed in isolation

Pass 2: Cross-file integration pass
    → Takes summaries from Pass 1 as input
    → Catches data flow issues across files
    → Catches architectural inconsistencies
    → Catches patterns that are individually fine but problematic in combination
```

**Why two passes beat one pass:** The local pass has full attention per file. The integration pass has full attention for cross-file relationships. A single pass must split attention between both concerns and does both poorly.

### Confidence-Based Routing

Build a routing layer on top of the review output:

```json
{
  "finding": "Potential race condition in cache invalidation",
  "severity": "high",
  "confidence": 0.62,
  "confidence_rationale": "Pattern matches known race condition but depends on threading model"
}
```

```
confidence >= 0.85  → Post as automated finding
confidence 0.60–0.84 → Route to human reviewer with finding + rationale
confidence < 0.60   → Discard or flag for prompt improvement
```

**Calibrating thresholds:**
- Build a labelled validation set (findings manually rated correct/incorrect)
- Measure precision at each confidence level
- Set threshold where precision meets your team's acceptable false positive rate
- Do not set thresholds by intuition — use the validation set

### The Full Domain 4 Architecture (Exam Meta-Pattern)

```
Document Input
    ↓
[Few-shot examples loaded]
    ↓
[tool_use extraction with schema: required + optional + nullable fields]
    ↓
[Validation layer: structural + semantic + cross-field checks]
    ↓
[Retry-with-error-feedback if structural/format errors]
    ↓
[Independent review instance]
    ↓
[Confidence-based routing: auto-post / human review / discard]
    ↓
[detected_pattern logging → prompt refinement loop]
```

Every layer solves a specific failure mode. The exam presents scenarios asking which layer is missing or broken.

### Check Questions
1. A code review pipeline produces contradictory findings: the same null-check pattern is flagged in `user.py` but approved in `order.py`. What is the root cause and fix?
2. An independent review instance is catching issues the original session missed. A developer suggests merging the two into one session to reduce API calls. What do you tell them?

---

## Domain 4 Exam Traps — Quick Reference

| Trap | Correct Answer |
|------|---------------|
| "Raise the confidence threshold" for false positives | Define explicit categorical criteria per finding type |
| "Add more detailed instructions" for inconsistent formatting | Use few-shot examples with target format |
| Retry loop for information absent from source | Make the field nullable/optional in schema |
| "tool_use prevents all extraction errors" | tool_use prevents syntax errors only; semantic errors and fabrication still occur |
| Batch API for everything to save costs | Keep blocking workflows synchronous; batch only latency-tolerant workflows |
| Same session reviews its own generated code | Use an independent review instance |
| Single-pass review of 14 files | Multi-pass: per-file local analysis + cross-file integration pass |
| Prose descriptions of severity levels | Concrete code examples for each severity level |
| Few-shot examples without reasoning | Each example must show why that output was chosen over alternatives |
| `tool_choice: "auto"` for mandatory structured output | Use `tool_choice: "any"` or specific tool name |

---

## 8-Question Practice Exam

*(Attempt before reading answers)*

**Q1 (4.1)** — A CI code review agent reports that 35% of its "security vulnerability" findings are false positives. Developers have begun ignoring all findings, including genuine critical bugs. What is the most effective immediate action?

A) Add "only report findings with very high confidence" to the system prompt  
B) Temporarily disable the security vulnerability category and iterate on its criteria separately while keeping other categories active  
C) Reduce the total number of findings by raising the minimum severity threshold to "critical"  
D) Add more examples of false positives to the prompt so the model learns to avoid them  

---

**Q2 (4.2)** — An extraction pipeline correctly extracts financial figures from structured tables but consistently returns null for identical figures when they appear as narrative prose (e.g., "revenue grew to £4.2 million last quarter"). What is the correct fix?

A) Add `"description": "Extract revenue even if stated in prose"` to the schema field  
B) Add few-shot examples showing correct extraction from narrative prose alongside existing table examples  
C) Make the revenue field required rather than optional to force extraction  
D) Switch from tool_use to prompt-based JSON to give the model more flexibility  

---

**Q3 (4.3)** — A tool_use extraction schema has `invoice_total` as a required field. Source invoices occasionally arrive without a stated total. Which schema change prevents the model from fabricating a total?

A) Add a description: "Do not guess the total"  
B) Change `invoice_total` to `{ "type": ["number", "null"] }` and add "Null if total not stated in document"  
C) Add an enum: `["stated", "calculated", "unknown"]` as a separate confidence field  
D) Use `tool_choice: "any"` instead of a specific tool name  

---

**Q4 (4.3)** — A coordinator agent must always call `verify_compliance` before any data export operation. Which tool_choice configuration enforces this?

A) `tool_choice: "auto"` — the model will choose the right tool naturally  
B) `tool_choice: "any"` — the model must call a tool but chooses which  
C) `tool_choice: {"type": "tool", "name": "verify_compliance"}` — forces this specific tool call  
D) Add `"call verify_compliance first"` to the system prompt  

---

**Q5 (4.4)** — An invoice extraction pipeline fails validation because the model placed the VAT amount in the `discount` field. Retries without error feedback produce the same error. What is the correct retry approach?

A) Resubmit with a higher temperature to encourage different outputs  
B) Send back the original document, the failed extraction, and the specific validation error describing the field placement mistake  
C) Make both `vat` and `discount` nullable to prevent validation failures  
D) Switch to prompt-based JSON extraction for more flexibility  

---

**Q6 (4.5)** — A platform runs two Claude-powered workflows: (1) real-time query classification for customer support routing, and (2) nightly batch generation of test cases for the previous day's code commits. An architect proposes switching both to the Message Batches API to reduce costs by 50%. What is the correct response?

A) Approve both — the 50% cost saving justifies any latency trade-off  
B) Approve only the nightly test generation; keep customer support routing synchronous  
C) Approve only customer support routing for batch; keep test generation synchronous for quality control  
D) Reject both — the 24-hour processing window makes Batch API unsuitable for production use  

---

**Q7 (4.6)** — A code review pipeline uses a single Claude session: it generates a refactored version of a module, then immediately reviews its own output for bugs. The pipeline is missing several subtle logic errors. What is the root cause?

A) The context window is too short to hold both the original and refactored code  
B) The same session retains generation reasoning context, making it less likely to question its own decisions  
C) Code review requires plan mode, which cannot run in the same session as generation  
D) The model needs few-shot examples of bug patterns to perform effective review  

---

**Q8 (4.6)** — A review of a 12-file module produces inconsistent findings: critical issues are caught in 4 files, surface-level observations in 6 files, and 2 files receive almost no analysis. The identical null-check anti-pattern is flagged in one file and approved in another. What is the root cause and correct fix?

A) Root cause: insufficient few-shot examples. Fix: add examples of the null-check anti-pattern.  
B) Root cause: attention dilution in a single-pass review. Fix: split into per-file local analysis passes plus a separate cross-file integration pass.  
C) Root cause: model confidence is low. Fix: add confidence thresholds and route uncertain findings to human review.  
D) Root cause: the files are too large for the context window. Fix: truncate files to fit within limits.  

---

## Practice Exam Answers

| Q | Answer | Key Reason |
|---|--------|------------|
| 1 | **B** | Temporarily disable the high-FP category. Restores trust immediately. Iterate separately. |
| 2 | **B** | Few-shot examples for varied document structures solve format-specific extraction failures. |
| 3 | **B** | Nullable field + instruction prevents fabrication. Required fields force the model to invent values. |
| 4 | **C** | Specific tool name forces that exact tool call. "auto" and "any" do not guarantee verify_compliance runs. |
| 5 | **B** | Retry-with-error-feedback: original + failed output + specific error. Blind retry repeats the same mistake. |
| 6 | **B** | Real-time routing is blocking (customer waiting). Keep synchronous. Nightly batch is latency-tolerant. |
| 7 | **B** | Same-session review retains generation context. Use an independent review instance. |
| 8 | **B** | Attention dilution in single-pass review causes inconsistent depth. Multi-pass architecture is the fix. |

**Scoring:**
- 8/8 — Exam-ready on Domain 4
- 6–7/8 — Review weak task statements, then re-test
- Below 6 — Work through missed task statements with additional scenarios

---

## Build Exercise

Build a complete extraction pipeline demonstrating all Domain 4 concepts:

### 1. Schema with required, optional, nullable, and enum fields
```json
{
  "name": "extract_invoice",
  "description": "Extract structured data from invoice documents",
  "input_schema": {
    "type": "object",
    "properties": {
      "vendor_name": { "type": "string" },
      "invoice_number": { "type": "string" },
      "invoice_date": {
        "type": ["string", "null"],
        "description": "ISO 8601 format (YYYY-MM-DD). Null if not stated."
      },
      "line_items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "description": { "type": "string" },
            "amount": { "type": "number" }
          }
        }
      },
      "stated_total": { "type": ["number", "null"], "description": "Null if not stated." },
      "calculated_total": { "type": "number", "description": "Sum of line item amounts." },
      "total_discrepancy_detected": { "type": "boolean" },
      "currency": {
        "type": "string",
        "enum": ["GBP", "USD", "EUR", "other"]
      },
      "currency_detail": {
        "type": ["string", "null"],
        "description": "Required when currency is 'other'."
      }
    },
    "required": ["vendor_name", "invoice_number", "line_items", "calculated_total", "total_discrepancy_detected", "currency"]
  }
}
```

### 2. Few-shot examples for varied document formats
Add 2–3 examples showing extraction from:
- Structured table invoices
- Narrative prose invoices ("we charge £500 for consulting services")
- Multi-currency documents with implicit totals

Each example must include reasoning explaining field choices.

### 3. Validation-retry loop
```python
def extract_with_retry(document, schema, max_retries=2):
    result = extract(document, schema)
    for attempt in range(max_retries):
        errors = validate(result)
        if not errors:
            return result
        result = extract_with_feedback(document, result, errors)
    return result  # Return best attempt; flag for human review
```

### 4. Comparison test
- Process 10 documents without few-shot examples → record null rates and format errors
- Add few-shot examples → reprocess same 10 documents
- Compare: null field rates, format compliance, fabrication rate (manually verify 3 documents)

---

## Summary: The Six Things You Must Know for Domain 4

1. **Categorical criteria beat confidence thresholds.** Define what to flag by condition, not by subjective confidence level.
2. **Few-shot examples solve consistency problems.** Inconsistent formatting, inconsistent judgment, empty fields from varied formats — few-shot first.
3. **tool_use prevents syntax errors only.** Semantic errors, field placement errors, and fabrication still require validation + nullable fields.
4. **Nullable fields prevent fabrication.** Required fields for absent information → model invents values. Make them nullable.
5. **Batch API = latency-tolerant only.** Blocking workflows (developer waiting, real-time routing) stay synchronous.
6. **Independent review instance catches what same-session review misses.** Never use the same session to review its own output.

