# Domain 2: Tool Design & MCP Integration — Complete Study Guide

**Claude Certified Architect (Foundations) | Exam Weight: 18%**

> 🇬🇧 This guide uses British English spelling throughout (e.g., "normalise", "optimise", "favour").

---

## How to Use This Guide

This guide is structured as a teach-then-check walkthrough. For each of the 5 task statements you will find:
1. **Concept explanation** with a concrete production example
2. **Exam traps** — specific anti-patterns and misconceptions that appear as distractors
3. **Practice scenario** with a worked answer
4. **Connection** to the next task statement

After all 5 task statements, a **7-question practice exam** covers the full domain. Score yourself, map gaps to task statements, and revisit weak areas.

---

## Self-Assessment: Where Do You Start?

Rate your familiarity before diving in:

| Rating | Description | How to Use This Guide |
|--------|-------------|----------------------|
| **Novice** | No experience with MCP or tool design | Read every section word for word. Do every practice scenario before reading the answer. |
| **Intermediate** | Used MCP tools in an IDE or agent | Focus on the Exam Traps boxes and practice scenarios. Skim concept explanations. |
| **Advanced** | Built MCP servers or designed tool APIs for agents | Jump to practice scenarios and the final exam. Use task statement explanations to fill gaps. |

---

## Domain Overview

- **Exam weight:** 18%
- **Passing score:** 720 / 1000
- **Format:** Scenario-based multiple choice. One correct answer. Three plausible distractors.
- **Primary exam scenarios:** Customer Support Resolution Agent · Multi-Agent Research System · Developer Productivity Tools

> **📌 Core Exam Philosophy for Domain 2**
>
> The exam consistently rewards:
> 1. **Low-effort, high-leverage fixes first** — improve tool descriptions before building routing classifiers
> 2. **Scoped access before full access** — constrained tools before general-purpose tools
> 3. **Community servers before custom builds** — use existing MCP servers before writing your own
> 4. **Root cause fixes** — if the problem is a bad description, fix the description, not the plumbing around it

---

## Task Statement 2.1: Tool Interface Design

### Tool Descriptions Are THE Mechanism

This is the single most important concept in Domain 2. Understand it at a bone-deep level.

When Claude decides which tool to call, it reads the **tool descriptions** you provide in the API request. That is the primary — and often the only — mechanism the model uses for tool selection. Descriptions are not documentation for humans. They are **selection instructions for the model**.

If your descriptions are minimal, the model cannot differentiate similar tools. It will misroute. It will guess. It will pick the wrong one.

This is not a "nice to have." This is **the mechanism**.

---

### What a Good Tool Description Includes

A production-grade tool description answers five questions:

| Component | What It Does | Example |
|-----------|-------------|---------|
| **Primary purpose** | What the tool does in one sentence | "Retrieves the current status, shipping carrier, and estimated delivery date for an order given its order ID." |
| **Input specification** | Formats, types, constraints on each parameter | "order_id: string, format 'ORD-XXXXX' where X is a digit. Required." |
| **Example queries** | Concrete natural-language requests this tool handles well | "Use this tool when the customer asks: 'Where is my order?', 'When will order #12345 arrive?', 'Has my package shipped?'" |
| **Edge cases and limitations** | What the tool cannot do or handles poorly | "Does NOT return order line items or pricing. Use get_order_details for itemised information." |
| **Explicit boundaries** | When to use THIS tool vs similar tools | "Use this tool for order STATUS queries only. For customer account information (name, email, address), use get_customer_profile instead." |

Here is a **bad** description versus a **good** one:

**❌ Bad — Minimal description:**
```json
{
  "name": "get_customer",
  "description": "Retrieves customer information."
}
```

**❌ Also bad — Minimal description:**
```json
{
  "name": "lookup_order",
  "description": "Retrieves order information."
}
```

These two descriptions are nearly identical in structure. "Retrieves [entity] information" gives the model almost nothing to differentiate them. When a customer asks "check the status of order #12345," the model sees two tools that both "retrieve information" and has to guess.

**✅ Good — Expanded descriptions:**
```json
{
  "name": "get_customer",
  "description": "Retrieves a customer's profile information including name, email address, phone number, and account status. Input: customer_id (string, format 'CUST-XXXXX') OR email (string, valid email format). Use this tool when the customer asks about their account details, contact information, or membership status. Do NOT use this tool for order tracking, billing, or purchase history — use lookup_order or get_billing_history instead."
}
```

```json
{
  "name": "lookup_order",
  "description": "Retrieves the current status, shipping carrier, tracking number, and estimated delivery date for a specific order. Input: order_id (string, format 'ORD-XXXXX'). Use this tool when the customer asks about order status, shipping progress, delivery timing, or package tracking. Handles queries like 'Where is my order?', 'Has order #12345 shipped?', 'When will my package arrive?'. Do NOT use this tool for customer profile information — use get_customer instead."
}
```

Now the model has unambiguous signals: order status queries → `lookup_order`, profile queries → `get_customer`.

---

### The Misrouting Problem

> **⚠️ EXAM CRITICAL — This pattern appears directly in the exam (Q2 in the sample set)**

**Scenario:** An agent routes "check the status of order #12345" to `get_customer` instead of `lookup_order`. Both tools have minimal descriptions: "Retrieves customer information" and "Retrieves order information."

**The exam presents four fixes:**

| Option | Why It's Wrong or Right |
|--------|------------------------|
| **A) Add few-shot examples to the system prompt** | ❌ Wrong root cause. Few-shot examples consume significant token overhead and address symptoms (misrouting behaviour) rather than the cause (indistinguishable descriptions). The model still has bad descriptions — you are just trying to override them with prompt engineering. |
| **B) Build a routing classifier** | ❌ Over-engineered first step. A routing classifier is a separate model or rules engine that decides which tool to use before the main model runs. This adds latency, complexity, and a new component to maintain. It is not warranted when the root cause is a simple description problem. |
| **C) Expand tool descriptions to include specific use cases and explicit boundaries** | ✅ **Correct.** Low effort, high leverage, addresses the root cause directly. |
| **D) Consolidate both tools into a single tool** | ❌ Too much effort and reduces specificity. A single combined tool loses the clarity of purpose-specific interfaces. |

**Exam principle:** Always pick the simplest fix that addresses the root cause. Better descriptions before routing classifiers. Always.

---

### Tool Splitting

When a tool is too generic, split it into purpose-specific tools with defined input/output contracts.

**❌ Generic tool:**
```json
{
  "name": "analyze_document",
  "description": "Analyses a document."
}
```

This tool does three different things depending on the input, which means the model (and the developer) cannot reason about what it will return for any given call.

**✅ Split into purpose-specific tools:**

```json
{
  "name": "extract_data_points",
  "description": "Extracts structured numerical data points and statistics from a document. Input: document_id (string), data_categories (array of strings, e.g., ['revenue', 'headcount', 'dates']). Returns a JSON array of {category, value, unit, page_number, confidence_score} objects. Use when you need specific facts or figures from a document. Do NOT use for summaries or verification — use summarise_content or verify_claim_against_source."
}
```

```json
{
  "name": "summarise_content",
  "description": "Generates a structured summary of a document's key arguments and conclusions. Input: document_id (string), max_length (integer, default 500 words), focus_areas (optional array of strings). Returns {executive_summary, key_arguments[], conclusions[], limitations_noted[]}. Use when you need an overview of what a document says. Do NOT use for data extraction or claim verification."
}
```

```json
{
  "name": "verify_claim_against_source",
  "description": "Checks whether a specific factual claim is supported, contradicted, or unaddressed by a given source document. Input: claim (string, the assertion to verify), document_id (string). Returns {verdict: 'supported'|'contradicted'|'not_addressed', relevant_excerpts[], confidence_score}. Use when you need to fact-check a claim against a source. Do NOT use for summarisation or data extraction."
}
```

Each tool now has:
- A single, clear purpose
- Defined input types and constraints
- A structured output contract
- Explicit boundaries against the other two tools

---

### System Prompt Interaction

> **⚠️ EXAM TRAP — Subtle but tested**

Keyword-sensitive instructions in system prompts can create unintended tool associations that **override** well-written tool descriptions.

**Example:** Your system prompt says:
```
When the user mentions an "order", always look up the customer first to verify identity.
```

Now, even with a perfect `lookup_order` description, the keyword "order" in the user's message triggers the system prompt rule, and Claude calls `get_customer` first — even when the user simply wants tracking information and has already been verified.

**The fix:** After updating tool descriptions, always review the system prompt for conflicts. Look for:
- Keywords that match tool names or domains
- Imperative instructions ("always", "never", "first") that create hard rules overriding tool selection
- Legacy instructions that no longer match the current tool set

---

### 🧪 Practice Scenario 2.1 — The Misrouted Order Query

**Scenario:** A customer support agent has two tools:

- `get_customer` — description: "Retrieves customer information."
- `lookup_order` — description: "Retrieves order information."

A customer asks: "Check the status of order #12345." The agent calls `get_customer` instead of `lookup_order`. This happens consistently across different order queries.

Your team proposes four fixes:

- **A)** Add few-shot examples showing the correct tool for order queries
- **B)** Build a routing classifier that examines the query and selects the tool before sending to Claude
- **C)** Expand both tool descriptions to include specific use cases, input formats, and explicit boundaries (e.g., "Use this for order STATUS queries, NOT customer profile queries")
- **D)** Merge both tools into a single `get_info` tool that accepts a type parameter

Which fix should you implement first?

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: C**

The root cause is that the descriptions are nearly identical — "Retrieves [entity] information." The model has insufficient signal to differentiate them. Expanding descriptions is:
- **Low effort:** change two JSON strings
- **High leverage:** directly addresses the root cause
- **No new infrastructure:** no classifier, no merged tool, no token-heavy few-shot examples

Option A adds token overhead without fixing the root cause. Option B is over-engineered for a description problem. Option D reduces tool specificity and requires refactoring.

After implementing C, if misrouting persists, check the system prompt for keyword-sensitive instructions that might be overriding the improved descriptions.

</details>

---

*This connects to Task 2.2: once your tools are correctly selected, you need to handle what happens when they fail.*

---

## Task Statement 2.2: Structured Error Responses

### The MCP isError Flag

When a tool fails, the result must communicate this clearly to the agent so it can decide what to do next. In the Model Context Protocol (MCP), tool results include an `isError` boolean flag:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "content": "...",
  "is_error": true
}
```

When `is_error` is `true`, Claude understands the tool call failed and will attempt recovery strategies rather than treating the error content as a successful result. Without this flag, Claude may try to interpret an error message as valid data, producing nonsensical responses.

---

### The Four Error Categories

Every tool error falls into one of four categories. The exam tests your ability to classify errors correctly because the classification determines the correct recovery action.

| Category | Examples | Retryable? | Correct Recovery |
|----------|----------|------------|-----------------|
| **Transient** | Timeout, service temporarily unavailable, rate limit hit, network blip | ✅ Yes | Retry with exponential backoff. The same request will likely succeed shortly. |
| **Validation** | Invalid input format, missing required field, value out of range | ✅ Yes (after fixing input) | Fix the input and retry. Do NOT retry with the same input — it will fail identically. |
| **Business** | Refund exceeds policy limit, account suspended, product discontinued | ❌ No | Needs an alternative workflow. Inform the customer. Escalate if necessary. Retrying is pointless — the policy will not change. |
| **Permission** | Access denied, insufficient role, expired credentials, IP blocked | ❌ No (for current credentials) | Needs escalation or different credentials. The agent cannot resolve this itself. |

---

### Structured Error Metadata

Return structured error metadata — not just a string message. This gives the agent everything it needs to make recovery decisions programmatically.

**❌ Bad — Unstructured error:**
```json
{
  "is_error": true,
  "content": "Error: something went wrong"
}
```

The agent cannot determine whether to retry, escalate, or inform the customer.

**✅ Good — Structured error with metadata:**
```json
{
  "is_error": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Order service timed out after 5000ms.",
    "suggestedAction": "Retry after 2 seconds.",
    "attemptNumber": 1,
    "maxRetries": 3
  }
}
```

```json
{
  "is_error": true,
  "content": {
    "errorCategory": "business",
    "isRetryable": false,
    "message": "Refund of £450 exceeds the maximum automated refund limit of £200.",
    "customerFriendlyMessage": "I'm sorry, but refunds above £200 require manager approval. I'll escalate this to a manager who can process it within 24 hours.",
    "suggestedAction": "Escalate to manager queue with refund details.",
    "policyReference": "REFUND-POLICY-2024-v3, Section 4.2"
  }
}
```

Notice the `customerFriendlyMessage` on the business error. The agent cannot retry — so it needs a message it can relay to the customer. Including `isRetryable: false` explicitly prevents any retry logic from firing.

---

### 🔴 The Critical Distinction: Access Failure vs Valid Empty Result

> **⚠️ EXAM CRITICAL — This distinction is directly tested**

These are two fundamentally different outcomes that look superficially similar:

| Outcome | What Happened | Correct Agent Behaviour |
|---------|---------------|------------------------|
| **Access failure** | The tool could NOT reach the data source (timeout, auth failure, network error) | Decide whether to retry. The data may exist — we just could not reach it. |
| **Valid empty result** | The tool successfully queried the source and found **no matches** | Do NOT retry. The answer IS "no results." Retrying will return the same empty result. |

**How they should look in tool responses:**

**Access failure:**
```json
{
  "is_error": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Database connection timed out after 5000ms.",
    "resultCount": null
  }
}
```

**Valid empty result:**
```json
{
  "is_error": false,
  "content": {
    "results": [],
    "resultCount": 0,
    "queryExecuted": true,
    "message": "No customer found matching email 'nonexistent@example.com'."
  }
}
```

The valid empty result has `is_error: false` because the tool **worked correctly**. It successfully queried the database and the answer is: this customer does not exist. The `queryExecuted: true` flag makes the distinction explicit.

**If you confuse these two, recovery logic breaks:**
- Treating access failure as "no results" → the agent tells the customer their account doesn't exist when actually the database was just down
- Treating valid empty result as access failure → the agent retries 3 times, wastes time, then escalates to a human for a "failure" that is actually a correct result

---

### Error Propagation in Multi-Agent Systems

In multi-agent architectures (covered in Domain 1), errors propagate from subagents back to the coordinator. The rules:

1. **Subagents implement local recovery for transient failures.** If a web search times out, the subagent retries locally with backoff — it does NOT immediately propagate to the coordinator.

2. **Only propagate errors the subagent cannot resolve locally.** After exhausting local retries, the subagent reports the failure to the coordinator with context about what was attempted.

3. **Include partial results when propagating.** If the subagent completed 3 of 5 searches before failing, send the 3 successful results along with the error. The coordinator can decide whether partial results are sufficient or whether to re-assign the failed queries.

**✅ Good error propagation from subagent to coordinator:**
```json
{
  "status": "partial_failure",
  "completedTasks": [
    {"query": "solar energy market 2024", "results": ["..."]},
    {"query": "wind energy capacity Europe", "results": ["..."]}
  ],
  "failedTasks": [
    {
      "query": "geothermal energy economics",
      "errorCategory": "transient",
      "attemptsExhausted": 3,
      "lastError": "Service unavailable (HTTP 503)"
    }
  ],
  "recommendation": "Partial results available. Re-assign failed query or proceed with available data."
}
```

---

### 🧪 Practice Scenario 2.2 — The Infinite Retry Loop

**Scenario:** A customer support tool returns the following when looking up a customer by email:

```json
{
  "results": [],
  "message": ""
}
```

The agent receives this response, interprets it as a potential failure, retries the lookup 3 times, gets the same empty result each time, and then escalates to a human agent with: "Unable to retrieve customer data after multiple attempts."

The human agent investigates and finds that the customer's account simply does not exist — they are a new user trying to create an account.

1. What is the root problem?
2. How should the tool response be structured to prevent this?

<details>
<summary>▶ Show Answer</summary>

**Root problem:** The tool's response does not distinguish between "I successfully queried the database and found nothing" (valid empty result) and "I failed to reach the database" (access failure). The empty array with no metadata is ambiguous. The agent defensively treats ambiguity as failure and retries.

**Fix:** Structure the response to make the distinction explicit:

```json
{
  "is_error": false,
  "content": {
    "results": [],
    "resultCount": 0,
    "queryExecuted": true,
    "message": "No customer account found matching email 'newuser@example.com'. This may be a new user without an existing account."
  }
}
```

Key changes:
- `is_error: false` — the tool worked correctly
- `queryExecuted: true` — the query reached the database successfully
- `resultCount: 0` — explicitly states the count
- Descriptive message suggesting the likely explanation

The agent now knows: the query succeeded, there are zero results, and this is not an error. It can respond to the customer: "I don't see an existing account with that email. Would you like to create a new account?"

</details>

---

*This connects to Task 2.3: once your tools return clean results and structured errors, you need to decide which agents get which tools.*

---

## Task Statement 2.3: Tool Distribution and tool_choice

### The Tool Overload Problem

> **⚠️ EXAM CRITICAL — Directly tested**

Giving an agent too many tools degrades selection reliability. With 18 tools available, the model must evaluate every description against the current query, increasing the chance of misrouting.

**The optimal range: 4–5 tools per agent, scoped to its role.**

| Agent Role | Correct Tools | Wrong Tools |
|-----------|---------------|-------------|
| **Synthesis agent** | `compile_report`, `format_citations`, `verify_fact`, `assess_coverage` | ~~`web_search`~~, ~~`fetch_url`~~, ~~`query_database`~~ |
| **Web search agent** | `web_search`, `extract_page_content`, `evaluate_source_credibility` | ~~`summarise_content`~~, ~~`compile_report`~~, ~~`format_citations`~~ |
| **Document analysis agent** | `read_document`, `extract_data_points`, `summarise_content`, `verify_claim_against_source` | ~~`web_search`~~, ~~`send_email`~~, ~~`create_ticket`~~ |

**Principle:** A synthesis agent should NOT have web search tools. A web search agent should NOT have document analysis tools. Each agent gets only the tools it needs for its specific role.

---

### The tool_choice Configuration

The `tool_choice` parameter in the Messages API controls whether and how Claude uses tools. There are three settings:

#### 1. `"auto"` — Model decides (default)

```json
{
  "tool_choice": {"type": "auto"}
}
```

Claude decides whether to call a tool or return a text response. Use this for **general operation** — the vast majority of your API calls should use `auto`.

#### 2. `"any"` — Must call a tool, model chooses which

```json
{
  "tool_choice": {"type": "any"}
}
```

Claude **must** call one of the available tools — it cannot return a plain text response. Use this when you need **guaranteed structured output** from one of multiple schemas.

**Example:** You have three output schema tools (`format_as_json`, `format_as_table`, `format_as_summary`). You need structured output, but you want Claude to pick the format. Setting `tool_choice: any` guarantees a tool call.

#### 3. Named tool — Must call this specific tool

```json
{
  "tool_choice": {"type": "tool", "name": "extract_metadata"}
}
```

Claude **must** call the specified tool. It cannot choose a different tool or return text. Use this to **force mandatory first steps** before enrichment.

**Example:** In a document processing pipeline, the first step must always extract metadata (author, date, document type) before any analysis. Force `extract_metadata` on the first call, then switch to `auto` for subsequent calls:

```python
# Step 1: Forced metadata extraction
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_metadata"},
    messages=messages
)

# Step 2+: Model decides what to do next
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=tools,
    tool_choice={"type": "auto"},
    messages=messages
)
```

---

### Scoped Cross-Role Tools

> **⚠️ EXAM CRITICAL — This exact pattern appears in Q9 of the sample set**

Sometimes a subagent frequently needs a capability that "belongs" to another role. The naive solution is to route every request through the coordinator to the other agent. But this creates round-trip latency.

**The problem:**
```
Synthesis Agent needs to verify a fact
    → Returns control to Coordinator
        → Coordinator routes to Verification Agent
            → Verification Agent performs lookup
        → Coordinator receives result
    → Coordinator passes result back to Synthesis Agent

Result: 2-3 extra round trips per verification. 40% latency increase.
```

85% of these verifications are simple lookups (e.g., "Is the capital of France Paris?"). Only 15% are complex multi-step verifications.

**The solution:** Give the synthesis agent a **scoped** `verify_fact` tool for simple lookups, while complex verifications still route through the coordinator.

```json
{
  "name": "verify_fact",
  "description": "Performs a simple factual lookup to verify a single claim. Input: claim (string), source_hint (optional string). Returns {verified: boolean, source: string, confidence: float}. Use ONLY for straightforward factual claims that can be verified with a single lookup. For complex claims requiring multi-step reasoning, cross-referencing multiple sources, or domain expertise, return the claim to the coordinator for full verification."
}
```

**Key characteristics of a scoped cross-role tool:**
- **Constrained:** handles only the simple case (85% of requests)
- **Bounded:** explicit instructions about when NOT to use it
- **Avoids coordinator round-trip** for the common case
- **Preserves the full workflow** for the complex case (15%)

---

### Replacing Generic Tools with Constrained Alternatives

Instead of giving a subagent a powerful, unconstrained tool, give it a constrained version that only does what it needs.

**❌ Generic and dangerous:**
```json
{
  "name": "fetch_url",
  "description": "Fetches content from any URL."
}
```

A subagent with `fetch_url` can access any URL on the internet. If the subagent is compromised or confused, it could exfiltrate data or access internal endpoints.

**✅ Constrained and safe:**
```json
{
  "name": "load_document",
  "description": "Loads a document from the approved document store. Input: document_id (string, format 'DOC-XXXXX'). Only accepts document IDs from the internal document catalogue. Cannot fetch arbitrary URLs. Returns the document text content and metadata."
}
```

Now the tool can only access approved documents. The attack surface is dramatically reduced.

---

### 🧪 Practice Scenario 2.3 — The Latency Problem

**Scenario:** A multi-agent research system has this architecture:

- **Coordinator Agent** — orchestrates all work
- **Web Search Agent** — has `web_search` and `extract_page_content` tools
- **Document Analysis Agent** — has `read_document`, `summarise_content`, and `extract_data_points` tools
- **Synthesis Agent** — has `compile_report` and `format_citations` tools

The synthesis agent frequently needs to verify facts while compiling reports. Currently, every verification request returns control to the coordinator, which routes to the web search agent for a lookup, then passes the result back through the coordinator to the synthesis agent. This adds 2–3 round trips per verification, increasing overall latency by 40%. Analysis shows 85% of verifications are simple single-lookup factual checks.

Your team proposes four solutions:

- **A)** Give the synthesis agent all of the web search agent's tools (`web_search`, `extract_page_content`) so it can verify facts directly
- **B)** Add a scoped `verify_fact` tool to the synthesis agent that handles simple lookups, with complex verifications still routed through the coordinator
- **C)** Cache all possible facts before the synthesis step begins
- **D)** Increase the coordinator's processing speed to reduce round-trip time

Which solution is correct?

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

- **A is wrong:** Giving the synthesis agent web search tools violates the tool scoping principle. The synthesis agent now has 4+ tools outside its core competency, degrading tool selection. It also blurs role boundaries.
- **B is correct:** A scoped `verify_fact` tool handles the 85% simple case directly, eliminating 2–3 round trips for most verifications. Complex verifications (15%) still route through the coordinator for full multi-step verification. Low effort, high leverage.
- **C is wrong:** You cannot predict every fact the synthesis agent might need to verify. Pre-caching is impractical and wasteful.
- **D is wrong:** Speed optimisation treats the symptom (slow round trips) not the cause (unnecessary round trips).

</details>

---

*This connects to Task 2.4: now that you understand which tools go where, you need to know how to configure and discover them via MCP servers.*

---

## Task Statement 2.4: MCP Server Integration

### What Is MCP?

The **Model Context Protocol (MCP)** is a standardised protocol that lets agents discover and use tools exposed by external servers. Instead of hardcoding tool definitions in your API calls, you connect to MCP servers that advertise their tools, and the agent discovers them at connection time.

Think of it as a USB-like interface: plug in a server, and the agent immediately knows what tools are available.

---

### The Scoping Hierarchy

MCP configuration exists at two levels. Understanding which to use is tested on the exam.

| Level | File | Version Controlled? | Shared with Team? | Use Case |
|-------|------|---------------------|--------------------| ---------|
| **Project-level** | `.mcp.json` in the project root | ✅ Yes | ✅ Yes | Standard integrations the entire team needs (Jira, GitHub, shared databases) |
| **User-level** | `~/.claude.json` in the user's home directory | ❌ No | ❌ No | Personal tools, experimental servers, individual preferences |

**Key behaviour:** All tools from all configured servers (both project and user level) are discovered at connection time and available simultaneously. There is no lazy loading — the agent sees everything at once.

**Project-level `.mcp.json` example:**
```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-jira"],
      "env": {
        "JIRA_URL": "https://mycompany.atlassian.net",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

---

### Environment Variable Expansion

> **⚠️ EXAM DETAIL — Configuration knowledge tested**

`.mcp.json` supports `${VARIABLE_NAME}` syntax for environment variable expansion. This is how you keep credentials out of version control:

```json
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_TOKEN}",
    "DATABASE_URL": "${DATABASE_URL}"
  }
}
```

Each developer sets their own tokens locally (e.g., in their shell profile or a `.env` file that is `.gitignore`d). The `.mcp.json` file is safe to commit because it contains only variable references, not actual secrets.

---

### MCP Resources

MCP servers can expose **resources** — read-only content that gives agents visibility into available data without requiring exploratory tool calls.

**Examples of MCP resources:**
- Issue summaries from Jira (list of open issues, their statuses, assignees)
- Documentation hierarchies (table of contents, section structure)
- Database schemas (table names, column types, relationships)

**Why resources matter:** Without resources, an agent that needs to know "what Jira issues are open" has to call a search tool with a broad query, parse the results, and potentially make multiple calls to narrow down. With resources, the agent can browse a pre-indexed catalogue and make targeted queries.

Resources reduce unnecessary exploratory queries → fewer tool calls → lower latency → lower cost.

---

### The Build-vs-Use Decision

> **⚠️ EXAM CRITICAL — The exam strongly favours community servers**

| Decision | When to Choose | Example |
|----------|---------------|---------|
| **Use community MCP server** | Standard integration with a well-known service. Community server exists and is maintained. | Jira, GitHub, Slack, PostgreSQL, Google Drive |
| **Build custom MCP server** | Team-specific workflow that community servers cannot handle. Internal APIs with no public equivalent. Custom business logic. | Internal deployment pipeline, proprietary data warehouse, company-specific approval workflow |

**The exam's default answer:** Use the community server unless there is a specific, stated reason it cannot meet the requirement.

**Why build custom?**
- Your internal API has authentication requirements the community server doesn't support
- You need to enforce business rules at the tool level (e.g., automatically redacting PII before returning results)
- The integration requires combining multiple API calls into a single atomic operation that the community server doesn't offer
- You need a domain-specific data transformation that doesn't exist in any community server

---

### Enhancing MCP Tool Descriptions

A subtle but important point: when you add an MCP server, its tools compete with built-in tools for the model's attention. If an MCP tool's description is worse than a built-in tool's description, the model will prefer the built-in tool even when the MCP tool is more capable.

**Example:** Your MCP server exposes `search_codebase` with a mediocre description: "Searches code." The built-in `Grep` tool has a clear, well-written description. Claude prefers `Grep` even though `search_codebase` has semantic understanding, cross-reference resolution, and other capabilities `Grep` lacks.

**Fix:** Enhance the MCP tool's description to highlight its specific advantages:
```json
{
  "name": "search_codebase",
  "description": "Performs semantic code search across the entire repository with cross-reference resolution. Unlike text-based grep, this tool understands code structure: it can find implementations of an interface, callers of a function across modules, and usages of a type including aliased imports. Input: query (string, natural language or code pattern). Use this tool for structural code queries. Use the built-in Grep tool only for literal string matching."
}
```

---

### 🧪 Practice Scenario 2.4 — The Custom Server Proposal

**Scenario:** Your team needs to integrate their Claude-based developer assistant with Jira for issue tracking. A senior developer proposes spending two weeks building a custom MCP server that connects to Jira's REST API.

1. Why should community servers be evaluated first?
2. When would a custom build be justified?

<details>
<summary>▶ Show Answer</summary>

**Why community servers first:**
- A well-maintained community MCP server for Jira already exists
- It handles standard CRUD operations (create/read/update issues, search, transitions)
- It is tested by many users, reducing the chance of bugs
- It receives updates when Jira's API changes
- Two weeks of development time is significant cost with no guarantee of a better result
- The exam principle: community servers before custom builds, always

**When custom build IS justified:**
- The team needs Jira integration combined with an internal approval system (e.g., "creating a P1 issue must also page the on-call engineer via our internal alerting API")
- They need to enforce PII redaction on issue descriptions before the agent sees them
- They need a custom field mapping that translates between their internal data model and Jira's schema in a way the community server doesn't support
- They need to combine Jira operations with internal database writes in a single atomic transaction

**Key principle:** Start with the community server. If it falls short for a specific, documented requirement, extend it or build a custom server for just the gap — don't rebuild the whole thing from scratch.

</details>

---

*This connects to Task 2.5: beyond MCP servers, Claude has built-in tools for file operations — knowing which to use when is the final piece of Domain 2.*

---

## Task Statement 2.5: Built-In Tools

### Grep vs Glob — The Fundamental Distinction

> **⚠️ EXAM CRITICAL — The exam deliberately presents scenarios where using the wrong one fails**

These two tools look similar but do completely different things:

| Tool | What It Searches | Use For | Example |
|------|-----------------|---------|---------|
| **Grep** | File **contents** — the text inside files | Finding function callers, locating error messages, searching import statements, finding TODO comments | "Find all files that call `processPayment()`" |
| **Glob** | File **paths** — the names and locations of files | Finding files by extension, locating configuration files, finding test files by naming convention | "Find all `*.test.tsx` files" or "Find all `tsconfig*.json` files" |

**Mnemonic:** **Grep** searches what's **inside** the file. **Glob** searches what the file is **called**.

**Grep example — find function callers:**
```
Grep for "processPayment(" across the codebase
→ Returns: src/checkout.ts:45, src/subscription.ts:112, src/refund.ts:23
```

**Glob example — find test files:**
```
Glob for "**/*.test.tsx"
→ Returns: src/checkout.test.tsx, src/subscription.test.tsx, src/components/Button.test.tsx
```

**Common exam trap:** A scenario asks "find all files that import the deprecated `legacyAuth` module." The distractor answer uses Glob (`**/*legacyAuth*`) — but this finds files *named* legacyAuth, not files that *import* it. The correct answer is Grep for `import.*legacyAuth` or `require.*legacyAuth`.

---

### Read / Write / Edit

Three tools for file modification, with a clear hierarchy:

#### Edit — The First Choice

- Makes **targeted modifications** using unique text matching
- Fast and precise — changes only the specific lines you target
- Works by finding a unique text anchor in the file and replacing it

**When Edit works:**
```
Edit file: src/config.ts
Find: "maxRetries: 3"
Replace: "maxRetries: 5"
```

This works because `"maxRetries: 3"` is a unique string in the file.

#### When Edit Fails — Non-Unique Text

**Edit fails when the text match is not unique.** Example:

```python
# File has multiple instances:
result = None  # line 12
result = None  # line 45
result = None  # line 78
```

Trying to Edit `result = None` to `result = []` is ambiguous — which line should change? Edit cannot determine the target.

#### Read + Write — The Reliable Fallback

When Edit fails due to non-unique text:
1. **Read** the full file to load its complete contents
2. Make modifications in memory
3. **Write** the entire modified file back

This is slower and more token-intensive (you must load the whole file), but it always works because you are replacing the entire file contents rather than matching a specific anchor.

**Hierarchy:**
```
Try Edit first (fast, precise)
    ↓ fails (non-unique text match)
Fall back to Read + Write (reliable, but heavier)
```

---

### Incremental Codebase Understanding

> **⚠️ EXAM CRITICAL — The exam tests the correct tool sequence for code exploration**

When you need to understand how a codebase works, there is a correct and an incorrect approach:

**❌ Wrong: Read all files upfront**
```
Read src/index.ts
Read src/auth.ts
Read src/database.ts
Read src/utils.ts
Read src/middleware.ts
... (50 more files)
```

This is a **context-budget killer**. You load thousands of lines of code, most of which are irrelevant to your task. You run out of context window space before you find what you need.

**✅ Correct: Incremental discovery**

1. **Start with Grep** to find entry points:
```
Grep for "export function processPayment" → finds src/payments/processor.ts:12
```

2. **Use Read** to examine the entry point and trace its imports:
```
Read src/payments/processor.ts
→ Discover: imports validateCard from './validation'
→ Discover: imports chargeAccount from '../billing/charge'
→ Discover: imports logTransaction from '../audit/logger'
```

3. **Follow imports selectively** — only Read the files that matter for your task:
```
Read src/payments/validation.ts   (relevant — validates payment input)
Read src/billing/charge.ts         (relevant — performs the charge)
Skip src/audit/logger.ts           (not relevant to the payment logic bug)
```

4. **Use Grep again** to trace function usage across modules:
```
Grep for "chargeAccount(" → finds callers in processor.ts, subscription.ts, refund.ts
```

**The principle:** Discover → Read → Discover more → Read more. Each step is targeted. You never load files you don't need.

---

### Tracing Across Wrapper Modules

A common pattern in production codebases: functions are re-exported through index files (barrel exports).

```typescript
// src/payments/index.ts
export { processPayment } from './processor';
export { validateCard } from './validation';
```

When searching for callers of `processPayment`, your first Grep might only find the re-export in `index.ts`. You need to then search for the module path:

```
Step 1: Grep for "processPayment" → finds src/payments/index.ts (re-export), src/payments/processor.ts (definition)
Step 2: Grep for "from.*payments" or "require.*payments" → finds all consumers of the payments module
Step 3: Read the consuming files to confirm which ones actually call processPayment (vs other exports from the module)
```

---

### 🧪 Practice Scenario 2.5 — The Deprecated Function Hunt

**Scenario:** A developer needs to:
1. Find all files that call a specific deprecated function `legacyEncrypt()`
2. Find the test files for each of those callers

Walk through the correct tool sequence.

<details>
<summary>▶ Show Answer</summary>

**Step 1: Grep for the deprecated function name**
```
Grep for "legacyEncrypt(" across the entire codebase
→ Results:
  src/auth/login.ts:34          — calls legacyEncrypt
  src/auth/token.ts:67          — calls legacyEncrypt
  src/data/export.ts:112        — calls legacyEncrypt
  src/crypto/legacy.ts:5        — definition (skip this)
```

This gives us the three caller files: `login.ts`, `token.ts`, `export.ts`.

**Step 2: Glob for test files matching the caller filenames**
```
Glob for "**/*login*.test.*"    → src/auth/login.test.ts
Glob for "**/*token*.test.*"    → src/auth/token.test.ts
Glob for "**/*export*.test.*"   → src/data/export.test.ts, src/api/export.test.ts
```

Note: `export` matches two test files. You might need to Read both to determine which one tests `src/data/export.ts` specifically.

**Why this sequence is correct:**
- Grep finds callers (searches file **contents** for the function name)
- Glob finds test files (searches file **paths** for matching names)
- Using Glob to find callers would fail — it would find files *named* legacyEncrypt, not files that *call* it
- Using Grep to find test files would work but is less efficient — you'd search inside every file for import statements, when the naming convention gives you the answer directly

</details>

---

*This completes all five task statements. Now let's test your knowledge with a practice exam.*

---

## Domain 2 Practice Exam — 7 Questions

### Question 1 (Task 2.1 — Tool Descriptions)

A customer support agent has three tools:

- `get_account_info` — "Gets account information"
- `get_billing_info` — "Gets billing information"
- `get_subscription_info` — "Gets subscription information"

When a customer asks "Why was I charged £49.99 last month?", the agent calls `get_account_info` instead of `get_billing_info`. What is the most effective first step to fix this?

- **A)** Add a pre-processing classifier that routes billing keywords to `get_billing_info`
- **B)** Expand all three tool descriptions to include specific use cases, example queries, input formats, and explicit boundaries between each tool
- **C)** Consolidate all three tools into a single `get_customer_data` tool with a `type` parameter
- **D)** Add few-shot examples to the system prompt showing the correct tool for billing queries

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

The root cause is indistinguishable descriptions — all three follow the pattern "Gets [entity] information." Expanding descriptions is the lowest-effort, highest-leverage fix that addresses the root cause directly.

A is over-engineered for a description problem. C reduces specificity. D adds token overhead without fixing the root cause.

</details>

---

### Question 2 (Task 2.1 — System Prompt Conflict)

After improving tool descriptions, a developer notices that the agent still calls `get_customer_profile` before `lookup_order` for every order-related query. The tool descriptions are now clear and distinct. What should the developer investigate next?

- **A)** The model is not reading tool descriptions — switch to a more capable model
- **B)** The system prompt may contain keyword-sensitive instructions that override tool descriptions (e.g., "When the user mentions an order, always verify their identity first")
- **C)** The tool descriptions need even more detail — add 500-word descriptions
- **D)** The tools need to be split into more fine-grained variants

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

After improving descriptions, the next suspect is the system prompt. Keyword-sensitive instructions like "always look up the customer first when the user mentions an order" create hard rules that override even excellent tool descriptions. Review the system prompt for conflicting instructions.

A is unlikely — model capability isn't the issue when descriptions are good. C is excessive. D is premature without checking for system prompt conflicts.

</details>

---

### Question 3 (Task 2.2 — Error Categories)

A tool attempts to process a refund of £450 but the company policy limits automated refunds to £200. Which error category is this, and what should the `isRetryable` flag be?

- **A)** Validation error, `isRetryable: true` — the input amount is invalid
- **B)** Business error, `isRetryable: false` — a policy limit was hit, and retrying with the same amount will fail identically
- **C)** Transient error, `isRetryable: true` — the refund service might accept it on a retry
- **D)** Permission error, `isRetryable: false` — the agent lacks permission to process large refunds

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

This is a **business error** — the policy says refunds above £200 cannot be automated. The £450 is a valid input (it's a real refund amount), so it's not a validation error. The service is working correctly, so it's not transient. The agent has permission to process refunds in general, so it's not a permission error — it's a business rule violation.

`isRetryable: false` because retrying with £450 will hit the same policy limit. The correct action is to escalate to a manager or offer an alternative workflow.

Trap: Option A is tempting because the amount "exceeds a limit" — but the limit is a business policy, not an input validation constraint. The format and type of the input are perfectly valid.

</details>

---

### Question 4 (Task 2.2 — Access Failure vs Empty Result)

A customer lookup tool returns an empty results array. The agent retries three times, gets the same empty result each time, and escalates to a human. Investigation reveals the customer account simply does not exist. What is the root cause of the wasted retries and unnecessary escalation?

- **A)** The retry logic should have a shorter timeout
- **B)** The tool response does not distinguish between a valid empty result (no account found) and an access failure (database unreachable), causing the agent to treat "no results" as a potential failure
- **C)** The agent should retry more than three times before escalating
- **D)** The human escalation path is misconfigured

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

The tool response is ambiguous — an empty array could mean "the database returned zero rows" (valid empty result, `is_error: false`) or "the database was unreachable" (access failure, `is_error: true`). Without explicit metadata (`queryExecuted: true`, `is_error: false`), the agent defensively treats ambiguity as failure.

The fix is structured error responses that distinguish the two cases. When the query executed successfully and returned zero results, the response must clearly communicate: "I worked correctly; the answer is zero results."

</details>

---

### Question 5 (Task 2.3 — Tool Distribution and tool_choice)

A document processing pipeline must always extract metadata (author, date, document type) as its first step before any analysis. What `tool_choice` configuration ensures this?

- **A)** `{"type": "auto"}` — Claude will naturally choose metadata extraction first
- **B)** `{"type": "any"}` — Claude must call a tool, and will pick the metadata extractor
- **C)** `{"type": "tool", "name": "extract_metadata"}` for the first API call, then switch to `{"type": "auto"}` for subsequent calls
- **D)** Remove all other tools except `extract_metadata` for the first call

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: C**

Forcing a named tool (`{"type": "tool", "name": "extract_metadata"}`) on the first call guarantees metadata extraction happens first. Switching to `auto` for subsequent calls allows Claude to choose the appropriate analysis tools based on the metadata.

A is risky — "naturally" is not "guaranteed." B forces a tool call but doesn't guarantee WHICH tool. D works but is over-engineered — removing and re-adding tools between calls is unnecessary when `tool_choice` exists for exactly this purpose.

</details>

---

### Question 6 (Task 2.4 — MCP Server Configuration)

A team wants to share Jira integration across all developers but allow each developer to use their own API token. Which configuration approach is correct?

- **A)** Store the Jira API token directly in `.mcp.json` and commit it to version control
- **B)** Configure the Jira server in `.mcp.json` with `${JIRA_API_TOKEN}` environment variable expansion, and have each developer set the token in their local environment
- **C)** Configure the Jira server in `~/.claude.json` on each developer's machine
- **D)** Create a shared service account token and embed it in the system prompt

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: B**

`.mcp.json` is project-level, version-controlled, and shared with the team — correct for a standard integration everyone needs. Environment variable expansion (`${JIRA_API_TOKEN}`) keeps credentials out of version control while allowing each developer to use their own token.

A commits secrets to version control — a security violation. C uses user-level config (`~/.claude.json`) which is NOT shared — each developer would have to configure it manually, and the configuration would not be version-controlled. D puts credentials in the system prompt — a severe security violation.

</details>

---

### Question 7 (Task 2.5 — Built-In Tools)

A developer needs to find all files in the codebase that import the deprecated `legacyAuth` module. Which approach is correct?

- **A)** Use Glob to search for `**/*legacyAuth*` — this finds all files related to the module
- **B)** Use Read to load every file in `src/` and search for import statements manually
- **C)** Use Grep to search for `import.*legacyAuth` or `require.*legacyAuth` across the codebase
- **D)** Use Glob to search for `**/auth/**` — all auth-related files likely import it

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: C**

Grep searches file **contents** — it finds files that contain `import legacyAuth` statements inside them. This directly answers the question.

A uses Glob, which searches file **paths** — it finds files *named* legacyAuth, not files that *import* it. A file named `userService.ts` that imports `legacyAuth` would be missed entirely.

B is a context-budget killer — loading every file is wasteful when Grep does the job in one call.

D is speculative — not all auth files import legacyAuth, and non-auth files might import it too.

</details>

---

## Scoring

| Score | Assessment | Next Step |
|-------|-----------|-----------|
| **7/7** | Exam-ready for Domain 2 | Move to Domain 3 |
| **6/7** | Nearly ready — review the one gap | Re-read the task statement for your missed question, redo the practice scenario |
| **5/7** | Needs targeted review | Re-read the 2 task statements where you missed questions, redo both practice scenarios |
| **Below 5** | Needs comprehensive review | Re-read the entire guide, focusing on exam trap boxes. Redo all practice scenarios before retaking the quiz. |

---

## Build Exercise

To solidify your understanding, complete this hands-on exercise:

### Exercise: MCP Tool Design, Error Handling, and Configuration

**Part 1 — Tool Design with Intentional Ambiguity**

Create three MCP tool definitions:
1. `search_knowledge_base` — searches internal documentation for answers
2. `search_support_tickets` — searches previous support tickets for similar issues
3. `search_help_articles` — searches public help centre articles

Make `search_knowledge_base` and `search_help_articles` intentionally ambiguous (near-identical descriptions). Then fix them with expanded descriptions that include specific use cases, input formats, and explicit boundaries.

**Part 2 — Structured Error Responses**

Write tool responses for all four error categories:
- Transient: knowledge base service timeout
- Validation: invalid ticket ID format
- Business: customer requesting access to enterprise-only content on a free plan
- Permission: agent lacks authorisation to view internal-only tickets

Each response must include: `errorCategory`, `isRetryable`, descriptive message, and (for business errors) a `customerFriendlyMessage`.

**Part 3 — MCP Configuration**

Create a `.mcp.json` file that:
- Configures a knowledge base MCP server with `${KB_API_KEY}` environment variable expansion
- Configures a ticketing MCP server with `${TICKET_SERVICE_URL}` and `${TICKET_API_TOKEN}`
- Is safe to commit to version control

**Part 4 — Forced Tool Selection**

Write a code snippet that uses `tool_choice` to force `search_knowledge_base` on the first API call, then switches to `auto` for subsequent calls. Explain when this pattern is useful (answer: when you always want to check internal docs before searching external sources).

---

## Quick Reference Card

| Concept | Key Fact |
|---------|----------|
| Tool descriptions | THE mechanism for tool selection — not supplementary |
| Misrouting fix | Better descriptions first, before classifiers or consolidation |
| Tool splitting | Split generic tools into purpose-specific tools with defined contracts |
| System prompt conflict | Keyword-sensitive instructions can override good descriptions |
| isError flag | Must be set on errors so Claude knows a tool call failed |
| Four error categories | Transient (retry), Validation (fix input), Business (escalate), Permission (escalate) |
| Access failure vs empty result | Access failure → maybe retry. Empty result → do NOT retry. |
| Error propagation | Local recovery first, propagate only unresolvable errors, include partial results |
| Tool overload | 4–5 tools per agent, scoped to role |
| tool_choice auto | Model decides — default for general operation |
| tool_choice any | Must call a tool, model picks which — for guaranteed structured output |
| tool_choice named | Must call specific tool — for forced first steps |
| Scoped cross-role tools | Give constrained tool to avoid coordinator round-trips for simple cases |
| .mcp.json | Project-level, version-controlled, shared with team |
| ~/.claude.json | User-level, NOT version-controlled, NOT shared |
| ${VAR} expansion | Keeps credentials out of version control |
| MCP resources | Content catalogues that reduce exploratory tool calls |
| Build vs use | Community servers first, custom only for team-specific gaps |
| Grep | Searches file CONTENTS — find callers, imports, error messages |
| Glob | Searches file PATHS — find files by extension or naming pattern |
| Edit | Targeted modification via unique text matching — first choice |
| Read + Write | Reliable fallback when Edit fails on non-unique text |
| Incremental discovery | Grep → Read → follow imports → Grep again. Never load everything upfront. |

