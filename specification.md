# Specification: Telling the Agent What You Mean

Without a spec, an agent is just a raw prompt with better marketing. This page covers the minimum viable contract that makes an agent predictable — from three-line AGENTS.md files to full spec-driven development pipelines.

## On This Page

- [Why Boundaries Aren't Optional](#why-boundaries-arent-optional)
- [The Minimum Viable Spec: DO / DO NOT / GLOSSARY](#the-minimum-viable-spec)
- [The Intent Formalization Spectrum](#the-intent-formalization-spectrum)
- [SDD Tools](#sdd-tools)
- [AGENTS.md as Industry Standard](#agentsmd-as-industry-standard)
- [Constrained Natural Language](#constrained-natural-language)
- [What Actually Works in AGENTS.md](#what-actually-works-in-agentsmd)
- [Fowler's Three Maturity Levels](#fowlers-three-maturity-levels)

---

## Why Boundaries Aren't Optional

An agent without explicit scope boundaries fails in a specific, reproducible way: it does too much, in the wrong direction, with high confidence. 🟢

In controlled ablation testing, the same task submitted twice without a pipeline produced two consecutive failures. Adding a spec with explicit boundaries — including a single `DO NOT modify classifyTicket` directive — resolved the task on the first attempt. The agent didn't need more intelligence. It needed a contract.

This aligns with what Microsoft Research calls the *intent gap*: the semantic distance between what a developer means and what a program actually does. The gap isn't a model capability problem. It's a specification problem. Every ambiguous instruction is an invitation for the agent to guess, and agents are optimistically wrong.

[Principles](./principles.md) covers why Principle 2 (Boundaries Are Not Optional) sits at the foundation of everything else.

---

## The Minimum Viable Spec

Three sections. That's it.

**DO** — what the agent is allowed to build or modify. Scoped tightly. One cohesive feature per spec. If you can't state it in two sentences, you have two specs.

**DO NOT** — what to leave alone. Named functions, files, modules, contracts. `DO NOT modify the public API surface of UserService`. Explicit prohibitions outperform vague guidance every time. 🟢

**GLOSSARY** — terms that mean different things in different contexts. "Ticket" in a support context is not "ticket" in a task-tracking context. Ambiguous terms inside a spec are bugs.

This structure emerged directly from failure analysis. Prose instructions get skimmed. Contradictory priorities create thrash. Vague directives like "be careful" produce no behavior change whatsoever. A command is not a suggestion.

Sizing matters too. Practical experience with BMAD-style quick-dev specs points to a 900–1600 token sweet spot per spec. 🟠 Below 900 tokens, ambiguity risk rises. Above 1600, context dilution sets in — the agent begins treating early constraints as background noise rather than active policy.

The mental model that makes this click: a spec is a program. `DO`/`DO NOT` is a contract. The ticket *is* the prompt. Prompt engineering at the task level is operational policy design. 🟠

---

## The Intent Formalization Spectrum

Microsoft Research RiSE identifies intent formalization as the central unsolved problem in AI-assisted development. 🟠 The challenge isn't writing specs — it's validating them. There's no oracle except the developer.

The spectrum runs from informal to formal:

**Vibe coding** sits at one end. No spec. The developer describes a feeling and the agent interprets freely. Fast for exploration, unreliable for anything that needs to survive a second run.

**AGENTS.md** is the first structured layer. Commands, not prose. Organized by task type. Explicit completion criteria. This is where most teams should start.

**SDD (Spec-Driven Development)** goes further. Functional specs written before code, kept as living documents that evolve with the system. The spec is the source of truth; code is derived from it.

**DSL synthesis** is the far end. A domain-specific language designed so that correct code can be mechanically generated from a valid spec. Currently academic outside narrow domains, but the trajectory is clear.

The practical insight from [ETH Zurich research (Feb 2026)](https://arxiv.org/abs/...) is that auto-generated context actually hurts: it reduced task success by 3%, while human-written boundaries improved it by 4%. 🟠 LLM-generated specs create overly obedient agents that do a lot of plausible-looking busywork. The spec has to come from a human who understands intent.

Interactive test-driven formalization offers a middle path: the agent generates postconditions from natural language, then asks the developer to validate. "Remove duplicates" has at least two valid interpretations — the postcondition makes the agent commit to one before writing a line of code. 🟠

---

## SDD Tools

The ecosystem has consolidated around three positions on the spectrum.

**[spec-kit](https://github.com/spec-kit/spec-kit)** (79K★) targets spec-anchored development. Its thesis: "The lingua franca of development moves to a higher level, and code is the last-mile approach." Five commands — `constitution → specify → plan → tasks → implement` — coordinate over 20 specialized agents. The spec persists after the task and drives future evolution.

**[Kiro](https://kiro.dev)** (AWS) targets spec-first development. The workflow is Requirements → Design → Tasks. It ships separate spec types for features and bugfixes. Kiro's own documentation is direct about positioning: "Use Specs when building complex features. Use Vibe when doing quick exploratory coding." The internal mandate at Amazon — 80% weekly usage across 21K agents — was driven by a hard lesson: 4 Sev-1 incidents in 90 days. 🟠 Velocity without verification is not velocity.

**[Tessl](https://tessl.io)** targets spec-as-source. The developer only edits the spec; code is a generated artifact. This is the most ambitious position and the hardest to retrofit onto existing codebases.

These tools don't replace judgment — they enforce it. Spec review is cheaper than code review. In practice, spec review catches architecture problems before they compound: when an agent proposes a barrel re-export during spec review and you ask "what's that?", the agent can recognize it as an anti-pattern and remove it before a single file is touched. 🟢 That's a zero-cost correction. The same catch at PR review costs hours.

---

## AGENTS.md as Industry Standard

AGENTS.md is now present in 60,000+ public repositories. 🟠 The Linux Foundation has adopted it as the coordination protocol, with Anthropic, Google, Microsoft, and OpenAI as Platinum members. It works across Claude Code, Codex, Cursor, GitHub Copilot, Amp, Windsurf, and Devin — the spec travels with the repository, not with the tool.

Pydantic's adoption illustrates the scaling pattern. Their team analyzed 4,668 pull request comments, extracted implicit reviewer judgment, and distilled it into 150 explicit AGENTS.md rules. 🟠 Implicit knowledge that lived in senior engineers' heads became a machine-readable contract. Every new contributor — human or agent — gets the same baseline.

This is the network effect that makes AGENTS.md worth investing in: one spec, written once, applied everywhere the code goes.

---

## Constrained Natural Language

Volodymyr Pavlyshyn's framing cuts through a lot of noise: there's a 60-year history of constrained natural language in programming — COBOL, SQL, Simula all occupy the space between free-form prose and formal code. 🟠 LLMs may finally provide the compiler that makes constrained natural language a first-class programming interface.

The practical consequence is that AGENTS.md files should be written in constrained natural language, not prose. The difference:

**Prose (fails):** "Please try to avoid making changes to files that aren't directly related to the task."

**Command (works):** "DO NOT modify files outside the `src/features/` directory unless the task explicitly names them."

Prose invites interpretation. Commands define behavior. An agent reading a command either executes it or violates it — there's no third state called "I think this probably means..." The constraint is the specification.

---

## What Actually Works in AGENTS.md

Blake Crosley's patterns, validated across 10+ controlled runs per pattern, cut through the speculation. 🟡

**What fails:**

- Prose paragraphs — agents skim or skip them when under task pressure
- Ambiguous directives — "be careful with database migrations" produces no measurable behavior change
- Contradictory priorities — agents resolve conflicts unpredictably; they don't ask

**What works:**

- **Command-first** — exact invocations, not descriptions. `Run npm test -- --watch=false` not "make sure tests pass."
- **Task-organized** — separate sections for coding, review, and release. An agent scoped to a coding task shouldn't have to parse release procedures to find its constraints.
- **Closure-defined** — explicit done criteria. "Task is complete when: all tests pass, no TypeScript errors, PR description includes migration notes." Without a definition of done, agents optimize for looking finished rather than being finished.

One finding from [ICLR 2026 AMBIG-SWE](https://arxiv.org/abs/...) reinforces this: agents running in non-interactive mode default to resolving ambiguity silently. Without an explicit instruction to surface ambiguity, task resolution rates drop from 48.8% to 28%. 🟠 "Ask when uncertain" needs to be a named instruction, not an assumed behavior.

---

## Fowler's Three Maturity Levels

Martin Fowler's SDD maturity model gives teams a migration path rather than a cliff. 🟠

**Level 1 — Spec-first:** The spec is written before coding begins and used for the current task. Disposable after completion. This is the entry point. Even a 15-minute spec-writing session before handing off to an agent changes the failure rate.

**Level 2 — Spec-anchored:** The spec is kept after the task, lives in the repository, and governs future changes. Evolution happens through spec changes that propagate to code. This is where spec-kit and most team workflows land.

**Level 3 — Spec-as-source:** Developers only edit the spec. Code is a generated artifact that developers don't directly maintain. This requires either narrow domain scope (Tessl's target market) or extremely mature tooling. Don't start here.

The progression maps directly to how an engineer's role changes. Spec-first is still mostly prompt engineering. Spec-anchored shifts the primary skill toward system modeling and spec writing. Spec-as-source makes the developer a spec author, with code review replaced by property verification. The title changes from "prompt engineer" to "spec-driven architect." ⚪

---

## Open Questions

- **Spec validation without an oracle.** Intent formalization works if developers validate generated postconditions — but most won't. What's the minimal validation loop that doesn't require developer discipline to work?
- **Spec decay.** A spec written at project start drifts from implementation over time. What triggers a spec update? Who owns it?
- **Cross-agent spec inheritance.** When Agent A spawns Agent B, how much of A's spec should B inherit? Currently implementation-defined by each orchestrator.

---

*Next: [Pipeline](./pipeline.md) — how specs feed into execution. [Verification](./verification.md) — how spec violations surface at runtime. [Back to overview](./index.md).*
