# Verification

An agent that builds a feature and then reviews it for correctness is the same agent — same context, same biases, same blind spots. Verification only works when the reviewer is structurally separated from the builder.

## The Self-Review Problem

Agents praise their own work — confidently, consistently, and incorrectly. 🟢 Across Experiments 3 and 5, we asked implementation agents to evaluate their own output. The verdict was always positive, even when the code contained logic errors, dead files, and broken runtime behavior. The pattern is not occasional — it is the default.

This is not a quirk of one model. The agent that produced the artifact shares the reasoning path that led to it. It cannot see what it cannot see. Anthropic names this "self-evaluation bias" explicitly in their agent architecture. The model that wrote the code will report success on that code; a different model, or the same model with clean context, will not.

Self-review is even less reliable than it appears on the surface. 🟢 In our model evaluation (Experiment 8), a documented pattern emerged at higher thinking levels: the model formally confirms it will follow a rule, then proceeds to violate it in implementation — not through misunderstanding, but through a "workaround" framing. This soft sycophancy is distinct from straightforward rule violation: it passes a superficial check of stated intent while substantively failing compliance. A verification pass that trusts a model's self-report on spec adherence is checking the wrong signal entirely.

The cost surfaces fast at scale. Four Sev-1 incidents in 90 days at a major cloud provider, including a six-hour outage costing an estimated 6.3 million orders, traced to one structural failure: agent output promoted without a verification gate. 🟡 Their internal diagnosis: *"the creation layer accelerated, but the verification layer stayed the same size."* More code shipping faster into an unchanged review process is not productivity — it is risk accumulation.

Two modes define the risk spectrum: *vibe coding* (accept and ship without review) versus *augmented coding* (human oversight at every stage). The distinction maps directly onto incident rate. Junior engineers are a structural part of the verification layer — removing them hollows it at the exact moment it needs to grow.

## Deterministic Gates First

The cheapest verification is automated. Run your type-checker, run your test suite, run your linter — whatever deterministic tools your stack provides. These gates catch a large class of errors in milliseconds, with zero hallucination risk. They should be the first stage of every pipeline — not a replacement for review, but a prerequisite for it. [See Pipeline](./pipeline.md) for how this fits as a stage.

But deterministic gates have a sharp limit. 🟢 In a production pipeline run (Experiment 5), an agent editing an authentication service broke `JIRA_SERVICE_USER` through an incorrect `||` operator split. The type-checker passed. All tests passed. The error was a runtime logic error — invisible to static analysis — caught only by a separate reviewer agent. In a different pipeline run (Experiment 3), review agents had a 0% first-attempt pass rate across three iterations — and both failure types (missing imports, duplicate exports from wildcard re-exports) were deterministically checkable errors that a type-check and a simple grep would have caught before any LLM review token was spent. 🟢

The lesson from both experiments points in the same direction: deterministic checks are necessary AND insufficient. They catch what can be specified formally — type errors, lint violations, import mismatches. They miss reasoning errors, misunderstood requirements, and logic that is syntactically correct but semantically wrong. Run them before review to clear the easy cases; then send the hard cases to a reviewer with clean context. [Specification](./specification.md) is the upstream complement — a precise spec reduces the space of logic errors before any code is written.

Edit corruption is a third category that both static analysis and logic review miss. 🟢 The Experiment 5 failure above exposed a symptom: code physically broken during editing, with the error invisible to type-checking. We initially attributed this to a platform limitation and built a `multi-edit.ts` extension to compensate. Experiment 7 found the actual cause: the extension itself was bypassing Pi's native fuzzy matching pipeline, which already handles trailing whitespace, Unicode quotes and dashes, NBSP, and NFKC normalization. Removing the extension revealed the true baseline: across 370 historical sessions and 1,527 pre-bug edits, the native error rate was **1.6%**. 🟢 During the bug period, the same metric spiked to 94.1%. A synthetic benchmark on unfamiliar code measured 7.1% (42 edits, 3 errors) — real sessions on familiar codebases run roughly 4× lower. Model choice matters: Opus produces 1.1% edit errors versus Sonnet's 2.2% per-edit, a 3.5× gap at the session level. 🟢 Edit volume is the stronger predictor — sessions with 20+ edits see 2.4% error rates versus 0.9% for lighter sessions. Counter-intuitively, reading the file before editing does not reduce errors; the failures are model oldText generation mistakes, not failures of context. The meta-lesson: before attributing a quality problem to the platform, audit your own tooling. For whole-declaration replacements where the file may have drifted since the last read, `semantic_edit` retains a ~7% advantage by matching on identifier rather than exact text.

Feedback signals form a hierarchy — execution, tests, static analysis, review, long-horizon outcome — each catching a different failure class. Operating only at the bottom two layers leaves the upper layers dark, and that is precisely where AI-generated logic errors accumulate.

## Separate Builder from Reviewer

Builder and reviewer must not share context. 🟢 In our pipeline, an implementation worker produced code that passed type-checking and reported success when asked to self-evaluate. A separate reviewer agent, starting from a clean session with only the spec and the diff, immediately found three bugs: a dead file that should have been deleted, a duplicate comment block, and a string constant silently corrupted during editing — the worker had split a multi-part expression across lines and dropped a prefix, producing code that compiled but would fail at runtime. The worker understood the task correctly but introduced the breakage through the edit operation itself. It could not see the problem because it shared the same reasoning path that caused it — the assumption "I just edited this, so it must be right" was baked into its context.

The structural fix is simple: different session, different model, different instructions — any of these breaks the shared-blind-spot problem. [Principle 4](./principles.md) encodes this as a hard architectural requirement, not a preference.

Separate review is not a guarantee. In our own pipeline runs, the reviewer occasionally missed issues that a human caught later — particularly when the bug was in the *intent* of the spec rather than the *implementation* of the code. A reviewer with clean context sees what the worker broke, but it still trusts the spec it was given. If the spec itself is wrong, the reviewer validates incorrect behavior as correct. This is why spec review (upstream) and code review (downstream) are complementary, not substitutes.

The BMAD Adversarial Review pattern makes structural isolation operational: the reviewer receives an explicit adversarial mandate. The instruction is not "review this code" but *"you MUST find issues — zero findings triggers a halt."* This eliminates the default toward approval. ⚪ The reviewer evaluates the artifact, not the intent behind it; it has no access to the author's reasoning, only to the output.

A smarter model does not fix a pipeline that routes builder output straight to production. The harness determines the ceiling, not the model.

## Multi-Model Review

Different models have different failure modes. Running the same model twice on the same artifact exposes the same blind spots twice — running different models does not.

In our KB experiments, agents with different knowledge bases and different contexts produced qualitatively different outputs on the same task. 🟢 A generic agent proposed a technical migration (replace JSON parsing with function calling). An agent with accumulated project knowledge proposed an architectural redesign (binary router + tools + verification). The difference was not code quality — it was thinking level. The same divergence applies to review: a model that reasons about architecture catches different things than one that focuses on syntax correctness.

A 133-cycle controlled experiment across four models, with strict isolation so models never saw each other's outputs, showed consistent divergence in what each model caught. 🟡 GPT concentrated on Python idioms and security vulnerabilities. Claude concentrated on reasoning chains and architectural coherence. A race condition in an async handler was found exclusively through the multi-model pass — neither model alone would have caught it, and a single-model re-review of the same code missed it.

Cross-model review has moved from experiment to tooling. A `codex-plugin` for Claude Code ships GPT-based review inside a Claude-driven pipeline — cross-model review officially endorsed by OpenAI. 🟡 Pi's `pi-council` extension spawns Claude, GPT, Gemini, and Grok in parallel, aggregating independent opinions to reduce single-model bias. 🟠 The pattern is institutional now, not experimental.

## Pre-flight Before Review

Review agents are expensive. Sending them code that fails type-checking or has obvious import errors wastes tokens on problems a shell command solves in milliseconds.

We learned this through waste, not cost. 🟢 In Experiment 3, a full pipeline run across 24 files required three review iterations before passing. Both failure types (missing import updates, duplicate exports from wildcard re-exports) would have been caught by a type-check pass and a targeted grep. After that run, pre-flight checks became a permanent pipeline stage: type-check, lint, and pattern-match for known error categories run before any review token is spent.

The pre-flight checklist is project-specific, but the pattern is universal:
- Type-check (`tsc --noEmit` for TypeScript, `mypy` for Python, `cargo check` for Rust — whatever your stack has)
- Linter pass (no new violations)
- Grep for known anti-patterns (orphaned operators, duplicate exports, debug statements in production paths)
- Test suite (existing tests still pass)

Pre-flight does not replace review. It clears the mechanical failures so the reviewer can focus on logic, architecture, and spec compliance — the things only a reasoning agent can evaluate.

Not every task needs the full stack. A one-file bug fix can ship with just pre-flight checks and a human glance. A multi-file refactor across domain boundaries needs pre-flight, a separate reviewer agent, and possibly multi-model review. The verification depth should match the blast radius of the change — over-verifying trivial fixes wastes tokens, under-verifying architectural changes wastes production stability.

## Adversarial Probes

Static multi-model review catches many issues; adversarial probes catch a different category — behavioral failures that only emerge under specific inputs.

A dedicated verification subagent whose sole role is generating adversarial test cases — edge inputs, invalid states, malformed payloads, permission boundary violations — adds a layer that passive review misses. The key constraint is that the probe agent never evaluates its own probes — it hands them to an execution environment and observes outcomes. The probing function and the judging function stay separate; collapsing them recreates the self-review problem at the probe level.

Formal contracts sharpen what probes test against. 🟡 In a study of 1,980 sessions, agents operating under formal behavioral contracts — specifying preconditions, invariants, goals, and recovery behaviors — detected 5.2–6.8 soft constraint violations per session that uncontracted agents missed. Overhead was under 10ms per invocation. The contract makes implicit assumptions explicit, giving the probe agent a defined surface to test against rather than an open-ended guessing exercise.

## Trajectory and Observability

A verification pass at the end of a task tells you whether the output is correct. Trajectory observability tells you *why* — which tool calls fired, in what order, how many steps were taken, where the agent diverged from the expected path.

Failures cluster at the Action→Result transition — tool output misinterpretation, not planning errors. 🟡 This means assertion checkpoints should focus on tool output parsing, not just plan compliance. A task completing in 47 tool calls when the expected path is 12 is a signal worth investigating, even if the final output looks correct.

Standard infrastructure now exists for this. 🟡 OpenTelemetry GenAI conventions define standard spans for LLM calls, agent invocations, and tool executions. Datadog, Honeycomb, and New Relic support the spec natively. Promptfoo extends this to assertions: `tool-used`, `tool-sequence`, and `step-count` are all testable properties — a three-layer approach covering black-box (final output), component (each step in isolation), and trace-based (full reasoning path).

Longitudinal capture addresses the incident investigation gap. HuggingFace Agent Traces launched native support for agent trajectory publishing — auto-detecting formats for Claude Code, Codex, and Pi. 🟠 When something breaks in production, you trace back through the session trajectory rather than reconstructing from memory and commit history.

## Silent Degradation

Model quality is not static, and degradation does not announce itself. A production incident in April 2026 demonstrated that quality collapse can happen silently, accumulate across sessions, and defeat every code-level quality gate in the pipeline. 🟢

The incident involved a single Opus instance operating across a three-day window. The behavioral signals were only visible in aggregate: the Read:Edit ratio dropped from 6.6 to 2.0, meaning the model was editing more aggressively without prior exploration. Measured thinking depth fell 67%. Cost per task increased 80×. And despite CLAUDE.md being present in context, the model violated its own documented conventions across multiple sessions — not by misunderstanding them, but by drifting past them without flagging the conflict.

None of this was caught by pre-flight checks or the review pipeline — the logic of each individual edit was defensible. The failure was invisible at the task level and only visible at the session level when trajectory metrics were compared against a prior baseline.

Two structural findings emerged. First, pipeline design provides partial protection: a structured workflow with externalized plans anchors task execution even when model quality drops. The degraded agent still follows the plan written to disk — that plan is not dependent on model state. The pipeline's weakness is perseveration: the model looping on a subtask, accumulating cost, without making forward progress. Externalized plans don't prevent that.

Second, trajectory baselines are the detection layer. 🟢 Read:Edit ratio, step count relative to task complexity, and cost per task tier form measurable baselines. When a session crosses 2σ above baseline on any of these metrics, it warrants investigation before a degraded run compounds into a production incident. The implication for evaluation design: behavioral baselines belong in your evaluation layer alongside output quality gates. A verification pipeline that only checks whether code compiles will miss a model running at 80× normal cost with 67% of its normal reasoning depth.

## The Cost of Skipping Verification

Subjective and objective speed diverge sharply under AI-assisted work. 🟡 Developers reported feeling 20% faster while objective measurement showed they were 19% slower — the dominant cause being time spent reviewing and correcting AI output (METR, controlled study). At scale, 8.1 million pull requests across 4,800 teams showed AI code producing 1.7× more issues per PR, with AI PR acceptance rates at 32.7% versus 84.4% for human code (LinearB, 2026). 🟡

Trust in AI output runs below adoption. 🟡 96% of developers report they do not trust AI-generated code without review — yet only 48% say they actually verify it before use (Sonar, 2025). That gap between stated distrust and actual verification behavior is the direct mechanism behind the LinearB quality delta. Engineers feel skeptical but skip the check anyway, under delivery pressure or because the code looks plausible.

The gap extends beyond individual sessions. A prompt that works 99% of the time in manual testing can silently degrade to 92% accuracy from model weight drift alone — without any change to the prompt or the application. 🟡 Continuous evaluation with deterministic assertions is the only defense. Manual spot-checking of AI output is not an evaluation strategy; it is a feeling.

The underlying dynamic is consistent across contexts. The Sev-1 incidents, the METR study, and the model-drift case all show the same gap between perceived and measured quality. Vibe coding at the session level produces outputs that feel correct. Vibe evaluation at the organizational level produces systems that feel reliable. Both are measurement errors, and neither one is caught by adding more AI to the pipeline.

Verification is not a tax on productivity. It is the mechanism by which AI output becomes production-ready software.

## Open Questions

- **Cross-session memory for reviewers**: If a reviewer subagent has access to prior review sessions on the same codebase, does accumulated context improve or degrade review quality?
- **Optimal probe surface**: For a given codebase, what adversarial probe categories yield the highest defect detection rate per compute-minute?
- **Drift detection thresholds**: Experiment 9 established Read:Edit ratio and thinking depth as degradation signals, but no statistical test has been validated for distinguishing model-weight drift from legitimate prompt sensitivity. At what sample size does such a test become actionable in a production pipeline, and what threshold separates noise from a genuine model-quality shift?
- **Human-in-the-loop placement**: At what complexity threshold does AI-only review become insufficient, and how should escalation to human review be triggered automatically?
