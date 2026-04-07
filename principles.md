# Principles

Six principles that explain why some agent workflows succeed and others fail — each backed by experiments, and each actionable from day one.

## The 6 Principles

### 1. Process > Context

Structured workflow beats more information. Giving an agent better scaffolding — spec → plan → implement → review — consistently outperforms giving it more context about the codebase. 🟢 The difference is measurable: the pipeline approach cost $6.63 and passed; the code-map approach cost $9.99 and failed. More context, worse result.

The gap is not marginal. A meta-harness produces a 6x performance improvement over baseline scaffolding (Stanford, 2026) 🟡 — consistent with SWE-bench results showing a custom task harness adds +10% over standard scaffolding. 🟢 The mechanism is not mysterious: a constrained search space routes the model toward solutions faster than raw information volume does.

The instinct to "give the agent everything it needs to know" is often counterproductive. Process constraints are more valuable than information volume. See [Pipeline](./pipeline.md) for this principle in action.

### 2. Boundaries Are Not Optional

Without explicit constraints — what to do, what not to touch, and what terms mean — an agent behaves identically to a raw prompt. Boundaries are not politeness; they are load-bearing.

🟢 A single instruction, `DO NOT modify classifyTicket`, prevented two distinct failure types across repeated runs. The industry has converged on this pattern under the name SDD (Specification-Driven Development) 🟠 — the mechanism is the same in both cases: constrained solution space produces fewer failure modes. Three lines — `DO / DO NOT / GLOSSARY` — prevent roughly 80% of task failures. 🟢

See [Specification](./specification.md) for the full pattern.

### 3. Exploration vs Exploitation

Navigational instructions narrow the agent's search space in ways that hurt outcomes. When you tell an agent *where* to look and *how* to proceed, it stops exploring — and exploration is where correct solutions come from.

🟢 Code maps made results worse, not better. Auto-generated context reduces performance by 3% versus letting the model navigate freely (ETH Zurich) 🟡 — a small delta that compounds badly when the task requires finding non-obvious solutions.

The correct formula: specify **WHAT** to build and **WHY** it matters, add **DO NOT** constraints for what must not change, and leave **HOW** entirely to the agent. See [Context Engineering](./context-engineering.md) for the mechanics.

### 4. Separate Builder from Reviewer

Agents grade their own work generously. An agent that writes code and then reviews it will report success even when the code is broken. Separation is not bureaucracy — it is the only reliable verification path.

🟢 In repeated experiments, self-review produced "excellent" verdicts on broken implementations. A race condition surfaced only under multi-model review — neither model found it alone (BSWEN, 133 cycles) 🟡 — and four Severity-1 incidents at a major cloud provider traced to the identical structural failure in production: agent output promoted without a verification gate (institutional post-mortem) 🟡. Claude Code separates the Verification Agent role for the same reason.

Different models have different blind spots. Multi-model review is not redundancy — it is coverage. See [Verification](./verification.md).

### 5. Atomic Tasks with Context Reset

One task. Verify. Commit. Reset context. Move to the next. This sequence is not overhead — it is the unit of reliable agent work.

🟢 A 106-turn task did not contaminate subsequent tasks after a context reset. The agent's effective working window sits at 1–2% of its nominal context limit (MECW) 🟡 — long-running tasks do not get more done; they get noisier. The commit-and-reset cadence also gives you a clean rollback point after every verified increment.

See [Pipeline](./pipeline.md) for the full sequencing pattern.

### 6. Extraction ≠ Promotion

Not every lesson belongs in long-term memory. Per-session extraction — writing down what went wrong — is cheap and worth doing every time. Cross-session promotion — elevating a finding into the shared knowledge base — requires the strongest available model and deliberate judgment.

Induction quality (the ability to extract generalizable rules from specific cases) correlates directly with model capability (OpenReview) 🟡. This is why `extractMemories` runs on the current model while promotion to the shared KB is reserved for the most capable available — a pattern Claude Code encodes directly in its tooling. Undiscriminating promotion pollutes the KB with noise that future sessions inherit. See [Self-Improvement](./self-improvement.md).

## Level 1 — Minimum Viable Process (Day One)

Three practices you can adopt immediately, before any pipeline or tooling exists.

**Every task gets DO/DO NOT/GLOSSARY.** Write three lines before you write the prompt: what must happen, what must not change, what the ambiguous terms mean in this codebase. 🟢 This alone eliminates the majority of task failures.

**Commands, not prose.** Agents do not respond to values or intent. `run: npm test -- --coverage` changes behavior; "we value well-tested code" does not. 🟡 Controlled testing across 10+ runs per pattern shows prose instructions produce zero measurable behavior change. Every behavioral requirement must be expressed as an executable command or a verifiable condition.

**Specify WHAT, not WHERE.** Resist the urge to point the agent at specific files or tell it which function to edit. Describe the outcome you need and let the agent find the path. Navigation constraints hurt more than they help (see Principle 3).

## Level 2 — Pipeline (First Week)

Once the single-task pattern is stable, introduce structure around the full workflow.

**Separate builder from reviewer.** Run implementation in one session, review in a fresh one — ideally with a different model. The builder session ends at a commit; the reviewer session starts from that commit with no prior context. This is the minimal multi-agent setup, and it works.

**Deterministic checks before LLM review.** TypeScript compilation, linters, and test runners catch entire classes of errors before any token is spent on review. The caveat: they do not catch logic bugs. 🟢 `tsc` passed on code with a significant logic error. Deterministic checks are necessary but not sufficient.

**One task → verify → commit → reset.** The pipeline is the unit. Do not chain tasks without verification steps between them. Each commit is a checkpoint; each reset is a clean start.

**Scout reads, workers implement.** Use a dedicated exploration pass — a scout agent with read-only tools — to gather context before implementation begins. The scout produces a structured summary; workers consume that summary, not the raw codebase. This keeps implementation sessions focused and prevents the over-navigation failure mode.

## Level 3 — Learning Loop (First Month)

Once the pipeline runs reliably, close the feedback loop so each feature makes the next one cheaper.

**Extract lessons after every feature.** At the end of each session, ask: what broke, what was unexpectedly hard, what assumption was wrong? Write it down. The extraction cost is minutes; the compounding value is weeks.

**Spec before code, always.** Writing the specification — acceptance criteria, constraints, definition of done — before implementation catches design problems that cost far more to fix in code. 🟢 The spec phase caught a barrel re-export anti-pattern before any code was written. The spec is not documentation; it is a pre-flight check.

**KB as working memory, not documentation.** The knowledge base is not a README. It is the accumulated understanding of what works and what doesn't in this specific codebase and workflow. Its value is not in what it records — it is in how it changes what the agent (and you) try next.

## Anti-Patterns

**"Make it good."** No DO/DO NOT, no GLOSSARY, no constraints. Fails identically to a raw prompt, regardless of how capable the model is. Boundaries are load-bearing; remove them and the structure collapses.

**Code map as default.** Passing a full file tree and codebase summary feels thorough. In practice it narrows the agent's search, increases cost, and degrades results. 🟢 The code-map run cost 51% more and failed. Use code maps only when you have evidence they help for a specific task type.

**Self-review.** An agent reviewing its own output will mark broken code as excellent. This is not a model quality issue — it is a structural property of single-agent review. Never use the same session for implementation and verification.

**Monster tasks.** Tasks spanning 50+ files do not get done — they get hallucinated. Decompose until each task touches fewer than 10 files and has a verifiable output. If you cannot write an acceptance test for it, the task is too large.

**Trust without verification.** Four Severity-1 incidents traced to one root cause — agent output promoted to production without a verification gate 🟡. The pipeline exists precisely to make verification unavoidable, not optional.

## Complementary Implementations

These principles are validated against production systems, not derived from theory. Claude Code implements several of them directly: the Verification Agent enforces Principle 4, and tiered memory promotion enforces Principle 6. 🟠 That convergence suggests the patterns are load-bearing rather than stylistic preferences.

The distinction between a framework like this and any production implementation is specificity. Production systems solve engineering problems — tool design, memory architecture, model selection. These principles address the meta-layer: why certain workflow shapes succeed, what failure modes are structural rather than incidental, and how to adapt when the standard playbook doesn't fit.

Teams building on top of existing agent infrastructure can apply these principles without changing the underlying tooling. The workflow is the variable; the implementation is the constant. ⚪

## Open Questions

- At what task complexity does the scout→worker split stop paying for itself?
- Can extraction quality be reliably measured, or does it require human judgment indefinitely?
- Does multi-model verification scale to teams, or does it require centralized model access?
