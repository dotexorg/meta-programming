# Verification

An agent that builds a feature and then reviews it for correctness is the same agent — same context, same biases, same blind spots. Verification only works when the reviewer is structurally separated from the builder.

## The Self-Review Problem

When you ask an agent to review its own output, it praises it. This is not a quirk — it is structural. The model that produced the artifact shares the same reasoning path that led to the artifact. It cannot see what it cannot see.

The cost surfaces fast at scale. 🟡 Four Sev-1 incidents in 90 days, including a six-hour outage that cost 6.3 million orders (Amazon internal post-mortem). Their diagnosis: *"the creation layer accelerated, but the verification layer stayed the same size."* Simultaneously reducing headcount by 30,000 while scaling AI output cut review capacity at the exact moment volume increased. More code shipping faster into an unchanged review process is not productivity — it is risk accumulation.

Two distinct modes define the risk spectrum: *vibe coding*, where AI output is accepted and shipped without meaningful review, and *augmented coding*, where AI operates under human oversight at every stage. ⚪ The distinction maps directly onto incident rate. Junior engineers, often framed as candidates for replacement by AI, are a structural part of the verification layer. Removing them hollows the system from the bottom up, at the exact moment that layer needs to expand.

## Deterministic Gates First

The cheapest verification is automated. Run `tsc`, run your test suite, run your linter. These gates catch a large class of errors in milliseconds, with zero hallucination risk. They should be the first stage of every pipeline — not a replacement for review, but a prerequisite for it. [See Pipeline](./pipeline.md) for how this fits as a stage.

The limit is sharp. 🟢 In a documented session (Experiment 5), an agent editing an authentication service broke `JIRA_SERVICE_USER` through an incorrect `||` operator split. TypeScript compilation passed. All tests passed. The error was a runtime logic error — invisible to static analysis — caught only by a human reviewer.

Deterministic tools catch what can be specified formally. They miss reasoning errors, misunderstood requirements, and logic that is syntactically correct but semantically wrong. [Specification](./specification.md) is the upstream complement — a precise spec reduces the space of logic errors before any code is written.

Feedback signal types span a hierarchy: execution feedback (exit code), test feedback (pass/fail), static analysis (lint/tsc), AI or human review, and long-horizon outcome signals (PR merged, production incident). Each layer catches a different failure class. Operating only at the bottom two layers — compilation and tests — leaves the upper layers completely dark, and that is precisely where AI-generated logic errors accumulate.

## Separate Builder from Reviewer

Builder and reviewer must not share context. Different session, different model, different instructions — any of these breaks the shared-blind-spot problem. [Principle 4](./principles.md) encodes this as a hard architectural requirement, not a preference.

The BMAD Adversarial Review pattern makes structural isolation operational: the reviewer receives an explicit adversarial mandate. The instruction is not "review this code" but *"you MUST find issues — zero findings triggers a halt."* This eliminates the default toward approval. The reviewer evaluates the artifact, not the intent behind it; it has no access to the author's reasoning, only to the output. ⚪

Swapping in a smarter model does not fix a pipeline that routes builder output straight to production — you can't vibe your way back to sanity with a better model. The harness determines the ceiling.

## Multi-Model Review

Different models have different failure modes. Running the same model twice on the same artifact exposes the same blind spots twice — running different models does not.

🟠 A 133-cycle controlled experiment across four models, with strict isolation so models never saw each other's outputs, showed consistent divergence in what each model caught. GPT concentrated on Python idioms and security vulnerabilities. Claude concentrated on reasoning chains and architectural coherence. A race condition in an async handler was found exclusively through the multi-model pass — neither model alone would have caught it, and a single-model re-review of the same code missed it (BSWEN, 133 cycles).

Each model notices different things. This is the operational summary from a fan-out implementation that evaluates the same artifact in parallel across several providers, weighting findings by consensus. 🟡 The pattern holds independently of the BSWEN experiment and has been validated in production across separate model providers.

Cross-model review has moved from experiment to tooling. 🟠 A `codex-plugin` for Claude Code ships GPT-based review inside a Claude-driven pipeline — institutional endorsement that multi-model verification belongs in the standard build process, not the research backlog.

## Adversarial Probes

Static multi-model review catches many issues; adversarial probes catch a different category — behavioral failures that only emerge under specific inputs.

The Claude Code Verification Agent pattern structures this as a dedicated subagent whose sole role is generating adversarial test cases: edge inputs, invalid states, malformed payloads, permission boundary violations. The key constraint is that the probe agent never evaluates its own probes — it hands them to an execution environment and observes outcomes. The probing function and the judging function stay separate; collapsing them recreates the self-review problem at the probe level.

Formal contracts sharpen what probes test against. 🟡 In a study of 1,980 sessions, agents operating under formal behavioral contracts — specifying preconditions, invariants, goals, and recovery behaviors — detected 5.2–6.8 soft constraint violations per session that uncontracted agents missed. Overhead was under 10ms per invocation. The contract makes implicit assumptions explicit, giving the probe agent a defined surface to test against rather than an open-ended guessing exercise.

## Trajectory and Observability

A verification pass at the end of a task tells you whether the output is correct. Trajectory observability tells you *why* — which tool calls fired, in what order, how many steps were taken, where the agent diverged from the expected path.

Standard infrastructure now exists for this. 🟡 OpenTelemetry GenAI conventions define standard spans for LLM calls, agent invocations, and tool executions. Datadog, Honeycomb, and New Relic support the spec natively. LangChain, CrewAI, and AG2 emit conforming traces without additional configuration — trajectory data flows through the same observability tooling already used for service infrastructure.

Promptfoo extends this to assertions: `tool-used`, `tool-sequence`, and `step-count` are all testable properties. A three-layer approach covers the full verification surface — black-box (does the final output pass?), component (does each agent step behave correctly in isolation?), and trace-based (did the agent follow the expected reasoning path?). A task completing in 47 tool calls when the expected path is 12 is a signal worth investigating, even if the final output looks correct.

Longitudinal capture addresses the incident investigation gap. Auto-capturing AI coding sessions as searchable Markdown creates a persistent record of what was built, by what sequence of agent actions, with what intermediate states. ⚪ When something breaks in production, you trace back through the session trajectory rather than reconstructing from memory and commit history — the difference between root-cause analysis and guesswork.

## The Cost of Skipping Verification

Subjective and objective speed diverge sharply under AI-assisted work. 🟡 Developers reported feeling 20% faster while objective measurement showed they were 19% slower — the dominant cause being time spent reviewing and correcting AI output (METR, controlled study). The subjective confidence of speed coexisted with an objective productivity loss. That gap is not a UX problem; it is a measurement problem.

The gap extends beyond individual sessions. A prompt that works 99% of the time in manual testing can silently degrade to 92% accuracy from model weight drift alone — without any change to the prompt or the application. 🟠 Continuous evaluation with deterministic assertions is the only defense. Manual spot-checking of AI output is not an evaluation strategy; it is a feeling (Google Cloud, production analysis).

The underlying dynamic is consistent across contexts. The Amazon Sev-1s, the METR study, and the Google model-drift case all show the same gap between perceived and measured quality. Vibe coding at the session level produces outputs that feel correct. Vibe evaluation at the organizational level produces systems that feel reliable. Both are measurement errors, and neither one is caught by adding more AI to the pipeline.

Verification is not a tax on productivity. It is the mechanism by which AI output becomes production-ready software.

## Open Questions

- **Cross-session memory for reviewers**: If a reviewer subagent has access to prior review sessions on the same codebase, does accumulated context improve or degrade review quality?
- **Optimal probe surface**: For a given codebase, what adversarial probe categories yield the highest defect detection rate per compute-minute?
- **Drift detection thresholds**: What statistical test distinguishes model-weight drift from legitimate prompt sensitivity, and at what sample size does it become actionable?
- **Human-in-the-loop placement**: At what complexity threshold does AI-only review become insufficient, and how should escalation to human review be triggered automatically?
