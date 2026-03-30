# Domain 5: Context Management & Reliability
## Claude Certified Architect (Foundations) — Study Guide
**Exam Weight: 15% | Primary Scenarios: Customer Support Agent, Multi-Agent Research, Structured Data Extraction**

---

## Overview

Domain 5 is the smallest domain by weight, but its concepts are load-bearing for everything else. Context management failures break multi-agent pipelines (Domain 1), structured extraction (Domain 4), and CI/CD workflows (Domain 3). Get this wrong and you will lose marks in those domains too — not just this one.

The exam tests these concepts across nearly every scenario. You will not see a section labelled "context management" — you will see a broken multi-agent system and need to identify that the root cause is context degradation, error suppression, or missing provenance.

---

## Task Statement 5.1: Context Preservation

### The Progressive Summarisation Trap

**The problem:** Summarising conversation history to manage context length destroys precision.

| Before summarisation | After summarisation |
|----------------------|---------------------|
| "Customer wants a refund of £247.83 for order #8891 placed on 3rd March" | "Customer wants a refund for a recent order" |
| "Failure occurred at 14:32 UTC on API call to `/payments/v2/refund` with 503 response" | "There was a payment error" |
| "Customer has escalated twice, reference IDs #4421 and #4423" | "Customer has contacted support before" |

Every time you summarise, transactional facts — amounts, dates, order numbers, reference IDs, percentages — collapse into vague prose. The agent downstream has no basis for a precise resolution.

**The fix: persistent "case facts" block.**

```
CASE FACTS (do not summarise — include verbatim in every prompt):
- Customer: Sarah Chen, account #88234
- Order: #8891, placed 2026-03-03
- Claimed amount: £247.83
- Product: Wireless headphones, SKU HW-7712
- Prior contact: ref #4421 (2026-03-10), ref #4423 (2026-03-12)
- Stated issue: Item arrived without charging cable
```

This block travels through the entire conversation unchanged. It is never summarised. Conversation prose around it can be compressed; this block cannot.

### The "Lost in the Middle" Effect

**The problem:** Models process the beginning and end of long inputs reliably. Content buried in the middle is processed less consistently.

```
[Beginning — HIGH ATTENTION]
  ... key findings from agent 1 ...
[Middle — REDUCED ATTENTION]
  ... findings from agents 2–8 ... ← these get missed
[End — HIGH ATTENTION]
  ... synthesis instruction ...
```

**The fixes:**
1. Place key findings summaries **at the beginning** of the prompt, before supporting detail
2. Use **explicit section headers** throughout long inputs so the model can navigate structure
3. For multi-agent results, put the most critical findings first, supporting evidence after

### Tool Result Trimming

**The problem:** Tool results return far more data than needed. Accumulated verbose results exhaust the token budget.

```python
# Raw order lookup: 47 fields
order = lookup_order("8891")

# Trim to relevant fields BEFORE appending to context
relevant = {
    "order_id": order["id"],
    "placed_date": order["created_at"],
    "total": order["total_amount"],
    "status": order["fulfillment_status"],
    "items": order["line_items"]
}
# Append relevant — not order — to conversation history
```

This is not about losing information. It is about preventing irrelevant fields (shipping warehouse ID, internal routing codes, A/B test flags) from filling the context window across dozens of tool calls.

### Full History Requirements

Every subsequent API request must include the **complete conversation history** from turn one. The model has no memory between API calls — each call is stateless. Omitting earlier messages breaks coherence:

```python
# WRONG — model has no memory of turns 1-3
messages = [{"role": "user", "content": current_message}]

# CORRECT — full history included
messages = conversation_history + [{"role": "user", "content": current_message}]
```

### Upstream Agent Optimisation

When downstream agents have limited context budgets, modify upstream agents to return **structured summaries** rather than verbose content and reasoning chains.

| Upstream agent returns | Downstream receives |
|------------------------|---------------------|
| Full research paper text + reasoning | ~8,000 tokens of noise |
| `{ "key_claim": "...", "source": "...", "relevance_score": 0.91, "excerpt": "..." }` | ~150 tokens of signal |

This is especially critical in hub-and-spoke architectures (Domain 1) where the coordinator must hold results from 4–6 subagents simultaneously.

### Check Questions
1. A customer support agent produces a resolution that refers to "the customer's recent order" without specifying amount or order number. The original issue was reported in turn 3 of a 12-turn conversation. What is the root cause?
2. A coordinator agent receives results from 6 research subagents. After integrating them, the findings from agents 3 and 4 are absent from the synthesis. What structural fix addresses this?

---

## Task Statement 5.2: Escalation and Ambiguity Resolution

### The Three Valid Escalation Triggers

These are the only three conditions that justify escalating to a human agent:

| Trigger | Why it is valid | Action |
|---------|----------------|--------|
| **Customer explicitly requests a human** | Autonomy. The customer has the right to choose. | Escalate immediately. Do NOT attempt to resolve first. |
| **Policy exception or gap** | The agent cannot act outside its policy scope. | Escalate with context. Note which policy gap triggered this. |
| **Inability to make meaningful progress** | Continuing wastes time and increases frustration. | Escalate with structured handoff: what was tried, what failed. |

### The Two Unreliable Triggers (Exam Traps)

| Trigger | Why it is unreliable |
|---------|---------------------|
| **Sentiment / frustration level** | Frustration does not correlate with case complexity. A simple refund can frustrate anyone. A genuinely complex case can be described calmly. |
| **Self-reported confidence score** | The model is systematically miscalibrated: often highly confident on hard cases, uncertain on easy ones. Never use the model's own confidence as the escalation signal. |

### The Frustration Nuance (Heavily Tested)

This is a two-step logic the exam consistently tests:

```
Customer is frustrated
    ↓
Is the issue straightforward?
    YES → Acknowledge frustration. Offer resolution. Do NOT escalate yet.
    ↓
Does customer reiterate preference for human after your offer?
    YES → Escalate now.
    NO  → Proceed with resolution.

BUT:
Customer explicitly says "I want to speak to a human"
    → Escalate immediately. No investigation. No "let me try to help first."
```

The distinction: **implicit frustration** ≠ **explicit human request**. Only the explicit request is an immediate, unconditional trigger.

### Ambiguous Customer Matching

Multiple customers match a search query (e.g., two "Sarah Chen" accounts):

**Wrong approach:** Select based on heuristics — most recent, most active, most likely.
- This has a non-zero error rate
- Wrong account selection leads to wrong resolutions, potential data disclosure

**Correct approach:** Ask for additional identifying information:
- Email address
- Phone number
- Order number
- Postcode

Never select. Always confirm.

### Check Questions
1. A customer contacts support, expresses significant frustration, and says "this is ridiculous." The agent immediately escalates to a human. Is this correct?
2. A customer asks about a competitor price match. Your policy document covers price matching for your own sale items but says nothing about competitors. What is the correct escalation trigger here?

---

## Task Statement 5.3: Error Propagation

### Structured Error Context

When a tool or subagent fails, the error must carry structured context — not just a status code. A downstream coordinator needs to know:

```json
{
  "error_type": "transient",
  "what_was_attempted": "Journal database query for 'geothermal energy capacity 2023-2025'",
  "parameters_used": { "source": "academic_journals", "date_range": "2023-2025", "topic": "geothermal" },
  "partial_results": [
    { "title": "Global Geothermal Capacity Report 2022", "relevance": 0.78 }
  ],
  "potential_alternatives": ["web_search with same query", "retry after 30s"],
  "retry_recommended": true
}
```

**The four error type classifications:**

| Type | Description | Typical Action |
|------|-------------|---------------|
| `transient` | Network timeout, rate limit, temporary unavailability | Retry with backoff |
| `validation` | Input did not meet tool's expected format | Fix input, retry |
| `business` | Policy prevents action (e.g., refund exceeds limit) | Escalate or offer alternative |
| `permission` | Insufficient access rights for this operation | Escalate, do not retry |

### The Two Anti-Patterns

**Anti-pattern 1: Silent suppression**
```python
try:
    result = fetch_journal_data(query)
except Exception:
    return []  # ← SILENT SUPPRESSION
```
Returns empty results marked as success. The coordinator has no idea the source was unavailable. Synthesis proceeds as if no journals exist on this topic. The error is invisible and unrecoverable.

**Anti-pattern 2: Workflow termination**
```python
try:
    result = fetch_journal_data(query)
except Exception:
    raise SystemExit("Pipeline failed")  # ← NUCLEAR OPTION
```
Throws away all partial results from agents that succeeded. One transient failure in one source destroys the entire research run.

**Correct pattern: structured propagation with partial results preserved**
```python
try:
    result = fetch_journal_data(query)
except TransientError as e:
    return ErrorResult(
        type="transient",
        attempted=query,
        partial_results=cache.get_recent(query),
        retry_recommended=True,
        error_detail=str(e)
    )
```

### Access Failure vs Valid Empty Result

The exam tests this distinction directly:

| Scenario | What happened | Correct action |
|----------|---------------|----------------|
| Tool returned HTTP 503, no data | Access failure — source unreachable | Retry. Note gap in output. |
| Tool returned HTTP 200, zero results | Valid empty result — source has no matching data | No retry. This IS the answer. Report it. |

**Confusing these is a critical error.** Retrying a valid empty result wastes resources and delays the pipeline. Treating an access failure as a valid empty result silently drops a source.

### Coverage Annotations

When a synthesis agent produces output, it must annotate **what it could not cover and why**:

```
## Renewable Energy Report

### Solar Energy ✓ (well-supported — 12 sources)
[content]

### Wind Energy ✓ (well-supported — 8 sources)
[content]

### Geothermal Energy ⚠️ (limited coverage)
Note: Academic journal access was unavailable during this session.
Coverage based on 2 web sources only. Findings should be verified.

### Tidal Energy ✗ (not covered)
Note: No accessible sources found. This gap should be addressed in follow-up research.
```

Silent omission is never acceptable. The consumer of the report must know what they are not getting.

### Check Questions
1. A research subagent queries a database and receives a 200 response with an empty results array. The coordinator retries three times. Is this correct?
2. A pipeline has 6 subagents. Agent 4 fails with a timeout. The coordinator receives a structured error from agent 4 but continues with the results from agents 1–3 and 5–6, noting the gap in output. Is this correct?

---

## Task Statement 5.4: Codebase Exploration

### Context Degradation in Extended Sessions

In long codebase exploration sessions, context degrades in a specific, recognisable pattern:

```
Early in session:
"The authentication service uses JWT tokens with RS256 signing in AuthService.java line 847"

Late in session (degraded):
"The authentication service typically uses token-based auth, as is common in Java applications"
```

The model has shifted from **specific discovered facts** to **generic pattern-based reasoning**. This happens because:
- The context window fills with verbose tool output from earlier exploration
- Earlier specific findings are pushed toward the "lost in the middle" zone
- The model compensates with plausible-sounding generalisations

### Mitigation Strategies

**Scratchpad files** — write key findings to a file as they are discovered:
```
# findings.md
## Authentication
- JWT RS256 in AuthService.java:847
- Token expiry: 24h hardcoded (see config/auth.yml:12)
- Refresh logic: RefreshTokenService.java:203

## Database Layer
- ORM: Hibernate 6.2
- Connection pool: HikariCP, max 20 connections (db.properties:34)
```
Reference this file rather than relying on in-context memory. The file persists; the context does not.

**Subagent delegation** — spawn subagents for specific investigations:
```
Main agent: "Investigate the payment flow end-to-end"
  → Subagent A: "Map all entry points to PaymentController"
  → Subagent B: "Trace database writes in payment processing"
  → Subagent C: "Identify all external API calls in payment flow"
```
Each subagent has a fresh context window for its specific task. Main agent holds high-level coordination only.

**Summary injection** — summarise findings from one phase before starting the next:
```
Phase 1 complete. Summary: [structured summary of findings]
Phase 2 begins with this summary as context, not the raw Phase 1 output.
```

**/compact** — built-in Claude Code command that reduces context usage when it fills with verbose discovery output. Use when the session feels like it is losing grip on earlier findings.

### Crash Recovery

For long multi-agent pipelines, build crash recovery into the architecture:

```python
# Each agent exports state on completion (or partial completion)
agent.export_manifest("./state/agent_3_manifest.json")

# Manifest structure:
{
  "agent_id": "research_agent_3",
  "status": "completed",
  "completed_at": "2026-03-27T14:32:00Z",
  "findings": [...],
  "sources_searched": [...],
  "gaps": [...]
}
```

On crash/resume, the coordinator loads all available manifests and reconstructs state:
```python
manifests = load_all_manifests("./state/")
coordinator_context = build_context_from_manifests(manifests)
# Only spawn subagents for work not yet completed
```

This prevents full pipeline restarts when one agent fails late in a long run.

### Check Questions
1. A codebase exploration session starts producing answers like "this follows typical Spring Boot patterns" instead of specific class references it identified earlier. What has happened and what is the correct mitigation?
2. A 6-agent research pipeline crashes at agent 5 after 3 hours. Without crash recovery architecture, what happens? What does crash recovery change?

---

## Task Statement 5.5: Human Review and Confidence Calibration

### The Aggregate Metrics Trap

**The dangerous headline:** "Our extraction pipeline achieves 97% accuracy."

**What it hides:**

| Document Type | Accuracy |
|---------------|----------|
| Standard invoices | 99.8% |
| Handwritten invoices | 58% |
| Multi-currency invoices | 71% |
| Invoices in German | 63% |

**Overall accuracy: 97%** — because handwritten invoices are 2% of volume.

Before automating any extraction pipeline, validate accuracy by:
- Document type
- Field segment (dates, monetary values, addresses separately)
- Language / locale
- Age of document (older formats differ)

A pipeline that is 99% accurate on standard invoices but 58% accurate on handwritten ones will fabricate data on every handwritten invoice that reaches it.

### Stratified Random Sampling

Even after deployment, ongoing verification is required:

```
Monthly verification sample:
- 50 randomly selected high-confidence extractions (confidence > 0.90)
- 20 randomly selected medium-confidence extractions (0.70–0.90)
- 10 randomly selected low-confidence extractions (< 0.70)
```

**Why sample high-confidence extractions?** The model's confidence scores are not perfectly calibrated. Novel error patterns can emerge at high confidence. Silent degradation — where accuracy drops gradually over months — is only detected through ongoing sampling.

### Field-Level Confidence Calibration

Apply confidence at the **field level**, not the document level:

```json
{
  "vendor_name": { "value": "Acme Ltd", "confidence": 0.99 },
  "invoice_date": { "value": "2026-03-15", "confidence": 0.95 },
  "total_amount": { "value": 1250.00, "confidence": 0.88 },
  "vat_number": { "value": "GB123456789", "confidence": 0.61 },
  "payment_terms": { "value": null, "confidence": null }
}
```

**Routing logic:**
```
confidence >= 0.90  → Auto-approve field
confidence 0.70–0.89 → Flag for spot-check (10% of these reviewed)
confidence < 0.70   → Route to human reviewer
```

**Calibration process:**
1. Build a labelled validation set — 200+ documents with ground truth for every field
2. Run pipeline on validation set
3. For each confidence band, measure actual precision
4. Adjust thresholds until routing produces acceptable false negative / false positive balance
5. Re-calibrate quarterly or after significant prompt changes

**Priority rule:** With limited reviewer capacity, route highest-uncertainty items first — not oldest, not highest value. Uncertainty is the signal.

### Check Questions
1. An extraction pipeline reports 96% overall accuracy. The team automates it fully. Three months later, finance discovers fabricated data on handwritten supplier invoices. What went wrong in the validation process?
2. A document has overall extraction confidence of 0.94 but the `vat_number` field has confidence 0.52. Should the document be auto-approved?

---

## Task Statement 5.6: Information Provenance

### Structured Claim-Source Mappings

Every claim in a multi-agent research output must carry its provenance through the entire pipeline:

```json
{
  "claim": "Global geothermal capacity reached 15.6 GW in 2023",
  "source_url": "https://irena.org/publications/2024/geothermal-capacity",
  "document_name": "IRENA Geothermal Power Report 2024",
  "relevant_excerpt": "Total installed geothermal capacity reached 15,608 MW by end of 2023...",
  "publication_date": "2024-02",
  "source_type": "primary_report",
  "retrieved_by": "research_agent_2"
}
```

**The provenance death chain** — what happens without structured mappings:

```
Research agent 2 finds: "15.6 GW in 2023 (IRENA 2024)"
    ↓ passes to synthesis agent as prose
Synthesis agent writes: "geothermal capacity is approximately 15 GW"
    ↓ passes to report formatter
Final report: "geothermal capacity is significant"
    ↓
Reviewer asks: "where did this come from?"
    → Cannot be traced. Source lost.
```

Structured mappings survive synthesis. Prose summaries do not.

### Conflict Handling

Two credible sources report different statistics. The wrong response is to pick one arbitrarily.

**Wrong:**
```
Source A: "15.6 GW (IRENA 2024)"
Source B: "14.9 GW (IEA 2023)"
Agent picks: "15.6 GW" — discards Source B silently
```

**Correct:**
```json
{
  "claim": "Global geothermal installed capacity",
  "values": [
    { "value": "15.6 GW", "source": "IRENA", "date": "2024-02", "scope": "end of 2023" },
    { "value": "14.9 GW", "source": "IEA", "date": "2023-11", "scope": "mid-2023" }
  ],
  "conflict_detected": true,
  "conflict_note": "Difference likely reflects different measurement dates (mid-year vs end-year). Both values preserved."
}
```

The consumer of the report decides which figure to use and for what purpose. The agent's job is to preserve both, not arbitrate.

### Temporal Awareness

Different dates explain different numbers. What looks like a conflict is often a timeline:

```
"Solar capacity: 1.2 TW" — IEA, 2022 annual report
"Solar capacity: 1.6 TW" — IRENA, 2024 annual report
```

These are not contradictory. They are a two-year growth curve. Without publication dates in structured outputs, downstream systems treat these as a conflict and either pick one or discard both.

**Require publication/data collection dates in all structured outputs.** This is not optional metadata — it is essential for correct interpretation.

### Content-Appropriate Rendering

Do not flatten all output into one uniform format. Match the format to the content type:

| Content Type | Format | Reason |
|-------------|--------|--------|
| Financial data, statistics | Tables | Enables direct comparison |
| News, narrative findings | Prose | Preserves context and nuance |
| Technical findings, code issues | Structured lists | Scannable, actionable |
| Conflicting sources | Side-by-side comparison | Shows both values clearly |
| Timeline data | Chronological list or chart | Shows trajectory |

**Exam trap:** An answer option that proposes "use consistent JSON format for all output" sounds like good engineering. It is wrong for human-facing reports where readability matters and format should serve content.

### Check Questions
1. A synthesis agent receives findings from 4 research agents and produces a 3,000-word report. A fact-checker cannot trace any claim back to its source. What architectural element was missing?
2. Two sources report different renewable energy percentages for the same country. One is from 2022, one from 2024. How should this be handled?

---

## Domain 5 Exam Traps — Quick Reference

| Trap | Correct Answer |
|------|---------------|
| Summarise conversation history to save tokens | Extract transactional facts into persistent case facts block — never summarise those |
| Escalate on customer frustration | Only escalate on explicit human request, policy gap, or inability to progress |
| Escalate on model's low confidence score | Model confidence is miscalibrated — use structured triggers, not self-reported confidence |
| Retry on valid empty result (200, zero rows) | No retry needed — this IS the answer (no data found) |
| Silent suppression of tool failure | Structured error propagation with partial results preserved |
| Terminate pipeline on single agent failure | Continue with available results, annotate gap, structured error for failed agent |
| 96% overall accuracy = safe to automate | Validate by document type and field segment before automating |
| One session for generation and review | Independent review instance (also in Domain 3 and 4) |
| Arbitrarily pick one of two conflicting sources | Preserve both with attribution and conflict_detected flag |
| Pick "most likely" customer from ambiguous match | Ask for additional identifiers — never select by heuristic |

---

## 6-Question Practice Exam

*(Attempt before reading answers)*

**Q1 (5.1)** — A customer support agent handles a 15-turn conversation about a disputed charge of £312.50 on order #7734. After turn 8, the agent summarises prior context to manage token usage. By turn 15, the agent offers to "resolve the billing issue" without specifying the amount. What is the root cause and fix?

A) The context window is too small; switch to a model with a larger context window  
B) Transactional facts were included in the progressive summary and compressed to vague prose; extract them into a persistent case facts block that is never summarised  
C) The agent should use tool_use to re-query the order at turn 15 to recover the specific details  
D) The agent needs few-shot examples showing how to reference order details in later turns  

---

**Q2 (5.2)** — A customer contacts support about a delayed parcel. After two turns of troubleshooting, they say: "I've been waiting 20 minutes and this is completely unacceptable. Just sort it out." The agent escalates to a human. Was this correct?

A) Yes — customer frustration is a valid escalation trigger  
B) Yes — the agent has spent two turns without resolving the issue, triggering the "inability to progress" condition  
C) No — frustration alone is not a valid trigger; the agent should acknowledge frustration and offer resolution, escalating only if the customer reiterates preference for a human  
D) No — the agent should have escalated after the first turn to prevent frustration from escalating further  

---

**Q3 (5.3)** — A research pipeline has 5 subagents. Subagent 3 queries an academic journal database and receives an HTTP 200 response with an empty results array. The coordinator retries subagent 3 four times. Is this correct?

A) Yes — empty results may indicate a transient issue; retrying is appropriate  
B) Yes — academic databases sometimes require multiple queries to populate results  
C) No — an HTTP 200 with empty results is a valid response indicating no matching data; retrying wastes resources and the empty result should be reported as-is  
D) No — the coordinator should terminate the pipeline and resubmit with a different query  

---

**Q4 (5.4)** — A developer is in a 2-hour Claude Code session exploring a large codebase. Early in the session, Claude correctly identified that authentication uses RS256 JWT tokens in `AuthService.java`. An hour later, Claude says "the authentication is likely using standard JWT patterns, as is typical for Java applications." What is the most likely cause?

A) Claude Code updated its model mid-session and lost prior findings  
B) Context degradation — the context window filled with verbose discovery output, pushing earlier specific findings toward the middle; the model compensated with generic pattern-based reasoning  
C) The developer asked a different question that triggered a different answer  
D) JWT authentication details are filtered from Claude Code's output for security reasons  

---

**Q5 (5.5)** — An invoice extraction pipeline achieves 98% accuracy on the validation set and is fully automated. Three months post-deployment, the finance team finds systematic errors on multi-currency invoices. The validation set had 500 invoices, of which 8 were multi-currency. What went wrong?

A) The validation set was too small overall; 500 documents is insufficient  
B) The pipeline needed more few-shot examples for multi-currency invoices  
C) Accuracy was validated at the aggregate level only; the 8 multi-currency documents were not enough to detect category-specific errors  
D) The model confidence thresholds were set too low, allowing low-confidence extractions to be auto-approved  

---

**Q6 (5.6)** — A multi-agent research system produces a report on energy policy. Source A (published 2022) states renewable penetration at 28%. Source B (published 2024) states 34%. The synthesis agent uses 34% and discards the 2022 figure as outdated. What is the correct handling?

A) Correct — always use the most recent figure when sources conflict  
B) Incorrect — both values should be preserved with their publication dates and a conflict_detected flag; the difference likely reflects genuine growth over two years, not a contradiction  
C) Incorrect — neither value should be used; conflicting sources should be excluded from the report  
D) Incorrect — the agent should query a third source to determine which figure is correct  

---

## Practice Exam Answers

| Q | Answer | Key Reason |
|---|--------|------------|
| 1 | **B** | Progressive summarisation compresses transactional facts. Fix: persistent case facts block, never summarised. |
| 2 | **C** | Frustration ≠ explicit human request. Acknowledge + offer resolution first. Escalate only on reiteration. |
| 3 | **C** | HTTP 200 + empty array = valid empty result. Not a transient failure. Retrying is incorrect. |
| 4 | **B** | Context degradation. Verbose discovery output fills context; specific findings go to "lost in the middle." |
| 5 | **C** | Aggregate accuracy masked category-specific failure. 8 multi-currency docs is statistically insufficient. |
| 6 | **B** | Different dates = timeline, not contradiction. Preserve both with attribution and temporal context. |

**Scoring:**
- 6/6 — Exam-ready on Domain 5
- 5/6 — Review the missed task statement; re-test with one additional scenario
- Below 5 — Work through missed task statements; focus especially on 5.2 (escalation logic) and 5.3 (error propagation)

---

## Build Exercise

Build a coordinator with two subagents demonstrating all Domain 5 reliability concepts:

### Architecture

```
Coordinator
├── SubAgent A: Web Search
├── SubAgent B: Document Analysis
├── Persistent case facts block (never summarised)
├── Structured error propagation (simulate timeout on SubAgent A)
├── Coverage annotations in synthesis output
└── Claim-source mappings preserved through synthesis
```

### 1. Case Facts Block (Customer Support Scenario)
```python
CASE_FACTS = """
CASE FACTS [DO NOT SUMMARISE — INCLUDE VERBATIM]:
- Customer: James Okafor, account #44821
- Order: #9923, placed 2026-03-01
- Item: Standing desk, SKU FN-8841, £649.00
- Reported issue: Arrived with missing bolts (bags B and C)
- Prior contacts: ref #7731 (2026-03-05), ref #7799 (2026-03-08)
- Pending action: Replacement parts dispatched 2026-03-09, not yet confirmed received
"""
```

### 2. Simulated Timeout with Structured Error Propagation
```python
def web_search_agent(query):
    try:
        result = search(query, timeout=10)
        return { "status": "success", "results": result }
    except TimeoutError:
        return {
            "status": "error",
            "error_type": "transient",
            "what_was_attempted": query,
            "partial_results": cache.get(query, []),
            "retry_recommended": True,
            "coverage_note": f"Web search unavailable. Findings in this area are limited to cached results only."
        }
```

### 3. Conflicting Sources — Preserved Attribution
```python
def synthesise(findings):
    conflicts = detect_conflicts(findings)
    for conflict in conflicts:
        # Never pick one — preserve both
        conflict["resolution"] = "both_preserved"
        conflict["conflict_detected"] = True
        conflict["note"] = "Consumer should select based on context and date"
    return findings  # With conflict annotations intact
```

### Test Case
Submit a query that will:
1. Return conflicting statistics from two sources with different dates
2. Trigger a timeout in the web search agent
3. Require the persistent case facts block across 8+ turns

Verify that:
- Final output preserves both conflicting values with attribution
- Coverage annotation notes the web search gap
- Case facts remain precise in turn 8 output

---

## Summary: The Six Things You Must Know for Domain 5

1. **Never summarise transactional facts.** Case facts block travels verbatim through every prompt.
2. **Frustration ≠ escalation.** Only three valid triggers: explicit human request, policy gap, inability to progress.
3. **HTTP 200 + empty array ≠ failure.** Valid empty result. Do not retry.
4. **Structured errors, not silence or termination.** Propagate partial results; annotate gaps; continue.
5. **Aggregate accuracy lies.** Validate by document type and field segment before automating anything.
6. **Conflicting sources: preserve both with attribution.** Never arbitrarily select one.

