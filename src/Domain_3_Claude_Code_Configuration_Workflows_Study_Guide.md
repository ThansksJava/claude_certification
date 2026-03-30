# Domain 3: Claude Code Configuration & Workflows
## Claude Certified Architect (Foundations) — Study Guide
**Exam Weight: 20% | Primary Scenarios: Code Generation, Developer Productivity, CI/CD**

---

## Overview

Domain 3 is the most configuration-heavy domain on the exam. You either know where the files go and what the options do, or you do not. Reasoning alone will not save you. This guide covers all six task statements with exam traps highlighted.

---

## Task Statement 3.1: CLAUDE.md Hierarchy

### The Three Levels

| Level | Location | Version-Controlled? | Scope |
|-------|----------|-------------------|-------|
| User-level | `~/.claude/CLAUDE.md` | ❌ No | Only you. Not shared via git. |
| Project-level | `.claude/CLAUDE.md` or root `CLAUDE.md` | ✅ Yes | Everyone who clones the repo. |
| Directory-level | `<subdirectory>/CLAUDE.md` | ✅ Yes | Only when working in that directory. |

### How They Load
- All applicable levels load simultaneously when Claude Code starts.
- More specific (directory-level) overrides less specific (project-level) on conflicting rules.
- User-level settings merge with project-level but are never pushed to the repo.

### The Exam's Favourite Trap
> **"A new team member is not receiving instructions."**
>
> Root cause: instructions are in `~/.claude/CLAUDE.md` (user-level) instead of `.claude/CLAUDE.md` (project-level).
> New team members cloning the repo get zero user-level config. They only get project-level files.

### Modular Organisation

**@import syntax** — reference external files from CLAUDE.md:
```markdown
@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
```

**.claude/rules/ directory** — topic-specific rule files:
```
.claude/
  rules/
    testing.md
    api-conventions.md
    deployment.md
    security.md
```
Keeps CLAUDE.md lean. Rules files can be independently updated.

### Debugging: /memory Command
- Run `/memory` inside a Claude Code session to see exactly which memory files are loaded.
- **Use this when:** Claude Code behaves inconsistently across sessions or team members.
- This is the diagnostic tool — not re-reading config files manually.

### Practice Scenario Answer
Developer A follows conventions; Developer B (new hire) does not.
- ❌ Wrong answers: B) project CLAUDE.md has errors, C) B needs to re-install, D) branch differences
- ✅ Correct: Instructions are in Developer A's `~/.claude/CLAUDE.md` (user-level), not the repo's `.claude/CLAUDE.md` (project-level). Fix: move instructions to project-level.

---

## Task Statement 3.2: Custom Slash Commands and Skills

### Directory Structure

| Location | Shared? | Use For |
|----------|---------|---------|
| `.claude/commands/` | ✅ Yes (version-controlled) | Team-wide slash commands |
| `~/.claude/commands/` | ❌ No (personal) | Personal slash commands |
| `.claude/skills/` (with SKILL.md) | ✅ Yes | On-demand task-specific workflows |
| `~/.claude/skills/` | ❌ No (personal) | Personal skill variants |

### Skill Frontmatter Options

```yaml
---
name: review
description: Performs a comprehensive code review
context: fork
allowed-tools: [Read, Grep, Glob]
argument-hint: "Enter the file or directory path to review"
---
```

| Option | What It Does | When to Use |
|--------|-------------|-------------|
| `context: fork` | Runs in isolated sub-agent context. Verbose output stays contained. Main conversation stays clean. | Codebase analysis, brainstorming, anything noisy. |
| `allowed-tools` | Restricts which tools the skill can use. | Prevent destructive actions (e.g., no Write/Edit tools during analysis). |
| `argument-hint` | Prompts the developer for required parameters when invoked without arguments. | Any skill that needs a target path or parameter. |

### The Key Distinction (Exam Trap)

| | Skills | CLAUDE.md |
|--|--------|-----------|
| **When active** | On-demand (must be invoked) | Always loaded automatically |
| **Use for** | Task-specific procedures | Universal team standards |
| **Exam trap** | Don't put universal standards in skills | Don't put task-specific procedures in CLAUDE.md |

### Personal Customisation Without Affecting Teammates
Create personal variants in `~/.claude/skills/` with a **different name** (e.g., `my-review` vs team's `review`).
Never override the shared skill — that would affect all teammates.

### Practice Scenario Answer
- `/review` command for everyone → `.claude/commands/review.md` (version-controlled)
- Personal `/brainstorm` with verbose output → `~/.claude/skills/brainstorm.md` with `context: fork` (personal, isolated)

---

## Task Statement 3.3: Path-Specific Rules

### Syntax

```yaml
# .claude/rules/test-conventions.md
---
paths: ["**/*.test.tsx", "**/*.spec.ts", "**/tests/**/*"]
---

# Test file conventions apply here
- Use describe/it blocks, not test()
- Mock external services with jest.mock()
- Snapshot tests require approval before commit
```

### The Critical Advantage Over Directory-Level CLAUDE.md

| Approach | Scope | Problem |
|----------|-------|---------|
| `Directory-level CLAUDE.md` | Files **in that one directory** | Test files spread across 50+ dirs each need their own CLAUDE.md |
| `Path-specific rules with glob` | Files matching the pattern **anywhere in the codebase** | One file catches all test files everywhere |

**Key exam rule:** If conventions must apply to files scattered across many directories (test files co-located with source), use path-specific rules with glob patterns — not directory-level CLAUDE.md.

### Token Efficiency
- Path-scoped rules load **only** when editing matching files.
- Reduces irrelevant context. Lowers token usage.
- Always-loaded instructions (root CLAUDE.md) consume tokens on every interaction regardless of relevance.

### Glob Pattern Examples
```
terraform/**/*        → all Terraform files
**/*.test.tsx         → all React test files anywhere
src/api/**/*.ts       → all TypeScript in the API directory
**/migrations/**/*    → all database migration files
```

### Practice Scenario Answer
Test files co-located with source across 50+ directories:
- ❌ B) CLAUDE.md in every directory — 50+ files to maintain, misses new directories
- ❌ C) Single root CLAUDE.md — always loads for every file, not scoped
- ❌ D) Skills — on-demand, not automatic; requires invocation
- ✅ A) Path-specific rules with `**/*.test.tsx` glob — single file, applies everywhere automatically

---

## Task Statement 3.4: Plan Mode vs Direct Execution

### Decision Framework

**Use Plan Mode when:**
- Complex tasks involving large-scale changes
- Multiple valid approaches exist (need to evaluate before committing)
- Architectural decisions required
- Multi-file modifications (e.g., library migration affecting 45+ files)
- Need to explore the codebase and design before changing anything
- Unfamiliar codebase — understand before acting

**Use Direct Execution when:**
- Well-understood change with clear, limited scope
- Single-file bug fix with a clear stack trace
- Adding a date validation conditional
- The correct approach is already known

### The Explore Subagent
- Isolates verbose discovery output from the main conversation.
- Returns **summaries** to the main conversation, not raw output.
- **Use during multi-phase tasks** to prevent context window exhaustion.
- Think of it as a "research assistant" that reports back cleanly.

### The Combination Pattern (Tested on Exam)
```
Phase 1: Plan Mode  → investigate, map codebase, design approach
Phase 2: Direct Execution → implement the designed approach
```
This hybrid is common in production and appears in exam scenarios.

### Practice Scenario Classification

| Task | Mode | Reasoning |
|------|------|-----------|
| Restructure monolith into microservices | Plan Mode | Multiple approaches, architectural, affects entire codebase |
| Fix null pointer exception in single function | Direct Execution | Clear scope, single file, known fix |
| Migrate logging library across 30 files | Plan Mode | Large-scale, multi-file, need to assess impact before changing |

---

## Task Statement 3.5: Iterative Refinement

### Technique Hierarchy (Most Effective First)

1. **Concrete input/output examples (2–3 examples)** — beats prose descriptions every time. The model generalises from examples more reliably than from descriptions.
2. **Test-driven iteration** — write tests first, share failures to guide improvement. Claude Code converges faster when it can verify correctness.
3. **Interview pattern** — have Claude ask clarifying questions before implementing. Surfaces considerations you would miss in unfamiliar domains.

### When to Batch vs Sequence Feedback

| Approach | When to Use |
|----------|-------------|
| **Single message (batch)** | Fixes interact with each other — changing one affects others |
| **Sequential iteration** | Issues are independent — fixing one does not affect others |

### Example-Based Communication
When prose descriptions produce inconsistent results:
```
Instead of: "Transform the date format to be more readable"

Use:
Input:  { "timestamp": 1711497600 }
Output: { "date": "2024-03-27", "time": "00:00:00 UTC" }

Input:  { "timestamp": 1711584000 }
Output: { "date": "2024-03-28", "time": "00:00:00 UTC" }
```

### Practice Scenario Answer
Developer describes a code transformation in prose. Claude Code interprets it differently each time.
- ❌ More detailed prose — same underlying problem
- ❌ Larger context window — irrelevant to interpretation consistency
- ✅ Concrete input/output examples (2–3 before/after pairs) — eliminates ambiguity through demonstration

---

## Task Statement 3.6: CI/CD Integration

### The -p Flag (Memorise This)
```bash
claude -p "Analyse this PR for security vulnerabilities"
```
- `-p` = **print mode** = non-interactive mode.
- Without `-p`, the CI job **hangs indefinitely** waiting for interactive input.
- This is the single most tested CI/CD fact in Domain 3. Memorise it.

### Structured CI Output
```bash
claude -p "Review this PR" \
  --output-format json \
  --json-schema ./schemas/review-findings.json
```
- Produces machine-parseable structured findings.
- Automated systems can post findings as **inline PR comments**.
- Without `--output-format json`, output is free-form text that requires fragile parsing.

### Session Context Isolation (Exam Trap)
> **"Use the same session that generated the code to review it."**
> 
> This is **wrong**. The same session retains reasoning context from generation, making it less likely to question its own decisions.
> 
> ✅ **Correct:** Use an **independent review instance** for code review.

### Incremental Review Context
When re-running reviews after new commits:
```bash
claude -p "Review changes. Prior findings: [prior findings here]. 
Report ONLY new or still-unaddressed issues."
```
- Include prior review findings in context.
- Instruct Claude to report **only** new or unaddressed issues.
- **Why:** Duplicate comments erode developer trust in the CI tool.

### CLAUDE.md for CI Quality
Document in project CLAUDE.md:
- Testing standards
- Valuable test criteria (what makes a test worth having)
- Available fixtures and test utilities

Without this, CI-invoked Claude Code generates **low-value boilerplate tests** rather than meaningful coverage.

### Practice Scenario Answer
CI pipeline: `claude "Analyse this PR"` hangs indefinitely.
- ❌ B) Increase timeout — the job never finishes regardless of timeout; it's waiting for input
- ❌ C) Use a different model — irrelevant to interactive mode issue
- ❌ D) Add `--verbose` — adds output, does not resolve interactive blocking
- ✅ A) Add `-p` flag: `claude -p "Analyse this PR"` — runs in non-interactive print mode

---

## Domain 3 Exam Traps — Quick Reference

| Trap | Correct Answer |
|------|---------------|
| New team member not getting instructions | Move from `~/.claude/CLAUDE.md` to `.claude/CLAUDE.md` |
| Test conventions across 50+ dirs | Path-specific rules with glob, not CLAUDE.md in every dir |
| Skill for verbose output contaminating chat | Use `context: fork` in skill frontmatter |
| CI job hanging | Add `-p` flag |
| Review same session that generated code | Use independent review instance |
| Universal standards in a skill | Move to CLAUDE.md |
| Task-specific procedures in CLAUDE.md | Move to a skill |
| Large-scale multi-file changes | Plan mode first, then direct execution |
| Prose description inconsistently interpreted | Switch to concrete input/output examples |
| Diagnosing inconsistent behaviour between sessions | Run `/memory` command |

---

## 8-Question Practice Exam

*(Attempt before reading answers)*

**Q1 (3.1)** — A development team has defined Claude Code conventions in their repository. Three senior engineers who set up the tool follow conventions perfectly. Five junior engineers who joined recently get inconsistent behaviour. All are on the same git branch. What is the most likely root cause?

A) The junior engineers' IDE plugins are overriding Claude Code settings  
B) The conventions are stored in `~/.claude/CLAUDE.md` on the senior engineers' machines, not in the repository  
C) The project-level CLAUDE.md has syntax errors that only affect newer versions  
D) The junior engineers need to run `claude --refresh` to pick up the latest settings  

---

**Q2 (3.1)** — A developer reports that their Claude Code session is not applying the correct rules. Which command provides the most direct diagnostic information?

A) `cat .claude/CLAUDE.md`  
B) `claude --debug`  
C) `/memory`  
D) `git diff HEAD~1 .claude/`  

---

**Q3 (3.2)** — A team wants a `/security-audit` skill that performs deep codebase analysis. During development, the team noticed that the skill's verbose output was cluttering the main conversation history and making it hard to track overall progress. Which frontmatter option resolves this?

A) `allowed-tools: [Read, Grep]`  
B) `context: fork`  
C) `argument-hint: "Enter audit scope"`  
D) `output: summary`  

---

**Q4 (3.3)** — A codebase has Terraform files spread across `infrastructure/`, `modules/`, `environments/staging/`, and `environments/production/`. The team wants Terraform-specific conventions to apply automatically when editing any of these files. What is the correct approach?

A) Add a CLAUDE.md to each of the four directories  
B) Add all Terraform conventions to the root CLAUDE.md  
C) Create `.claude/rules/terraform.md` with `paths: ["terraform/**/*", "infrastructure/**/*", "environments/**/*", "modules/**/*"]`  
D) Create a `/terraform-mode` skill that loads conventions on demand  

---

**Q5 (3.4)** — A team needs to migrate from `moment.js` to `date-fns` across 47 files, updating import statements, API calls, and date formatting patterns. Which approach is correct?

A) Direct execution — run `claude "Migrate all moment.js to date-fns"` and let it execute immediately  
B) Plan mode — have Claude map all usages and design the migration approach before making any changes  
C) Plan mode for the first 10 files, then direct execution for the remaining 37  
D) Use a custom skill with `allowed-tools: [Edit]` to perform the migration  

---

**Q6 (3.4)** — During a large multi-phase refactoring task, a developer notices the main conversation has become extremely long and Claude Code is losing track of earlier decisions. The current phase involves extensive codebase exploration. What is the correct solution?

A) Start a new session and re-explain everything from scratch  
B) Use the Explore subagent to isolate discovery output and return summaries to the main conversation  
C) Switch to plan mode for the remainder of the task  
D) Increase the context window by splitting the conversation across multiple files  

---

**Q7 (3.5)** — A developer asks Claude Code to "reformat all API responses to use camelCase keys instead of snake_case." After several attempts, Claude Code applies the transformation inconsistently — some responses are reformatted correctly, others are not. What technique should the developer try first?

A) Write a more detailed prose description specifying every edge case  
B) Provide 2–3 concrete before/after examples of the expected transformation  
C) Use the interview pattern to have Claude ask clarifying questions  
D) Switch to plan mode to map all API response locations first  

---

**Q8 (3.6)** — A CI pipeline contains the following script:
```bash
claude "Review this pull request for code quality issues and output findings as JSON"
```
The pipeline hangs after 30 seconds with no output. What is the correct fix?

A) Add `--timeout 120` to give Claude more time to complete the review  
B) Replace with `claude -p "Review this pull request for code quality issues" --output-format json`  
C) Add `--non-interactive` flag  
D) Pipe the output to a file: `claude "Review..." > findings.txt`  

---

## Practice Exam Answers

| Q | Answer | Key Reason |
|---|--------|------------|
| 1 | **B** | User-level `~/.claude/CLAUDE.md` is not version-controlled; new team members don't receive it |
| 2 | **C** | `/memory` shows exactly which memory files are loaded in the current session |
| 3 | **B** | `context: fork` runs the skill in an isolated sub-agent; verbose output stays contained |
| 4 | **C** | Path-specific rules with glob patterns span the entire codebase; directory CLAUDE.md only covers one dir |
| 5 | **B** | 47-file migration = large-scale, multi-file, multiple valid approaches → Plan mode first |
| 6 | **B** | Explore subagent isolates verbose discovery output; returns summaries to main conversation |
| 7 | **B** | Concrete input/output examples beat prose descriptions; model generalises from examples reliably |
| 8 | **B** | `-p` flag enables non-interactive print mode; without it, CI hangs waiting for input |

**Scoring:**
- 8/8 — Exam-ready on Domain 3
- 6–7/8 — Review weak task statements, then re-test
- Below 6 — Work through missed task statements again with the scenarios

---

## Build Exercise

Set up a project demonstrating all Domain 3 concepts:

```
project-root/
├── CLAUDE.md                          ← Project-level (team standards)
├── src/
│   └── CLAUDE.md                      ← Directory-level (src-specific rules)
├── .claude/
│   ├── CLAUDE.md                      ← Alternative project-level location
│   ├── commands/
│   │   └── review.md                  ← Shared /review command
│   ├── skills/
│   │   └── security-audit.md          ← Shared skill with context: fork
│   └── rules/
│       ├── test-conventions.md        ← paths: ["**/*.test.ts"]
│       └── api-conventions.md         ← paths: ["src/api/**/*"]
└── .github/
    └── workflows/
        └── claude-review.yml          ← CI using -p flag + JSON output
```

**CI workflow (`claude-review.yml`):**
```yaml
- name: Claude Code Review
  run: |
    claude -p "Review this PR for quality issues. Prior findings: ${{ env.PRIOR_FINDINGS }}. 
    Report only new or unaddressed issues." \
    --output-format json \
    --json-schema ./schemas/review.json > findings.json
```

**Skill with fork context (`.claude/skills/security-audit.md`):**
```yaml
---
name: security-audit
description: Performs deep security analysis of the codebase
context: fork
allowed-tools: [Read, Grep, Glob]
argument-hint: "Enter the directory or file path to audit"
---

Perform a comprehensive security audit of $ARGUMENTS...
```

**Test conventions rule (`.claude/rules/test-conventions.md`):**
```yaml
---
paths: ["**/*.test.ts", "**/*.spec.ts", "**/tests/**/*"]
---

All test files must follow these conventions:
- Use describe/it blocks
- Mock all external dependencies
- Each test must have an assertion
```

Test the setup with a multi-concern request that exercises plan mode, the skill, and CI output together.

---

## Summary: The Five Things You Must Memorise for Domain 3

1. **User-level = not shared. Project-level = shared via git.** New team members only get project-level.
2. **`context: fork` = isolated sub-agent.** Verbose output stays contained.
3. **Glob patterns in `.claude/rules/` span the whole codebase.** Directory CLAUDE.md only covers one directory.
4. **`-p` flag = non-interactive mode.** Without it, CI hangs.
5. **Independent review instance for code review.** Same session is less critical of its own output.

