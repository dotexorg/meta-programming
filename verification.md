# Verification

An agent that builds a feature and then reviews it for correctness is the same agent — same context, same biases, same blind spots. Verification only works when the reviewer is structurally separated from the builder.

## On This Page

- [The Self-Review Problem](#the-self-review-problem)
- [Deterministic Gates First](#deterministic-gates-first)
- [Separate Builder from Reviewer](#separate-builder-from-reviewer)
- [Multi-Model Review](#multi-model-review)
- [Adversarial Probes](#adversarial-probes)
- [Trajectory and Observability](#trajectory-and-observability)
- [The Cost of Skipping Verification](#the-cost-of-skipping-verification)
- [Open Questions](#open-questions)

---

## The Self-Review Problem

When you ask an agent to review its own output, it praises it. This is not a quirk — it is structural. The model that produced the artifact shares the same reasoning path that led to the artifact. It cannot see what it cannot see.

Amazon learned this at scale. 🟠 **External** — Four Sev-1 incidents in 90 days, including a six-hour outage that lost 6.3 million orders. Their post-mortem framing: *"the creation layer accelerated, but the verification layer stayed the same size."* They simultaneously laid off 30,000 employees while scaling AI output — reducing review capacity at the exact moment volume increased. More code shipping faster into an unchanged review process is not productivity; it is risk accumulation.

Matt Garman, AWS CEO, drew the line explicitly: *"vibe coding"* (blind acceptance of AI output) versus *"augmented coding"* (AI with human oversight). He called replacing junior engineers with AI *"one of the dumbest ideas"* — junior engineers are a core part of the verification layer. ⚪ **Opinion**, but from someone operating at consequence scale.

---

## Deterministic Gates First

The cheapest verification is automated. Run `tsc`, run your test suite, run your linter. These gates catch a large class of errors in milliseconds, with zero hallucination risk. They should be the first stage of every pipeline — not a replacement for review, but a prerequisite for it. [See Pipeline](./pipeline.md) for how this fits as a stage.

The limit is sharp, however. 🟢 **Proven** — In one documented session (Experiment 5), an agent editing an authentication service broke `JIRA_SERVICE_USER` through an incorrect `||` operator split. TypeScript compilation passed. All tests passed. The error was a runtime logic error, invisible to static analysis. A human reviewer caught it.

Deterministic tools alone are not sufficient. They catch what can be specified formally; they miss reasoning errors, misunderstood requirements, and logic that is syntactically correct but semantically wrong. [Specification](./specification.md) is the upstream complement — a precise spec reduces the space of logic errors before any code is written.

Feedback signal types span a hierarchy: execution feedback (exit code), test feedback (pass/fail), static analysis (lint/tsc), AI or human review, and long-horizon outcome signals (PR merged, production incident). Each layer catches different things. Operating only at the bottom two layers leaves the upper ones dark.

---

## Separate Builder from Reviewer

The core principle: builder and reviewer must not share context. 🟠 **External** — this is [Principle 4](./principles.md) for a reason. Different session, different model, different instructions — any of these breaks the shared-blind-spot problem.

The BMAD Adversarial Review pattern makes this structural: the reviewer receives an explicit adversarial mandate. The instruction is not "review this code" but *"you MUST find issues — zero findings triggers a halt."* This eliminates the default toward approval. The reviewer evaluates the artifact, not the intent behind it; it has no access to the author's reasoning, only to the output. ⚪ **Opinion/practice**, but grounded in documented usage.

As mitsuhiko noted: *"You can't vibe yourself back to sanity with better models."* Swapping in a smarter model does not fix a pipeline that routes builder output straight to production. The harness matters more than the model.

---

## Multi-Model Review

Different models have different failure modes. Running the same model twice does not expose those blind spots — but running different models does.

🟠 **External** — BSWEN ran a 133-cycle experiment across four models, with a strict isolation rule: models never see each other's outputs. GPT concentrated on Python idioms and security vulnerabilities. Claude concentrated on reasoning chains and architectural coherence. A race condition in an async handler was found exclusively through the multi-model pass — neither model would have caught it alone, and a single-model re-review of the same code missed it.

Mozilla's Star Chamber skill implements this as fan-out: several providers evaluate the same artifact in parallel, and consensus signals weight the findings. *"Each model notices different things"* is their operational summary. 🟡 **Observed** — consistent with the BSWEN data but separately measured.

OpenAI shipping a `codex-plugin` for Claude Code — cross-model review in production tooling — is the most direct institutional endorsement of this pattern. 🟠 **External**.

---

## Adversarial Probes

Static multi-model review catches many issues; adversarial probes catch a different category — behavioral failures that only emerge under specific inputs.

The Claude Code Verification Agent pattern structures this as a dedicated subagent whose sole role is generating adversarial test cases: edge inputs, invalid states, malformed payloads, permission boundary violations. The key is that the probe agent never evaluates its own probes — it hands them to an execution environment and observes outcomes.

Agent Behavioral Contracts (ABC) formalize this further. 🟠 **External** — In a study of 1,980 sessions, agents operating under formal behavioral contracts (specifying preconditions, invariants, goals, and recovery behaviors) detected 5.2–6.8 soft constraint violations per session that uncontracted agents missed. Overhead was under 10ms per invocation. The contract creates an explicit surface for the probe to test against.

---

## Trajectory and Observability

A verification pass at the end of a task tells you whether the output is correct. Trajectory observability tells you *why* — which tool calls fired, in what order, how many steps were taken, where the agent diverged from the expected path.

🟠 **External** — OpenTelemetry GenAI conventions define standard spans for LLM calls, agent invocations, and tool executions. Datadog, Honeycomb, and New Relic support the spec natively. LangChain, CrewAI, and AG2 emit conforming traces without additional configuration. This means trajectory data is now observable through the same tooling used for service infrastructure.

Promptfoo extends this to assertions: `tool-used`, `tool-sequence`, and `step-count` are all testable properties. A three-layer approach covers the full surface — black-box (does the final output pass?), component (does each agent step behave correctly in isolation?), and trace-based (did the agent follow the expected reasoning path?). A task completed in 47 tool calls when the expected path is 12 is a signal, even if the final output looks correct.

Contrails addresses the longitudinal gap: auto-capturing AI coding sessions as searchable Markdown creates a persistent record of what was built, by what sequence of agent actions, with what intermediate states. The primary use case is incident investigation — when something breaks in production, you can trace back through the session trajectory rather than reconstructing from memory.

---

## The Cost of Skipping Verification

The METR developer study is the sharpest available evidence on this. 🟠 **External** — Developers reported feeling 20% faster. Objective measurement showed they were 19% *slower*. The dominant cause: time spent reviewing and correcting AI output. The subjective confidence of speed coexisted with an objective productivity loss.

This is the vibe-check trap described by Google Cloud: a prompt that works in manual testing 99% of the time can silently degrade to 92% accuracy from model weight drift alone — without any change to the prompt or the application. Continuous evaluation with deterministic assertions is the only defense. Manual review of AI output is not a strategy; it is a feeling. 🟠 **External**.

The pattern is consistent across contexts. Vibe coding at the session level produces outputs that feel correct. Vibe evaluation at the organizational level produces systems that feel reliable. Both are measurement errors. The METR study, the Amazon Sev-1s, and the Google model-drift case all show the same underlying dynamic: the subjective experience of AI-assisted work is systematically more positive than the objective outcomes.

Verification is not a tax on productivity. It is the mechanism by which AI output converts into production-ready software.

---

## Open Questions

- **Cross-session memory for reviewers**: If a reviewer subagent has access to prior review sessions on the same codebase, does accumulated context improve or degrade review quality?
- **Optimal probe surface**: For a given codebase, what adversarial probe categories yield the highest defect detection rate per compute-minute?
- **Drift detection thresholds**: What statistical test distinguishes model-weight drift from legitimate prompt sensitivity, and at what sample size does it become actionable?
- **Human-in-the-loop placement**: At what complexity threshold does AI-only review become insufficient, and how should escalation to human review be triggered automatically?
