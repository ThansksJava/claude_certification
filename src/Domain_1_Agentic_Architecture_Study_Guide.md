# Domain 1: Agentic Architecture & Orchestration — Complete Study Guide

**Claude Certified Architect (Foundations) | Exam Weight: 27% (Highest Single Domain)**

> 🇬🇧 This guide uses British English spelling throughout (e.g., "normalise", "optimise", "favour").

---

## How to Use This Guide

This guide is structured as a teach-then-check walkthrough. For each of the 7 task statements you will find:
1. **Concept explanation** with a concrete production example
2. **Exam traps** — specific anti-patterns and misconceptions that appear as distractors
3. **Practice scenario** with a worked answer
4. **Connection** to the next task statement

After all 7 task statements, a **10-question practice exam** covers the full domain. Score yourself, map gaps to task statements, and revisit weak areas.

---

## Self-Assessment: Where Do You Start?

Rate your familiarity before diving in:

| Rating | Description | How to Use This Guide |
|--------|-------------|----------------------|
| **Novice** | No experience with agentic or LLM systems | Read every section word for word. Do every practice scenario before reading the answer. |
| **Intermediate** | Built a simple single-agent system | Focus on the Exam Traps boxes and practice scenarios. Skim concept explanations. |
| **Advanced** | Built multi-agent systems in production | Jump to practice scenarios and the final exam. Use task statement explanations to fill gaps. |

---

## Domain Overview

- **Exam weight:** 27% — the single highest-weighted domain
- **Passing score:** 720 / 1000
- **Format:** Scenario-based multiple choice. One correct answer. Three plausible distractors.
- **Primary exam scenarios:** Customer Support Resolution Agent · Multi-Agent Research System · Developer Productivity Tools

> **📌 Core Exam Philosophy**
>
> The exam consistently rewards three things:
> 1. **Deterministic solutions over probabilistic ones** when stakes are high (financial, security, compliance)
> 2. **Proportionate fixes** — do not use a sledgehammer on a formatting problem
> 3. **Root cause tracing** — failures are attributed to the component where they originate, not where they manifest

---

## Task Statement 1.1: Agentic Loops

### The Complete Agentic Loop Lifecycle

An agentic loop is the core execution pattern for any agent built on Claude. Here is what it looks like in a production customer support agent resolving a billing dispute:

```
1. Send the customer's message to Claude via the Messages API
2. Claude responds — inspect the stop_reason field in the response
3. If stop_reason == "tool_use":
      → Execute the tool(s) Claude requested
      → Append the tool results to conversation history as a new message
      → Send the updated conversation back to Claude
      → Go to step 2
4. If stop_reason == "end_turn":
      → Claude has finished reasoning
      → Present the final response to the customer
```

The key insight: **tool results must be appended to conversation history** so that Claude can reason about new information on the next iteration. The model has no memory between API calls — the conversation history is the only thing it knows.

Here is a correct Python implementation:

```python
import anthropic

client = anthropic.Anthropic()

def run_agentic_loop(initial_messages, tools, system_prompt):
    messages = initial_messages.copy()

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )

        # CORRECT: Inspect stop_reason — not content type
        if response.stop_reason == "end_turn":
            final_text = next(
                (block.text for block in response.content
                 if hasattr(block, "text")), ""
            )
            return final_text

        elif response.stop_reason == "tool_use":
            # Step 1: Append assistant response (includes tool_use blocks)
            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # Step 2: Execute each tool and collect results
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # Step 3: Append tool results — loop continues
            messages.append({
                "role": "user",
                "content": tool_results
            })
            # Back to top of while loop → sends updated history to Claude
```

---

### ⚠️ The Three Anti-Patterns (Exam Traps)

| Anti-Pattern | Why It's Wrong | What To Do Instead |
|---|---|---|
| **Parsing natural language signals** — checking if the assistant said "I'm done" or "Task complete" | Natural language is ambiguous and unreliable. The model might say "I'm done with the first part" mid-task. The `stop_reason` field exists for exactly this purpose. | Use `stop_reason == "end_turn"` |
| **Arbitrary iteration caps as the primary stopping mechanism** — "stop after 10 loops regardless" | Cuts off useful work mid-task, or runs unnecessary empty iterations. The model signals completion via `stop_reason`. | Use `stop_reason` as the primary signal. A cap is a safety net only, not the primary logic. |
| **Checking content type as a completion indicator** — `if response.content[0].type == "text": done` | Claude can return a text block (reasoning narration) alongside a `tool_use` block in the same response. The check fires on the text and terminates before the tool executes. | Check `stop_reason`, never content type. |

---

### Model-Driven vs Pre-Configured Approaches

- **Model-driven:** Claude reasons about which tool to call based on context. Flexible, adapts to novel situations.
- **Pre-configured:** Fixed decision tree or predetermined tool sequence. Predictable, auditable.

**Exam rule:** Favour model-driven for general flexibility. Use programmatic enforcement (pre-configured gates) for critical business logic — covered in Task 1.4.

---

### 🧪 Practice Scenario 1.1 — The Premature Termination Bug

**Scenario:** A developer builds a customer support agent. During testing, the agent sometimes terminates before completing multi-step tasks. Here is their termination logic:

```python
response = client.messages.create(...)

# Developer's termination check
if response.content and response.content[0].type == "text":
    return response.content[0].text  # ← something is wrong here
```

**Questions:**
1. What is the bug?
2. What is the correct fix?

<details>
<summary>▶ Show Answer</summary>

**Bug:** Claude sometimes returns a text block (a reasoning narration, e.g., "Let me look up the customer's account…") alongside a `tool_use` block in the same response. The check fires on the text block and returns immediately, never processing the `tool_use` block. The tool never executes.

**Fix:** Replace the content-type check with `stop_reason`:

```python
if response.stop_reason == "end_turn":
    # Safely extract text — this is the genuine final response
    return next((b.text for b in response.content if hasattr(b, "text")), "")

elif response.stop_reason == "tool_use":
    # Process tool calls — loop must continue
    ...
```

</details>

---

*This connects to Task 1.2: once your loop is correct, you need to orchestrate multiple agents through it.*

---

## Task Statement 1.2: Multi-Agent Orchestration

### The Hub-and-Spoke Architecture

```
                    ┌─────────────────┐
                    │   COORDINATOR   │
                    │     AGENT       │
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  Subagent A │  │  Subagent B │  │  Subagent C │
    │ (Web Search)│  │  (Doc Anal.)│  │ (Synthesis) │
    └─────────────┘  └─────────────┘  └─────────────┘

    ✅ ALL communication flows through the coordinator
    ❌ Subagents NEVER communicate directly with each other
```

### Coordinator Responsibilities

1. **Task decomposition** — break the user's request into well-scoped subtasks
2. **Dynamic subagent selection** — decide which subagents to invoke (not always all of them)
3. **Scope partitioning** — assign distinct subtopics to minimise duplication between agents
4. **Context passing** — explicitly include all information each subagent needs in its prompt
5. **Result aggregation** — combine subagent outputs into a coherent response
6. **Error handling** — detect subagent failures and recover or escalate
7. **Iterative refinement** — evaluate output for gaps, re-delegate targeted queries, repeat until quality threshold is met

---

### 🔴 The Critical Isolation Principle

> **⚠️ EXAM CRITICAL — Most Commonly Misunderstood Concept**
>
> - Subagents do **NOT** automatically inherit the coordinator's conversation history
> - Subagents do **NOT** share memory between invocations
> - Subagents do **NOT** have access to other subagents' results unless explicitly passed
>
> **Every piece of information a subagent needs must be explicitly included in its prompt by the coordinator.**
>
> If Subagent A discovers finding X, and Subagent B needs to know about X, the coordinator must explicitly pass X in Subagent B's prompt.

---

### The Narrow Decomposition Failure

This failure pattern is directly tested in the exam (Q7 in the sample set).

**What it looks like:** A coordinator decomposes "impact of AI on creative industries" into only visual arts subtopics: (1) AI in painting, (2) AI in photography, (3) AI in illustration. The final report completely misses music, writing, film, and theatre.

**Root cause:** The coordinator's decomposition logic — NOT the subagents. The subagents did their assigned work correctly. The failure originates at decomposition.

**Exam trap:** Answer options will blame the subagents for not researching missing topics, or blame the synthesis agent for not flagging gaps. Reject both. **Always trace the failure to its origin component.**

---

### 🧪 Practice Scenario 1.2 — The Incomplete Research Report

**Scenario:** A multi-agent research system receives: *"Analyse the impact of renewable energy technologies on global electricity markets."* The final report comprehensively covers solar and wind power but contains no analysis of geothermal, tidal, biomass, or nuclear fusion.

System components:
- **Coordinator Agent:** decomposes query and delegates to subagents
- **Research Subagent 1:** web search specialist
- **Research Subagent 2:** academic database specialist
- **Synthesis Subagent:** combines findings into final report

Which component is the root cause?

- **A)** Research Subagent 1 — it only searched for solar and wind sources
- **B)** Research Subagent 2 — it only queried solar and wind academic papers
- **C)** The Coordinator Agent — its task decomposition only assigned solar and wind subtopics
- **D)** The Synthesis Subagent — it failed to identify the missing technology categories

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: C**

The subagents performed their assigned work correctly — they researched what they were assigned. The synthesis agent cannot create content it was never given. The root cause is the coordinator's decomposition, which failed to cover the full scope of renewable energy technologies. Trace failures to their origin.

</details>

---

*This connects to Task 1.3: once you understand the architecture, you need to know HOW to invoke subagents and pass context correctly.*

---

## Task Statement 1.3: Subagent Invocation and Context Passing

### The Task Tool

- The mechanism for spawning subagents from a coordinator
- The coordinator's `allowedTools` **must include `"Task"`** — without it, the coordinator cannot spawn subagents at all
- **Exam trap:** If a multi-agent system isn't spawning subagents, check `allowedTools` before anything else
- Each subagent is defined via an `AgentDefinition`: description, system prompt, and tool restrictions

---

### Context Passing — Right vs Wrong

**❌ Poor context passing:**

```python
subagent_prompt = """
Synthesise the research findings into a comprehensive report
on renewable energy technologies.
"""
# Problem: subagent has no actual findings to synthesise.
# It will either hallucinate or produce a generic response.
```

**✅ Correct context passing:**

```python
import json

subagent_prompt = f"""
Synthesise the following research findings into a comprehensive report.

## WEB SEARCH FINDINGS
{web_search_results}

## ACADEMIC DOCUMENT FINDINGS
{document_analysis_results}

## SOURCE METADATA
{json.dumps(source_metadata, indent=2)}
# source_metadata format:
# [{"claim": "...", "source_url": "...", "document": "...", "page": 4}, ...]

## Research Goal
{original_research_goal}

## Quality Criteria
- Cover all major technology categories found in the research
- Cite specific sources for each factual claim using the source metadata above
- Flag any areas where source coverage appears thin
"""
```

**Key principles:**
1. Include complete findings from prior agents directly in the prompt
2. Use structured data formats that separate content from metadata (source URLs, document names, page numbers) — this preserves attribution
3. Specify **goals and quality criteria**, NOT step-by-step procedural instructions — this enables subagent adaptability
4. Pass **claim-source mappings** explicitly so the synthesis agent can produce citations

---

### Parallel Spawning

By emitting multiple `Task` tool calls in a **single coordinator response**, subagents are spawned in parallel:

```python
# Coordinator emits ALL of these in ONE response → parallel execution
[
    Task(agent="web_search_agent",  prompt="Search for solar energy market data for 2020–2025..."),
    Task(agent="web_search_agent",  prompt="Search for wind energy capacity and investment trends..."),
    Task(agent="academic_agent",    prompt="Find peer-reviewed papers on geothermal economics..."),
    Task(agent="academic_agent",    prompt="Find research on tidal and wave energy commercialisation..."),
]

# Sequential alternative: each Task spawned in a separate coordinator turn
# Parallel is ~4x faster for these independent tasks
```

**Exam rule:** When latency is a concern and subtasks are independent, parallelise by spawning in a single response.

---

### fork_session

- Creates **independent branches** from a shared analysis baseline
- Use for exploring divergent approaches simultaneously from the same starting point
- **Example:** After analysing a codebase, fork into two branches — one evaluates testing strategy A, the other evaluates testing strategy B. Each fork operates independently after the branching point.
- Different from parallel subagent spawning: forks share a complete baseline context; parallel subagents are given only the context the coordinator passes them.

---

### 🧪 Practice Scenario 1.3 — The Uncited Report

**Scenario:** A research system produces a detailed, well-written report on climate change policy. The report contains many specific factual claims but no source citations. Investigation reveals:
- The web search subagent is working correctly and returning results with full source URLs
- The document analysis subagent is working correctly and returning content with document names and page numbers
- The synthesis subagent is producing fluent, well-structured prose

What is the root cause, and what is the fix?

<details>
<summary>▶ Show Answer</summary>

**Root cause:** The coordinator's context passing did not include structured source metadata when passing findings to the synthesis subagent. The synthesis agent received content but not the associated attribution information, so it could not produce citations even if it wanted to.

**Fix:** Require research subagents to output structured claim-source mappings:

```json
[
  {
    "claim": "Solar capacity increased 45% in the EU between 2020 and 2024",
    "source_url": "https://iea.org/reports/solar-2024",
    "document": null,
    "page": null
  },
  {
    "claim": "The Paris Agreement mandates net-zero by 2050 for signatories",
    "source_url": null,
    "document": "UNFCCC_Paris_Agreement.pdf",
    "page": 12
  }
]
```

The coordinator passes this structured data to the synthesis subagent, which uses it to produce inline citations.

</details>

---

*This connects to Task 1.4: once subagents are running correctly, you need to enforce that critical workflow steps always happen.*

---

## Task Statement 1.4: Workflow Enforcement and Handoff

### The Enforcement Spectrum

| Enforcement Type | Mechanism | Reliability | Use When |
|---|---|---|---|
| **Prompt-based guidance** | System prompt instructions ("always verify the customer first") | ~92–99% (probabilistic) | Low-stakes preferences, style, formatting |
| **Programmatic enforcement** | Prerequisite gates or hooks that physically block downstream tools | 100% (deterministic) | Financial, security, compliance operations |

---

### The Exam's Decision Rule

> **🔴 EXAM RULE — High-Stakes = Programmatic, Always**
>
> When a **single failure** causes financial loss, a security breach, or a compliance violation: **use programmatic enforcement**.
>
> When a single failure is merely suboptimal (wrong tone, formatting preference): prompt-based guidance is acceptable.
>
> The exam will offer "enhanced system prompt" and "few-shot examples" as answer options for high-stakes scenarios. **Reject both.** They are probabilistic — they reduce the failure rate but cannot eliminate it.

---

### 🧪 Practice Scenario 1.4 — The 8% Refund Problem

*(This is Q1 in the sample exam set)*

**Scenario:** Production monitoring shows that in 8% of customer support interactions, the agent processes refunds without verifying account ownership. Several refunds have been issued to the wrong account. The engineering team proposes four solutions:

- **A)** Implement a programmatic prerequisite gate: the refund tool cannot execute unless a verified account token is present in session state
- **B)** Rewrite the system prompt with stronger instructions: "YOU MUST ALWAYS verify account ownership before processing any refund"
- **C)** Add 10 few-shot examples demonstrating the correct verify-then-refund sequence
- **D)** Add a routing classifier that sends refund requests to a dedicated refund sub-pipeline

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: A**

| Option | Why |
|---|---|
| **A ✅** | The gate is deterministic. The refund tool literally cannot execute without a verified token. The 8% failure rate drops to 0%. |
| **B ❌** | A system prompt instruction is already presumably in place. Stronger language is still probabilistic. Financial security requires 0% failure. |
| **C ❌** | Few-shot examples improve consistency but remain probabilistic. The failure rate decreases but doesn't reach 0%. |
| **D ❌** | A routing classifier moves the request to a different pipeline. It does nothing to enforce verification *within* that pipeline. Same failure can occur. |

</details>

---

### Multi-Concern Request Handling

When a customer request has multiple issues (e.g., billing dispute + service interruption + account access problem):
1. Decompose into distinct investigation items
2. Investigate each in parallel using shared context (efficient, not sequential)
3. Synthesise a unified resolution

---

### Structured Handoff Protocols

When escalating to a human agent, compile a **self-contained handoff summary**:

```
ESCALATION HANDOFF SUMMARY
===========================
Customer ID:          [ID]
Account verified:     [Yes / No]
Conversation summary: [2–3 sentences describing what happened]
Root cause analysis:  [What went wrong and why]
Refund amount:        [£X.XX, if applicable]
Actions already taken:[List of completed steps]
Recommended action:   [Specific, actionable recommendation]
Escalation reason:    [Why human intervention is required]
```

> **⚠️ Critical:** The human agent does **NOT** have access to the conversation transcript. The handoff summary must be entirely self-contained. Every relevant fact must be in the summary itself.

---

*This connects to Task 1.5: hooks are the primary mechanism for implementing programmatic enforcement.*

---

## Task Statement 1.5: Agent SDK Hooks

### PostToolUse Hooks (After Execution, Before Model Sees Result)

```
Tool executes → [PostToolUse Hook intercepts] → Model receives normalised result
```

**Primary use case:** Normalise heterogeneous data formats from different MCP tools so the model always receives clean, consistent data.

```python
from datetime import datetime

def post_tool_use_hook(tool_name: str, tool_result: dict) -> dict:
    """Normalise tool outputs before the model processes them."""

    if tool_name == "database_query":
        # Normalise Unix timestamps → ISO 8601
        if "created_at" in tool_result:
            tool_result["created_at"] = datetime.fromtimestamp(
                tool_result["created_at"]
            ).isoformat()

        # Normalise numeric status codes → human-readable strings
        status_map = {1: "active", 2: "suspended", 3: "pending"}
        if "status" in tool_result:
            tool_result["status"] = status_map.get(
                tool_result["status"], "unknown"
            )

    return tool_result
    # Model receives clean data — it never sees the raw output
```

---

### Tool Call Interception Hooks (Before Execution)

```
Model requests tool call → [Pre-Execution Hook intercepts] → Execute / Block / Redirect
```

**Use cases:** Block high-value transactions; enforce compliance rules; require approvals.

```python
def pre_tool_use_hook(tool_name: str, tool_input: dict):
    """Intercept outgoing tool calls before they execute."""

    if tool_name == "process_refund":
        amount = tool_input.get("amount", 0)
        if amount > 500:
            # Block the refund entirely — redirect to human escalation
            trigger_human_escalation(
                reason=f"Refund of £{amount} exceeds automated limit of £500",
                details=tool_input
            )
            return None  # None = block tool execution

    if tool_name == "international_transfer":
        if not compliance_check_passed(tool_input):
            raise ComplianceError(
                "International transfer blocked: compliance check not completed"
            )

    return tool_input  # Proceed normally
```

---

### Hooks vs Prompts — The Decision Framework

| Scenario | Use Hook? | Reason |
|---|---|---|
| Block refunds above £500 | ✅ Hook | Financial risk from a single failure |
| International transfer compliance check | ✅ Hook | Legal / regulatory risk |
| Normalise inconsistent API response formats | ✅ Hook | Correctness requirement |
| Prefer formal tone in customer responses | ❌ Prompt | Style preference, low stakes |
| Summarise findings before responding | ❌ Prompt | Soft workflow preference |
| Security scan before code refactoring | ✅ Hook | Security risk from a single failure |

**Rule:** If the business would lose money or face legal risk from a single failure, use a hook.

---

### 🧪 Practice Scenario 1.5 — The Compliance Gap

**Scenario:** An agent that processes international wire transfers has occasionally completed transfers without running the required SWIFT compliance check. This has happened three times in the past month, resulting in regulatory penalties. The compliance team proposes adding more detailed compliance instructions to the system prompt.

Should you use a hook or enhanced prompt instructions? Why?

<details>
<summary>▶ Show Answer</summary>

**Use a pre-execution hook.**

This is a legal/regulatory risk scenario. Regulatory penalties make any non-zero failure rate unacceptable. A hook that physically blocks the `international_transfer` tool from executing unless a passed compliance check result is present provides 100% enforcement.

The three historical failures demonstrate that the current prompt-based guidance is already insufficient. Stronger prompt instructions would reduce the frequency of failures but remain probabilistic. On the exam: when the question tells you prompt-based guidance has already failed in production, it is confirming that programmatic enforcement is required.

</details>

---

*This connects to Task 1.6: hooks enforce rules within a pipeline — decomposition strategies determine how the pipeline itself is structured.*

---

## Task Statement 1.6: Task Decomposition Strategies

### The Two Main Patterns the Exam Tests

#### Pattern 1: Fixed Sequential Pipelines (Prompt Chaining)

```
Input → [Step 1: Analyse] → [Step 2: Cross-check] → [Step 3: Report] → Output
              ↓                      ↓                       ↓
       Step 1 output         Step 2 output           Final synthesis
       becomes context       becomes context         with full traceability
       for Step 2            for Step 3
```

**Definition:** Break the work into predetermined sequential steps where each step depends on the previous one.

**Production example — code review:**
1. Analyse each file individually for local defects
2. Run a cross-file integration pass for dependency and data-flow issues
3. Run a security pass using the integrated view
4. Produce a final review summary with prioritised fixes

**Best for:** predictable, structured tasks such as code reviews, document processing, or compliance workflows.

**Advantage:** consistent and reliable. You know exactly what happens at each stage, which makes behaviour easier to audit.

**Limitation:** cannot adapt well to unexpected findings. If step 1 discovers an entirely new area of concern, the pipeline does not naturally redesign itself unless you explicitly code that path.

---

#### Pattern 2: Dynamic Adaptive Decomposition

```
Input → [Initial mapping] → [Discover structure] → [Generate subtasks based on findings]
                                                     │
                                                     ├── New dependency found → add targeted subtask
                                                     ├── High-risk area found → deepen analysis there
                                                     └── Low-value area found → reduce effort there
```

**Definition:** Generate subtasks based on what is discovered at each step, rather than committing to the full plan upfront.

**Production example — adding tests to a legacy codebase:**
1. Map the repository structure
2. Identify entry points, brittle modules, and missing coverage hotspots
3. Prioritise high-impact areas for test creation
4. Adapt the plan as hidden dependencies and risky code paths emerge

**Best for:** open-ended investigation tasks where the shape of the problem is not fully known at the start.

**Advantage:** adapts to the problem. It lets the system deepen analysis where new evidence says it matters.

**Limitation:** less predictable. Outputs and runtime vary because the plan evolves during execution.

---

### Supporting Patterns You Combine with the Main Two

#### Parallel fan-out / fan-in

Use when subtasks are independent and latency matters.

```
                    ┌──────────────────┐
         ┌──────────│   Coordinator    │──────────┐
         ▼          └──────────────────┘          ▼
   [Subagent A]     [Subagent B]     [Subagent C]  [Subagent D]
         └──────────────────┬──────────────────────┘
                            ▼
                      [Aggregator]
                            │
                            ▼
                       Final Output
```

**Exam point:** If tasks do not depend on each other, parallelise them. The exam rewards latency-aware design.

#### Iterative refinement

Use when coverage and quality matter more than finishing in a single pass.

```
Initial output → [Evaluator: sufficient?]
                       │
              ┌────────┴────────┐
             YES                NO
              │                 │
            Done     Re-delegate targeted gaps and re-evaluate
```

**Exam point:** This is how a coordinator closes gaps without re-running the entire workflow blindly.

---

### The Attention Dilution Problem

> **Exam trap:** asking one agent to review too many files in a single pass produces inconsistent depth.

What it looks like in production:
- detailed, high-quality feedback on the first few files
- superficial feedback on later files
- obvious defects missed in some files
- inconsistent judgements where identical code is criticised in one file and approved in another

**Why it happens:** the model's attention is spread across too much local detail at once. It cannot maintain the same scrutiny level over every file and every cross-file interaction simultaneously.

**Correct fix:** split the review into multiple passes:
1. **Per-file local analysis passes** — each file gets focused, consistent scrutiny
2. **Cross-file integration pass** — a separate pass checks data flow, dependency interactions, duplicated logic, and system-wide patterns

This is the architecture the exam prefers for medium-to-large code reviews.

---

### Decomposition Strategy — Decision Table

| Scenario | Best Strategy | Reason |
|---|---|---|
| Code review with clear dependencies between stages | Fixed sequential pipeline | Predictable steps; later stages depend on earlier outputs |
| Adding tests to a legacy codebase with unknown structure | Dynamic adaptive decomposition | The plan must adapt as dependencies and hotspots are discovered |
| Broad research with independent subtopics | Parallel fan-out | Independent tasks; latency matters |
| Coverage-critical report | Iterative refinement | Coordinator keeps filling gaps until criteria are met |
| Comparing two approaches from the same baseline | `fork_session` | Shared starting point, divergent follow-up analysis |

---

### ⚠️ Exam Traps for 1.6

1. **Single-pass review across too many files** — sounds efficient, performs inconsistently.
2. **Parallelising dependent steps** — wrong when later analysis needs earlier outputs.
3. **Using a fixed pipeline for exploratory work** — too rigid when the problem evolves during investigation.
4. **Using dynamic adaptive decomposition for simple deterministic workflows** — over-complicates predictable tasks.

---

### 🧪 Practice Scenario 1.6 — The Inconsistent 14-File Review

**Scenario:** A developer productivity agent reviews 14 files in one pass. It produces detailed feedback for some files but misses obvious bugs in others. It also flags a pattern as problematic in one file while approving nearly identical code elsewhere.

**Question:** What is the underlying problem, and what is the correct architectural fix?

<details>
<summary>▶ Show Answer</summary>

**Problem:** attention dilution caused by trying to do a large multi-file review in a single pass.

**Fix:** move to a multi-pass architecture:
- first, run focused per-file local analysis passes
- then run a separate cross-file integration pass for system-level issues and consistency checks

This gives each file consistent depth while still catching cross-file issues in a later stage.

</details>

---

*This connects to Task 1.7: once you know how to decompose work, you need to manage session state across long or interrupted workflows.*

---

## Task Statement 1.7: Session State and Resumption

### The Three Session Management Options

When a long agentic task is interrupted, paused, or needs to branch into alternative approaches, you have three distinct options. Choosing the wrong one is a common source of bugs in production.

| Option | Mechanism | What it Does |
|---|---|---|
| `--resume <session-name>` | Resume a named session | Continues the exact prior conversation — all prior tool results, reasoning, and context are restored |
| `fork_session` | Fork from a baseline | Creates an independent branch from a shared analysis point — both branches run forward independently |
| **Fresh start with summary injection** | New session + structured summary | Starts a clean session but injects a condensed, structured summary of prior findings as initial context |

---

### When to Use Each Option

#### `--resume <session-name>` — Continue Where You Left Off

**Use when:**
- The prior context is still valid and accurate
- Files and data sources have **not** changed since the session was suspended
- The task was interrupted mid-execution (e.g., network failure, user pause) and simply needs to continue
- The session is relatively short and recently created

**Avoid when:**
- Code files have been modified since the last tool result was captured — the agent will reason from stale data
- The session is old and context may have drifted from the current state of the codebase
- Prior tool results reflect a system state that no longer exists

**Production example:** A research agent is halfway through synthesising a 20-source report when the user closes their laptop. They resume the next morning. The sources have not changed. `--resume` is correct.

---

#### `fork_session` — Explore Divergent Approaches from a Shared Baseline

**Use when:**
- You have a shared analysis that both branches need, but the branches need to diverge afterwards
- You want to compare two approaches, architectures, or solutions simultaneously without cross-contaminating context
- You want independent evaluations of the same artefact (e.g., two reviewers assessing the same codebase independently)

**Avoid when:**
- The branches are not truly independent — if one branch's findings should influence the other, forking is wrong
- You simply want to continue a task (use `--resume`)
- You need a clean slate (use fresh start with summary injection)

**Production example:** After analysing a legacy codebase, you want to evaluate two test strategies simultaneously. Fork the session at the point where the analysis is complete. Branch A explores strategy 1; Branch B explores strategy 2. Neither branch sees the other's reasoning.

```
Shared baseline analysis
        │
        ├──── Fork A: Testing Strategy 1 (unit-first)
        │              (independent branch)
        │
        └──── Fork B: Testing Strategy 2 (integration-first)
                       (independent branch)
```

---

#### Fresh Start with Summary Injection — Clean Slate, Preserved Knowledge

**Use when:**
- Tool results are **stale** — the underlying files, data, or APIs have changed
- Context has **degraded** over a very long session (many thousands of tokens of history leading to confused reasoning)
- You want to compact a complex prior investigation into a clean, structured starting point
- Files have been **modified** since the prior session, and re-analysis is needed on a fresh footing

**How to do it correctly:**

```python
# Step 1: Distil prior session into a structured summary
prior_summary = """
## Prior Analysis Summary (as of 2026-03-26)

### Repository Structure
- Entry points: app/main.py, workers/processor.py
- Total files analysed: 14
- Language: Python 3.11

### Key Findings
1. Authentication module (auth/jwt_handler.py) contains a timing-side-channel vulnerability
2. Database connection pool (db/pool.py) is not thread-safe under high concurrency
3. Three files modified since last session: auth/jwt_handler.py, db/pool.py, tests/test_auth.py

### Outstanding Work
- Security fix for jwt_handler.py not yet reviewed
- No test coverage for db/pool.py concurrency behaviour
- tests/test_auth.py has been rewritten — prior analysis is now invalid

### Tools Run in Prior Session
- Static analysis: complete (except modified files)
- Dependency audit: complete, no outstanding CVEs
- Integration review: NOT YET STARTED
"""

# Step 2: Inject the summary as the first message in a brand-new session
messages = [
    {
        "role": "user",
        "content": f"""
You are continuing a security and quality review of a Python codebase.
Here is a structured summary of prior findings to bootstrap your context:

{prior_summary}

Three files have changed since the prior analysis. Please begin by re-analysing
ONLY those three files, then continue with the outstanding integration review.
"""
    }
]
```

**Key principle:** the injected summary replaces stale conversation history with accurate, distilled facts. The agent starts fresh but is not starting blind.

---

### The Stale Context Problem — What Goes Wrong and Why

This is the most commonly tested failure mode for Task 1.7.

#### What stale context looks like in production:

1. Developer runs a code review agent on a codebase. Agent produces a detailed report.
2. Developer fixes several bugs, modifying 3 files.
3. Developer resumes the same session and asks follow-up questions.
4. The agent gives contradictory advice — it "knows" from its tool results that `db/pool.py` has a certain bug, but the developer already fixed it. The agent recommends a fix that conflicts with the new code.

**Root cause:** the agent is reasoning from **cached tool results** that reflect the old state of the files, not the current state.

#### The Exam's Decision Rule for Stale Context

> **🔴 EXAM RULE — Stale Tool Results = Fresh Start with Summary Injection**
>
> When a session is resumed after files have been modified:
> - Do **NOT** use `--resume` as-is (stale tool results will corrupt reasoning)
> - Do **NOT** ask the agent to re-explore everything from scratch (wastes time; loses valid prior findings)
> - **DO** start fresh with a structured summary that marks which findings are still valid and which files need re-analysis

---

### Targeted Re-Analysis — Informing the Agent About Specific Changes

When resuming or starting fresh after code modifications, tell the agent **specifically which files changed** rather than asking it to re-examine everything:

**❌ Inefficient (re-explores everything):**
```
"The codebase may have changed. Please re-analyse everything from the beginning."
```

**✅ Correct (targeted re-analysis):**
```
"Three files have changed since the prior analysis:
  1. auth/jwt_handler.py — timing vulnerability fix applied
  2. db/pool.py — thread safety patch applied
  3. tests/test_auth.py — fully rewritten with new test cases

All other prior findings remain valid. Please re-analyse only these three files,
then continue with the cross-file integration review that was not yet started."
```

This preserves valid prior findings and directs the agent's effort precisely where it is needed.

---

### Session Management — Decision Table

| Scenario | Correct Option | Why |
|---|---|---|
| Task interrupted mid-execution, no file changes | `--resume` | Prior context is still valid; just continue |
| Evaluate two architectural approaches from a shared baseline | `fork_session` | Independent branches from a shared starting point |
| Files modified since prior session; prior findings partially valid | Fresh start + summary injection | Stale tool results; preserve valid findings, discard stale ones |
| Session context has degraded after thousands of tokens | Fresh start + summary injection | Clean slate prevents confused reasoning |
| Two independent reviewers assessing the same artefact | `fork_session` | Shared baseline, independent divergent analysis |
| Long session paused overnight, no system changes | `--resume` | Context is fresh enough; no staleness issue |

---

### ⚠️ Exam Traps for 1.7

1. **Using `--resume` after file modifications** — the most common trap. Stale tool results lead to contradictory advice.
2. **Asking the agent to re-explore everything after a partial change** — wastes time; loses valid prior analysis.
3. **Confusing `fork_session` with `--resume`** — forks create independent branches; resume continues a single thread.
4. **Fresh start without a summary** — discards all valid prior findings; the agent starts blind.
5. **Not specifying which files changed** — causes the agent to either re-examine everything unnecessarily or miss the changed files.

---

### 🧪 Practice Scenario 1.7 — The Contradictory Advice Problem

**Scenario:** A developer runs a security review agent on a 12-file Python codebase. The agent produces a detailed report identifying three vulnerabilities. The developer fixes two of them, modifying `auth/jwt_handler.py` and `db/pool.py`. The developer then resumes the same session and asks: *"Are the two vulnerabilities I fixed now resolved?"*

The agent responds with contradictory advice: it simultaneously says "the timing vulnerability in `jwt_handler.py` appears to be fixed" and "you should apply patch X to `jwt_handler.py`" — where patch X is the exact fix the developer just applied.

Which of the following is the correct approach going forward?

- **A)** Resume the session and explicitly tell the agent: *"Ignore your prior tool results for the two modified files and re-read them now"*
- **B)** Start a fresh session with no context — the agent should discover everything from scratch
- **C)** Fork the session and have one branch verify each fix independently
- **D)** Start a fresh session, injecting a structured summary of the prior valid findings and specifying the two modified files for targeted re-analysis

<details>
<summary>▶ Show Answer</summary>

**Correct Answer: D**

| Option | Why |
|---|---|
| **A ❌** | Attempting to selectively override prior tool results within a resumed session is unreliable. The stale results remain in context and will continue to interfere with reasoning. |
| **B ❌** | Discards all valid prior findings (the third vulnerability still exists and was correctly identified). The agent would re-discover it, wasting time and potentially missing it in a fresh pass. |
| **C ❌** | `fork_session` is for exploring divergent approaches from a shared baseline. It does not address the stale context problem — both forks would inherit the same stale tool results. |
| **D ✅** | Fresh start eliminates the stale tool results entirely. The structured summary preserves the valid finding (the third vulnerability) and the knowledge about the codebase structure. Specifying the two modified files directs re-analysis precisely where it is needed. |

</details>

---

*You have now completed all 7 task statements. The section below is a full 10-question practice exam covering the complete Domain 1.*

---

## Domain 1 Practice Exam — 10 Questions

**Instructions:** Select the single best answer for each question. Time yourself: aim for 90 seconds per question. After completing all 10, check your score using the answer key. For any incorrect answer, return to the corresponding task statement and re-read the exam trap that matches your wrong answer.

---

### Question 1 — Agentic Loop Termination *(Task 1.1)*

A customer support agent's agentic loop occasionally exits before completing multi-step tasks. Inspection shows this termination check:

```python
for block in response.content:
    if block.type == "text":
        return block.text
```

What is the bug?

- **A)** The loop should check `response.stop_reason == "end_turn"` instead of inspecting content block types
- **B)** The loop should check for `block.type == "tool_result"` rather than `"text"`
- **C)** The loop should append tool results before checking for text blocks
- **D)** The loop should use `response.content[-1]` to check the last block rather than iterating all blocks

---

### Question 2 — Multi-Agent Root Cause Attribution *(Task 1.2)*

A multi-agent research system produces a comprehensive report on "the impact of AI on the global labour market." The report covers white-collar and knowledge work in detail but contains no analysis of blue-collar, manufacturing, agricultural, or gig economy labour.

The system has: a Coordinator Agent, a Web Search Subagent, an Academic Research Subagent, and a Synthesis Subagent.

Which component is the root cause of the incomplete coverage?

- **A)** The Web Search Subagent — it only retrieved sources about knowledge work
- **B)** The Academic Research Subagent — it only queried white-collar labour databases
- **C)** The Coordinator Agent — its task decomposition only assigned knowledge-work subtopics
- **D)** The Synthesis Subagent — it failed to flag missing labour categories during aggregation

---

### Question 3 — Subagent Context Isolation *(Task 1.2)*

A coordinator agent delegates a web search task to Subagent A and a document analysis task to Subagent B. Subagent A discovers that a key regulatory filing contradicts the company's public statements. The coordinator needs Subagent B to cross-reference this contradiction against internal documents.

What must the coordinator do?

- **A)** Nothing — Subagent B automatically inherits Subagent A's findings through shared memory
- **B)** Explicitly include Subagent A's specific finding in the prompt it sends to Subagent B
- **C)** Instruct Subagent B to query Subagent A's results directly via the Task tool
- **D)** Wait for Subagent A to push its findings to a shared state store that Subagent B can poll

---

### Question 4 — Parallel Subagent Spawning *(Task 1.3)*

A coordinator agent needs to run four independent research tasks. Current implementation spawns them sequentially — one per coordinator turn. End-to-end latency is 40 seconds. What is the correct fix to reduce latency?

- **A)** Increase `max_tokens` on each subagent call to allow faster, more complete responses
- **B)** Emit all four `Task` tool calls in a single coordinator response to spawn them in parallel
- **C)** Use `fork_session` to run the four tasks as independent branches
- **D)** Cache the results of the first task and reuse them across subsequent subagent calls

---

### Question 5 — Programmatic Enforcement *(Task 1.4)*

A financial services agent processes account withdrawals. Production monitoring shows that 4% of withdrawals over £10,000 are processed without triggering the required fraud review. The compliance team has already strengthened the system prompt twice. What is the correct fix?

- **A)** Add 20 few-shot examples demonstrating the correct fraud-review-then-withdrawal sequence
- **B)** Add a routing classifier to direct large withdrawals to a dedicated compliance pipeline
- **C)** Implement a programmatic prerequisite gate: the withdrawal tool cannot execute unless a passed fraud review token is present in session state
- **D)** Rewrite the system prompt to include explicit step-by-step instructions with numbered enforcement checkpoints

---

### Question 6 — PostToolUse Hook Purpose *(Task 1.5)*

A customer support agent calls three different MCP tools: a CRM system returning Unix timestamps for `last_contact`, a billing system returning numeric status codes (1 = active, 2 = suspended), and a logistics API returning ISO 8601 dates. The agent frequently misinterprets the numeric status codes and Unix timestamps.

What is the correct fix?

- **A)** Add instructions to the system prompt explaining how to interpret each data format
- **B)** Implement a PostToolUse hook that normalises all three tool outputs to a consistent format before the model processes them
- **C)** Instruct each subagent to add a data-normalisation step before returning results to the coordinator
- **D)** Rewrite each MCP tool to return data in the same format

---

### Question 7 — Pre-Execution Hook *(Task 1.5)*

An international payments agent has three times completed wire transfers without running the mandatory SWIFT compliance check. Each failure triggered regulatory penalties. The compliance team proposes rewriting the system prompt with more detailed compliance instructions.

Why is the system prompt approach insufficient, and what is the correct fix?

- **A)** System prompts are not read for tool-execution decisions; the fix is to add compliance logic to the tool definition itself
- **B)** The three historical failures demonstrate that prompt-based guidance is already insufficient; a pre-execution hook that blocks the transfer tool unless a passed compliance check is present provides deterministic enforcement
- **C)** System prompts cannot reference external compliance systems; the fix is a post-execution hook that audits completed transfers
- **D)** The system prompt approach is correct; the failures were caused by model hallucination that stronger instructions will resolve

---

### Question 8 — Decomposition Strategy Selection *(Task 1.6)*

A developer productivity agent is tasked with adding test coverage to a large legacy codebase. The codebase has no documentation, no known entry points, and an entirely unknown dependency graph. Which decomposition strategy is most appropriate?

- **A)** Fixed sequential pipeline — analyse each file in alphabetical order, then write tests file by file
- **B)** Dynamic adaptive decomposition — map the structure first, identify high-impact areas, then generate a prioritised plan that adapts as dependencies emerge
- **C)** Parallel fan-out — spawn one subagent per file simultaneously to maximise throughput
- **D)** Iterative refinement — produce a first draft of tests, evaluate coverage, and add more tests until a threshold is reached

---

### Question 9 — Attention Dilution *(Task 1.6)*

A code review agent reviews 18 files in a single pass. The review is detailed and accurate for the first five files, superficial for the middle eight, and misses two critical security vulnerabilities in the last five. What is the root cause and correct fix?

- **A)** Root cause: insufficient `max_tokens`. Fix: increase token limit to allow more thorough output
- **B)** Root cause: attention dilution across too many files in one pass. Fix: split into per-file local analysis passes followed by a separate cross-file integration pass
- **C)** Root cause: the security tool is not being invoked for later files. Fix: explicitly call the security analysis tool for each file
- **D)** Root cause: the model processes files in order and runs out of computation. Fix: randomise file order so important files are not always last

---

### Question 10 — Session Resumption After Code Changes *(Task 1.7)*

A developer runs an architecture review agent on a microservices codebase, producing a thorough report. The developer then refactors three services — `auth-service`, `api-gateway`, and `notification-service` — making significant changes to each. They resume the prior session and ask the agent to validate that the refactoring addressed the identified issues.

The agent gives contradictory recommendations, suggesting fixes that conflict with the newly refactored code. What is the correct approach?

- **A)** Resume the session and instruct the agent to ignore all prior tool results and re-read every file
- **B)** Fork the session so that one branch validates the refactoring while preserving the original analysis in another branch
- **C)** Start a fresh session with no prior context so the agent discovers all issues independently
- **D)** Start a fresh session, injecting a structured summary of the prior valid findings, and direct the agent to re-analyse only the three refactored services

---

## Answer Key and Explanations

<details>
<summary>▶ Reveal All Answers</summary>

### Q1 — Answer: A *(Task 1.1)*

**Correct: A.** The loop checks `block.type == "text"` and returns immediately when it finds a text block. Claude can return a text block (reasoning narration) alongside a `tool_use` block in the same response. The check fires prematurely. The fix is `if response.stop_reason == "end_turn"`.

- **B** is wrong: `tool_result` is what the *developer* sends back, not what Claude returns.
- **C** is wrong: tool results are appended after this check, not before — the ordering is irrelevant to the bug.
- **D** is wrong: checking the last block is not meaningful; `stop_reason` is the correct signal regardless of block position.

---

### Q2 — Answer: C *(Task 1.2)*

**Correct: C.** The subagents researched exactly what they were assigned. The synthesis agent cannot synthesise content it was never given. The root cause is the coordinator's task decomposition, which failed to cover the full scope of the labour market. Trace failures to the component where they originate.

- **A and B** are wrong: the subagents performed their assigned scope correctly.
- **D** is wrong: the synthesis agent cannot flag missing content if it was never aware of the broader scope.

---

### Q3 — Answer: B *(Task 1.2)*

**Correct: B.** Subagents do not share memory or automatically inherit each other's findings. Every piece of information Subagent B needs must be explicitly included in the prompt the coordinator sends to it.

- **A** is wrong: there is no automatic shared memory between subagents.
- **C** is wrong: subagents cannot directly query each other — all communication flows through the coordinator.
- **D** is wrong: there is no built-in shared state store that subagents can poll.

---

### Q4 — Answer: B *(Task 1.3)*

**Correct: B.** Emitting multiple `Task` tool calls in a single coordinator response spawns the subagents in parallel. The total latency approaches the time of the slowest single task rather than the sum of all tasks.

- **A** is wrong: `max_tokens` affects response length, not execution parallelism.
- **C** is wrong: `fork_session` creates independent branches from a shared baseline — it is not designed for independent parallel task spawning.
- **D** is wrong: the tasks are independent research queries; caching results from one and reusing them in another is semantically incorrect.

---

### Q5 — Answer: C *(Task 1.4)*

**Correct: C.** A programmatic prerequisite gate makes it physically impossible to execute the withdrawal tool without a verified fraud review token. This reduces the failure rate to 0%.

- **A** is wrong: few-shot examples remain probabilistic; a non-zero failure rate persists.
- **B** is wrong: a routing classifier moves the request to a different pipeline but does nothing to enforce fraud review *within* that pipeline.
- **D** is wrong: the compliance team has already tried strengthening the system prompt twice; probabilistic approaches cannot achieve the 0% failure rate required for financial operations.

---

### Q6 — Answer: B *(Task 1.5)*

**Correct: B.** A PostToolUse hook intercepts tool results before the model processes them and normalises them to a consistent format. The model always receives clean, correctly formatted data.

- **A** is wrong: system prompt instructions are probabilistic — the model may still misinterpret formats, especially under novel or ambiguous inputs.
- **C** is wrong: the subagents in this scenario are the MCP tools themselves, not separate agents; adding normalisation logic there conflates tool responsibility with orchestration responsibility.
- **D** is wrong: rewriting external MCP tools may not be possible, and it introduces coupling between your orchestration needs and the tool implementation.

---

### Q7 — Answer: B *(Task 1.5)*

**Correct: B.** Three historical production failures confirm that the existing prompt-based guidance is insufficient. Regulatory penalties make any non-zero failure rate unacceptable. A pre-execution hook that physically blocks the transfer tool unless a passed compliance check result is present provides deterministic, 100% enforcement.

- **A** is wrong: system prompts are read for tool-execution decisions; this is not the reason they are insufficient.
- **C** is wrong: a post-execution hook audits after the fact — the illegal transfer has already happened.
- **D** is wrong: the three failures falsify the claim that stronger instructions will resolve the problem.

---

### Q8 — Answer: B *(Task 1.6)*

**Correct: B.** When the structure of the problem is unknown at the start, dynamic adaptive decomposition is required. The agent maps the structure first, discovers high-impact areas, and generates a prioritised plan that adapts as hidden dependencies emerge.

- **A** is wrong: alphabetical file ordering is arbitrary and does not respect dependency structure or risk level.
- **C** is wrong: spawning one subagent per file in parallel ignores dependencies between files — some tests can only be written after understanding cross-file data flow.
- **D** is wrong: iterative refinement addresses *coverage* of tests once the plan is known, not the initial problem of *not knowing* what to test or where.

---

### Q9 — Answer: B *(Task 1.6)*

**Correct: B.** Processing 18 files in a single pass dilutes the model's attention across too much local detail simultaneously. The model cannot maintain consistent scrutiny depth. The fix is a multi-pass architecture: focused per-file local analysis passes (consistent depth per file) followed by a separate cross-file integration pass.

- **A** is wrong: attention dilution is not a token budget problem; more tokens in a single pass still dilutes attention.
- **C** is wrong: the security tool may be invoked correctly; the problem is inconsistent depth, not tool invocation.
- **D** is wrong: processing order does not address attention dilution; later files receive less scrutiny regardless of which files they are.

---

### Q10 — Answer: D *(Task 1.7)*

**Correct: D.** Resuming with stale tool results (the three refactored services) causes contradictory reasoning — the agent's cached knowledge of those services is now wrong. Starting fresh with a structured summary preserves valid prior findings (the unchanged services' analysis) and directs targeted re-analysis at only the three services that changed.

- **A** is wrong: instructing the agent to "ignore prior tool results" within a resumed session is unreliable; the stale results remain in context.
- **B** is wrong: `fork_session` inherits the same stale tool results in both branches — it does not resolve the staleness problem.
- **C** is wrong: discarding all prior context loses valid analysis of the unchanged services, forcing unnecessary re-work.

</details>

---

## Scoring and Next Steps

| Score | Readiness | Action |
|---|---|---|
| **10 / 10** | Exceptional — domain mastery | Proceed to Domain 2 immediately |
| **8–9 / 10** | Ready — minor gaps | Note the 1–2 wrong answers; re-read the corresponding exam trap boxes; proceed to Domain 2 |
| **6–7 / 10** | Not yet ready | Re-read the task statements for your wrong answers in full; redo the practice scenarios; retake this exam |
| **5 or below** | Needs significant review | Re-read the entire domain guide from the beginning; focus on the ⚠️ Exam Traps boxes; retake after 24 hours |

**Map wrong answers to task statements:**

| Wrong on Q | Revisit |
|---|---|
| Q1 | Task 1.1 — Agentic Loop anti-patterns |
| Q2, Q3 | Task 1.2 — Coordinator responsibilities and isolation principle |
| Q4 | Task 1.3 — Parallel spawning |
| Q5 | Task 1.4 — Programmatic enforcement decision rule |
| Q6, Q7 | Task 1.5 — Hooks vs prompts decision framework |
| Q8, Q9 | Task 1.6 — Decomposition strategy selection; attention dilution |
| Q10 | Task 1.7 — Session management and stale context |

---

## Domain 1 Capstone Build Exercise

This exercise consolidates all 7 task statements into a single working system. Complete it before moving to Domain 2.

---

### The Task

> Build a **coordinator agent** with two subagents (web search and document analysis), proper context passing with structured source metadata, a programmatic prerequisite gate, and a PostToolUse normalisation hook. Test with a multi-concern request.

---

### Architecture Specification

```
                    ┌──────────────────────────┐
                    │     COORDINATOR AGENT     │
                    │  (hub-and-spoke centre)   │
                    └────────────┬─────────────┘
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │  Web Search      │  │  Document        │  │  Synthesis       │
   │  Subagent        │  │  Analysis        │  │  Subagent        │
   │                  │  │  Subagent        │  │                  │
   └──────────────────┘  └──────────────────┘  └──────────────────┘
          │                      │
   [PostToolUse hook:      [PostToolUse hook:
    normalise timestamps    normalise page refs
    and status codes]       to consistent format]
          │                      │
          └──────────┬───────────┘
                     ▼
           [Prerequisite Gate:
            synthesis tool blocked
            unless both subagents
            have completed]
```

---

### Build Checklist

Work through each requirement in order. Tick each off before proceeding.

**1. Correct agentic loop** *(Task 1.1)*
- [ ] Loop terminates on `stop_reason == "end_turn"`, NOT content type check
- [ ] Tool results are appended to conversation history before the next API call
- [ ] A safety cap of 20 iterations is present as a backstop (not the primary termination signal)

**2. Coordinator with two subagents** *(Task 1.2)*
- [ ] Coordinator decomposes the incoming request into distinct subtopics
- [ ] Web Search Subagent and Document Analysis Subagent are defined with separate system prompts and tool restrictions
- [ ] Subagents are spawned via the `Task` tool (coordinator's `allowedTools` includes `"Task"`)
- [ ] Subagents are spawned in **parallel** (both `Task` calls emitted in a single coordinator response)
- [ ] Subagents do NOT communicate with each other directly

**3. Structured context passing** *(Task 1.3)*
- [ ] Each subagent's prompt explicitly includes all context it needs (no reliance on implicit inheritance)
- [ ] The synthesis subagent receives structured claim-source metadata from both prior subagents
- [ ] Metadata includes: `claim`, `source_url` or `document_name`, `page` (where applicable)
- [ ] The synthesis prompt specifies quality criteria (cover all major findings; cite sources; flag thin coverage)

**4. Programmatic prerequisite gate** *(Task 1.4)*
- [ ] The synthesis tool is gated: it cannot execute unless session state contains completion tokens from BOTH the web search subagent AND the document analysis subagent
- [ ] The gate is implemented as code (not a system prompt instruction)
- [ ] If the gate blocks execution, an informative error is raised (not a silent failure)

**5. PostToolUse normalisation hook** *(Task 1.5)*
- [ ] Hook fires after each subagent tool result is returned
- [ ] Hook normalises Unix timestamps → ISO 8601
- [ ] Hook normalises numeric status codes → human-readable strings
- [ ] Model only ever receives normalised output — it never sees raw tool results

**6. Test with a multi-concern request** *(Task 1.4 + integration)*
- [ ] Test input: *"What are the current regulatory requirements for AI systems in the EU, and how do the major AI providers (OpenAI, Google, Anthropic) currently comply with them?"* — this requires both web search (current news, compliance statements) and document analysis (regulatory texts)
- [ ] Verify both subagents are spawned in parallel
- [ ] Verify the prerequisite gate fires correctly if one subagent is artificially disabled
- [ ] Verify the PostToolUse hook transforms at least one timestamp and one status code
- [ ] Verify the final report contains source citations drawn from the structured metadata

---

### Reference Implementation Skeleton

Use this as a starting scaffold. Fill in the `TODO` sections.

```python
import anthropic
import json
from datetime import datetime

client = anthropic.Anthropic()

# ─────────────────────────────────────────────
# PostToolUse Normalisation Hook  (Task 1.5)
# ─────────────────────────────────────────────
def post_tool_use_hook(tool_name: str, tool_result: dict) -> dict:
    """Normalise heterogeneous tool outputs before the model processes them."""

    # Normalise Unix timestamps → ISO 8601
    for field in ["created_at", "published_at", "last_modified"]:
        if field in tool_result and isinstance(tool_result[field], (int, float)):
            tool_result[field] = datetime.fromtimestamp(tool_result[field]).isoformat()

    # Normalise numeric status codes → human-readable strings
    status_map = {1: "active", 2: "suspended", 3: "pending", 4: "archived"}
    if "status" in tool_result and isinstance(tool_result["status"], int):
        tool_result["status"] = status_map.get(tool_result["status"], "unknown")

    # TODO: add any additional normalisation required for your specific tools

    return tool_result


# ─────────────────────────────────────────────
# Programmatic Prerequisite Gate  (Task 1.4)
# ─────────────────────────────────────────────
class SessionState:
    def __init__(self):
        self.web_search_complete = False
        self.document_analysis_complete = False
        self.web_search_results = None
        self.document_analysis_results = None

    def mark_web_search_complete(self, results):
        self.web_search_complete = True
        self.web_search_results = results

    def mark_document_analysis_complete(self, results):
        self.document_analysis_complete = True
        self.document_analysis_results = results

    def synthesis_gate(self):
        """Programmatic prerequisite gate — raises if preconditions are not met."""
        if not self.web_search_complete:
            raise RuntimeError(
                "Prerequisite gate: synthesis blocked. "
                "Web search subagent has not completed."
            )
        if not self.document_analysis_complete:
            raise RuntimeError(
                "Prerequisite gate: synthesis blocked. "
                "Document analysis subagent has not completed."
            )
        return True  # Gate passed — proceed with synthesis


# ─────────────────────────────────────────────
# Subagent Definitions  (Task 1.3)
# ─────────────────────────────────────────────
WEB_SEARCH_SYSTEM_PROMPT = """
You are a web search specialist. Your task is to search for current, reliable information
on the topic provided. For EVERY factual claim you return, you MUST include:
  - The claim itself
  - The source URL
  - The publication or retrieval date

Return your findings as a JSON array of claim-source objects:
[{"claim": "...", "source_url": "...", "date": "...", "document": null, "page": null}]
"""

DOCUMENT_ANALYSIS_SYSTEM_PROMPT = """
You are a document analysis specialist. Your task is to analyse the provided documents
and extract relevant information. For EVERY factual claim you return, you MUST include:
  - The claim itself
  - The document name
  - The page number (where applicable)

Return your findings as a JSON array of claim-source objects:
[{"claim": "...", "source_url": null, "date": null, "document": "...", "page": 4}]
"""

SYNTHESIS_SYSTEM_PROMPT = """
You are a research synthesis specialist. You will receive structured findings from
web search and document analysis subagents, each with full source attribution.

Your report MUST:
1. Cover all major findings from both sources
2. Cite the source of every factual claim using the provided metadata
3. Flag any areas where source coverage appears thin or contradictory
4. Be structured with clear headings and a summary section
"""


# ─────────────────────────────────────────────
# Context Passing Helper  (Task 1.3)
# ─────────────────────────────────────────────
def build_synthesis_prompt(
    original_query: str,
    web_results: list,
    doc_results: list
) -> str:
    return f"""
Synthesise the following research findings into a comprehensive report.

## ORIGINAL RESEARCH QUERY
{original_query}

## WEB SEARCH FINDINGS
{json.dumps(web_results, indent=2)}

## DOCUMENT ANALYSIS FINDINGS
{json.dumps(doc_results, indent=2)}

## Quality Criteria
- Cover all major findings from both sources
- Cite the specific source (URL or document + page) for every factual claim
- Flag any topic areas where coverage appears thin
- Produce a structured report with an executive summary, detailed findings, and source list
"""


# ─────────────────────────────────────────────
# Agentic Loop  (Task 1.1)
# ─────────────────────────────────────────────
def run_agentic_loop(
    messages: list,
    tools: list,
    system_prompt: str,
    session_state: SessionState,
    max_iterations: int = 20  # Safety backstop — NOT the primary termination signal
) -> str:
    iteration = 0

    while iteration < max_iterations:
        iteration += 1

        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )

        # PRIMARY termination signal: stop_reason, NOT content type  (Task 1.1)
        if response.stop_reason == "end_turn":
            return next(
                (block.text for block in response.content if hasattr(block, "text")),
                ""
            )

        elif response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    # Apply PostToolUse normalisation hook  (Task 1.5)
                    raw_result = execute_tool(block.name, block.input, session_state)
                    normalised_result = post_tool_use_hook(block.name, raw_result)

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(normalised_result)
                    })

            messages.append({"role": "user", "content": tool_results})

        else:
            # Unexpected stop_reason — surface it rather than silently failing
            raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")

    raise RuntimeError(f"Safety cap reached: exceeded {max_iterations} iterations")


# ─────────────────────────────────────────────
# Tool Executor (stub — replace with real tools)
# ─────────────────────────────────────────────
def execute_tool(tool_name: str, tool_input: dict, session_state: SessionState):
    if tool_name == "web_search":
        # TODO: implement real web search
        results = [{"claim": "placeholder", "source_url": "https://example.com",
                    "date": "2026-03-27", "document": None, "page": None}]
        session_state.mark_web_search_complete(results)
        return {"findings": results}

    elif tool_name == "document_analysis":
        # TODO: implement real document analysis
        results = [{"claim": "placeholder", "source_url": None,
                    "date": None, "document": "example.pdf", "page": 1}]
        session_state.mark_document_analysis_complete(results)
        return {"findings": results}

    elif tool_name == "synthesise_report":
        # Programmatic prerequisite gate fires here  (Task 1.4)
        session_state.synthesis_gate()  # Raises if preconditions not met
        synthesis_prompt = build_synthesis_prompt(
            tool_input["original_query"],
            session_state.web_search_results,
            session_state.document_analysis_results
        )
        # TODO: call synthesis subagent with synthesis_prompt
        return {"report": "TODO: synthesis output"}

    raise ValueError(f"Unknown tool: {tool_name}")


# ─────────────────────────────────────────────
# Entry Point
# ─────────────────────────────────────────────
def run_research_system(query: str) -> str:
    session_state = SessionState()

    initial_messages = [{"role": "user", "content": query}]

    # TODO: define coordinator tools list (web_search, document_analysis, synthesise_report)
    coordinator_tools = []

    return run_agentic_loop(
        messages=initial_messages,
        tools=coordinator_tools,
        system_prompt="You are a research coordinator. Decompose the query, "
                      "spawn web search and document analysis subagents in parallel, "
                      "then synthesise their findings into a comprehensive cited report.",
        session_state=session_state
    )


if __name__ == "__main__":
    result = run_research_system(
        "What are the current regulatory requirements for AI systems in the EU, "
        "and how do the major AI providers (OpenAI, Google, Anthropic) currently "
        "comply with them?"
    )
    print(result)
```

---

### What a Passing Implementation Must Demonstrate

After completing the build, verify each of the following manually:

1. **Loop correctness:** Introduce a deliberate text-block-alongside-tool-use response and confirm the loop does not terminate prematurely.
2. **Parallel spawning:** Log timestamps when subagents start. Confirm both start within 1 second of each other, not sequentially.
3. **Gate enforcement:** Disable the web search subagent. Confirm the synthesis tool raises a clear `RuntimeError` rather than producing a hallucinated report.
4. **Hook normalisation:** Return a raw Unix timestamp (e.g., `1711497600`) from a mock tool. Confirm the model receives `"2024-03-27T00:00:00"` — never the raw integer.
5. **Source citations:** Confirm the final report contains at least one citation directly traceable to the structured metadata, not a hallucinated URL.

---

*You have completed Domain 1: Agentic Architecture & Orchestration. Proceed to Domain 2 when your practice exam score is 8 / 10 or above and you have ticked off all items on the build checklist.*
