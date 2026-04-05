---
title: "Principles"
description: "6 battle-tested principles for working with coding agents, and how to apply them at each maturity level"
sidebar_position: 2
---

# Principles

Six principles that explain why some agent workflows succeed and others fail — each backed by experiments, and each actionable from day one.

## On This Page

- [The 6 Principles](#the-6-principles)
- [Level 1 — Minimum Viable Process](#level-1--minimum-viable-process-day-one)
- [Level 2 — Pipeline](#level-2--pipeline-first-week)
- [Level 3 — Learning Loop](#level-3--learning-loop-first-month)
- [Anti-Patterns](#anti-patterns)
- [vs Claude Code's Approach](#vs-claude-codes-approach)

---

## The 6 Principles

### 1. Process > Context

Structured workflow beats more information. Giving an agent better scaffolding — spec → plan → implement → review — consistently outperforms giving it more context about the codebase.

🟢 **Proven:** SWE-bench results show a custom task harness produces +10% improvement over standard scaffolding. Our own A/B test is sharper: the pipeline approach cost $6.63 and passed; the code-map approach cost $9.99 and failed. More context, worse result.

The implication is uncomfortable: the instinct to "give the agent everything it needs to know" is often counterproductive. Process constraints are more valuable than information volume. See [Pipeline](./pipeline.md) for this principle in action.

### 2. Boundaries Are Not Optional

Without explicit constraints — what to do, what not to touch, and what terms mean — an agent behaves identically to a raw prompt. Boundaries are not politeness; they are load-bearing.

🟢 **Proven:** A single instruction, `DO NOT modify classifyTicket`, prevented two distinct failure types across repeated runs. 🟠 **External:** The industry has converged on this pattern under the name SDD (Specification-Driven Development). The mechanism is the same: constrained solution space → fewer failure modes.

Three lines — `DO / DO NOT / GLOSSARY` — prevent roughly 80% of task failures. See [Specification](./specification.md) for the full pattern.

### 3. Exploration vs Exploitation

Navigational instructions narrow the agent's search space in ways that hurt outcomes. When you tell an agent *where* to look and *how* to proceed, it stops exploring — and exploration is where correct solutions come from.

🟢 **Proven:** Code maps made our results worse, not better. 🟠 **External:** ETH Zurich found that auto-generated context (their equivalent of a code map) reduced performance by 3% versus letting the model navigate freely.

The correct formula: specify **WHAT** to build and **WHY** it matters, add **DO NOT** constraints for what must not change, and leave **HOW** entirely to the agent. See [Context Engineering](./context-engineering.md) for the mechanics.

### 4. Separate Builder from Reviewer

Agents grade their own work generously. An agent that writes code and then reviews it will report success even when the code is broken. Separation is not bureaucracy — it is the only reliable verification path.

🟡 **Observed:** In our experiments, self-review produced "excellent" verdicts on broken implementations. 🟠 **External:** The BSWEN study found a race condition caught *only* by multi-model review — neither model found it alone. Amazon's internal post-mortem: prioritizing velocity over verification produced four Severity-1 incidents. Claude Code itself separates the Verification Agent role for the same reason.

Different models have different blind spots. Multi-model review is not redundancy — it is coverage. See [Verification](./verification.md).

### 5. Atomic Tasks with Context Reset

One task. Verify. Commit. Reset context. Move to the next. This sequence is not overhead — it is the unit of reliable agent work.

🟢 **Proven:** A 106-turn task did not contaminate subsequent tasks after a context reset. 🟠 **External:** Research on effective context window utilization (MECW) puts the agent's effective working window at 1–2% of its nominal limit. Long-running tasks do not get more done; they get noisier.

The commit-and-reset cadence also gives you a clean rollback point after every verified increment. See [Pipeline](./pipeline.md).

### 6. Extraction ≠ Promotion

Not every lesson belongs in long-term memory. Per-session extraction — writing down what went wrong — is cheap and worth doing every time. Cross-session promotion — elevating a finding into the shared knowledge base — requires the strongest available model and deliberate judgment.

🟠 **External:** OpenReview analysis shows induction quality (the ability to extract generalizable rules from specific cases) correlates directly with model capability. 🟡 **Observed:** Claude Code's own tooling reflects this: `extractMemories` runs on the current model; `/insights` promotion is reserved for Opus.

Undiscriminating promotion pollutes the KB with noise that future sessions inherit. See [Self-Improvement](./self-improvement.md).

---

## Level 1 — Minimum Viable Process (Day One)

Three practices you can adopt immediately, before any pipeline or tooling exists.

**Every task gets DO/DO NOT/GLOSSARY.** Write three lines before you write the prompt. What must happen. What must not change. What the ambiguous terms mean in this codebase. 🟢 **Proven:** this alone eliminates the majority of task failures.

**Commands, not prose.** Agents do not respond to values or intent. `run: npm test -- --coverage` changes behavior. "We value well-tested code" does not. 🟠 **External:** Blake Crosley tested this across 10+ runs — prose instructions produced zero measurable behavior change. Every behavioral requirement must be expressed as an executable command or a verifiable condition.

**Specify WHAT, not WHERE.** Resist the urge to point the agent at specific files or tell it which function to edit. Describe the outcome you need. Let the agent find the path. Navigation constraints hurt more than they help (see Principle 3).

---

## Level 2 — Pipeline (First Week)

Once the single-task pattern is stable, introduce structure around the full workflow.

**Separate builder from reviewer.** Run implementation in one session, review in a fresh one — ideally with a different model. The builder session ends at a commit; the reviewer session starts from that commit with no prior context. This is the minimal multi-agent setup, and it works.

**Deterministic checks before LLM review.** TypeScript compilation, linters, and test runners catch entire classes of errors for free, before any token is spent on review. The caveat: they do not catch logic bugs. 🟢 **Proven (Experiment 5):** `tsc` passed on code with a significant logic error. Deterministic checks are necessary but not sufficient.

**One task → verify → commit → reset.** The pipeline is the unit. Do not chain tasks without verification steps between them. Each commit is a checkpoint; each reset is a clean start.

**Scout reads, workers implement.** Use a dedicated exploration pass — a scout agent with read-only tools — to gather context before implementation begins. The scout produces a structured summary; workers consume that summary, not the raw codebase. This keeps implementation sessions focused and prevents the over-navigation failure mode.

---

## Level 3 — Learning Loop (First Month)

Once the pipeline runs reliably, close the feedback loop so each feature makes the next one cheaper.

**Extract lessons after every feature.** At the end of each session, ask: what broke, what was unexpectedly hard, what assumption was wrong? Write it down. The extraction cost is minutes; the compounding value is weeks.

**Spec before code, always.** Writing the specification — acceptance criteria, constraints, definition of done — before implementation catches design problems that cost far more to fix in code. 🟢 **Proven (Experiment 4):** the spec phase caught a barrel re-export anti-pattern before any code was written. The spec is not documentation; it is a pre-flight check.

**KB as working memory, not documentation.** The knowledge base is not a README. It is the accumulated understanding of what works and what doesn't in this specific codebase and workflow. Its value is not in what it records — it is in how it changes what the agent (and you) try next.

---

## Anti-Patterns

**"Make it good."** No DO/DO NOT, no GLOSSARY, no constraints. Fails identically to a raw prompt, regardless of how capable the model is. Boundaries are load-bearing; remove them and the structure collapses.

**Code map as default.** Passing a full file tree and codebase summary feels thorough. In practice it narrows the agent's search, increases cost, and degrades results. 🟢 **Proven:** our code-map run cost 51% more and failed. Use code maps only when you have evidence they help for a specific task type.

**Self-review.** An agent reviewing its own output will mark broken code as excellent. This is not a model quality issue — it is a structural property of single-agent review. Never use the same session for implementation and verification.

**Monster tasks.** Tasks spanning 50+ files do not get done — they get hallucinated. Decompose until each task touches fewer than 10 files and has a verifiable output. If you cannot write an acceptance test for it, the task is too large.

**Trust without verification.** 🟠 **External:** Amazon's four Severity-1 incidents traced back to one root cause — agent output promoted to production without a verification gate. The pipeline exists precisely to make verification unavoidable, not optional.

---

## vs Claude Code's Approach

Claude Code's engineering is battle-tested at massive scale, and these principles build on top of it rather than against it. The distinction is the level of focus.

CC addresses how to **build** an agent system — tool design, memory architecture, prompt construction, model selection. These principles address how to **use** one — workflow design, failure prevention, learning loops, verification structure.

🟠 **External / ⚪ Opinion:** Several differences are worth naming directly. This framework is explicitly named and A/B tested; CC's process guidance is largely implicit in its tooling. Exploration is formalized here as a principle (Principle 3); CC treats context loading as a positive by default. Multi-model verification is a first-class pattern here; CC's Verification Agent is present but not foregrounded. The KB is framed as working memory that changes behavior, not as documentation. And because these patterns are implemented in Pi, they work across any LLM provider, not just Claude.

The honest summary: CC contributes the engineering foundation. Our contribution is the meta-layer — understanding *why* the patterns work, so you can apply them, adapt them, and extend them when the standard playbook doesn't fit.

---

## Open Questions

- At what task complexity does the scout→worker split stop paying for itself?
- Can extraction quality be reliably measured, or does it require human judgment indefinitely?
- Does multi-model verification scale to teams, or does it require centralized model access?
