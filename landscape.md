---
title: "Landscape"
description: "Who else is doing this (April 2026)"
sidebar_position: 4
---

# Landscape

The meta-programming space moved from experimental to mainstream between late 2025 and April 2026. This page maps the tools, studies, and people defining it right now.

## On This Page
- [Trends: April 2026](#trends-april-2026)
- [Cursor 3: Agent-First IDE](#cursor-3-agent-first-ide)
- [Amazon: The Vibe Coding Failure](#amazon-the-vibe-coding-failure)
- [Key Tools](#key-tools)
- [Semantic Edit Approaches](#semantic-edit-approaches)
- [Pi Ecosystem](#pi-ecosystem)
- [Academic Work](#academic-work)
- [People to Follow](#people-to-follow)
- [Open Questions](#open-questions)

---

## Trends: April 2026

Four shifts define the current moment.

**SDD went mainstream.** Specification-Driven Development moved from a niche practice to an expected workflow in roughly six months. spec-kit hit 79K stars, Kiro launched with an 80% internal mandate, and Tessl entered the space — all treating the spec as the primary artifact and code as last-mile output. [See Specification](./specification.md) for tool depth.

**Agent-first design became the IDE default.** Cursor 3 (Apr 2, 2026) is the clearest signal: a complete rebuild around parallel agents rather than a chat sidebar bolted onto an editor. The interface assumption shifted from "one conversation" to "fleet coordination."

**Self-improving agents went mainstream.** Three independent projects shipped autonomous self-improvement loops in a single week. Karpathy's autoresearch loop ran 70+ experiments and raised a chess engine from expert to grandmaster rank #311 🟢. The pattern is no longer research-only.

**Token optimization emerged as a discipline.** Semantic editing (read by symbol, edit by function name) and semantic diff (Relace, 4300+ tok/s, ~40% token reduction 🟠) are now first-class concerns alongside prompt quality. Effective context utilization — the MECW study found models use 1-2% of advertised context window 🟠 — is the underlying problem these tools address.

Two structural moves matter in the background: OpenAI acquired Astral (uv/ruff), Anthropic acquired Bun. Both companies are racing to own the toolchain layer around coding agents. 🟠 MCP has become the de facto integration standard — 50+ official servers, described as "USB-C of AI tool integration." 🟠

---

## Cursor 3: Agent-First IDE

**Released April 2, 2026.** Cursor 3 is not an incremental update — it's a complete architectural rebuild. 🟠

The core change: Composer 2 replaces the single-agent model with a parallel agents sidebar that runs both local and cloud agents simultaneously, with seamless handoff between them. Multi-workspace support means a developer can run isolated agent contexts per feature branch. A new marketplace covers MCPs, skills, and subagents.

The underlying model is Kimi K2.5, reporting 73.7 on SWE-bench multilingual. Cursor's framing — "fleets of agents work autonomously" — signals that the unit of work is shifting from a single LLM call to coordinated agent sessions.

For teams using Pi, Cursor 3 is the primary competitor reference point for multi-agent UX patterns.

---

## Amazon: The Vibe Coding Failure

The most-cited failure case study of early 2026. 🟠

Amazon's internal mandate — Kiro, 80% usage across 21,000 agents, targeting $2B in value — produced four Sev-1 incidents in 90 days. The largest: a 6-hour outage that caused 6.3 million lost orders. The root cause described internally: *"the creation layer accelerated, the verification layer stayed the same."* 30,000 employees were laid off in the restructuring that followed.

The lesson is not that agent coding fails. It's that generation velocity without a proportional investment in verification creates systemic fragility. SDD addresses the specification gap; observability tooling (OTel, Promptfoo) addresses the verification gap. Neither alone is sufficient. [See Verification](./verification.md) for the full observability picture.

---

## Key Tools

**spec-kit** (GitHub, 79K★) is the leading SDD scaffold. Five commands take a project from idea to agent-ready context. Its explicit thesis: "the lingua franca moves to a higher level; code is last-mile." 🟠

**DSPy** (Stanford, 33K★) treats prompts as learnable parameters rather than hand-written strings. Optimizers — BootstrapFewShot, MIPROv2, GEPA — search over prompt space the way gradient descent searches over weights. Of all current tools, DSPy maps most directly to the "Layer 3" vision of self-optimizing agent pipelines. 🟠 [See Self-Improvement](./self-improvement.md).

**Promptfoo** has evolved into a three-layer testing harness: unit-level prompt evals, agent trajectory assertions, and full OTLP tracing for production observability. Red team mode runs adversarial probes automatically. It's the closest thing to a complete quality gate for agentic systems. 🟠

**AGENTS.md** is a cross-tool context standard now present in 60,000+ repos and formalized under the Linux Foundation. 🟠 Where CLAUDE.md is Claude-specific, AGENTS.md is tool-agnostic — a convergence point for the ecosystem. The markdown convergence trend more broadly (every major tool reading project-context files) makes structured specification a durable investment rather than a platform bet.

---

## Semantic Edit Approaches

A cluster of projects is solving the same problem: LLMs working on code should operate on semantic units (functions, classes, symbols), not raw text strings. Four approaches are worth knowing.

**AFT** (ualtinok) provides symbol-level read and edit tools. Reading by symbol costs ~40 tokens versus ~375 for a full file — roughly a 10x reduction. 🟠 Edit targets a function name rather than a line range, making edits robust to file changes between planning and execution. Multi-language support via tree-sitter.

**Relace Instant Apply** approaches the problem from the diff side. Rather than the model reproducing changed file text, Relace computes a semantic diff. Reported throughput: 4,300+ tokens/second with ~40% token reduction versus full-file rewrite. 🟠

**tree-sitter AST-aware editing** is being adopted directly in agent harnesses. The framing from neil_agentic: "when tree-sitter gives you the AST, why make the model reproduce strings?" CC's default text-match edit was replaced with an AST-aware version, reducing edit failures on reformatted code. 🟡

**Pi's `semantic_edit`** tool follows the same pattern — edit by identifier, immune to whitespace drift. For Pi users this is the default path for whole-declaration replacements.

---

## Pi Ecosystem

Three community extensions that define the current Pi meta-programming surface.

**ohmypi** implements multi-model routing at the orchestration layer. The reference configuration pairs GPT-5.4 as orchestrator with Claude Opus 4.6 as specialist — using each model where its strengths apply rather than routing all tasks to a single provider. 🟡

**pi-subagents** (636★) adds the `subagent` tool as a Pi extension. The key implementation detail is `createBranchedSession` — fork-context mode that gives each subagent an isolated copy of parent context without serialization overhead. 🟠

**pi-autoresearch** (2,288★) packages Karpathy's autoresearch loop as a Pi extension. An agent generates a hypothesis, runs an experiment, evaluates the result, and iterates — with no human in the loop between cycles. The chess ELO benchmark (expert → grandmaster #311 over 70+ experiments 🟢) is the most concrete published result for autonomous self-improvement.

**pi-council** spawns Claude, GPT, Gemini, and Grok in parallel on the same problem and aggregates independent opinions — useful for high-stakes architectural decisions where single-model bias is a concern. 🟡

The broader pattern: features that Anthropic ships (extended thinking, extended context, Projects) tend to absorb community tools over time. Platform convergence is accelerating. Investing in harness patterns that are model-agnostic reduces this risk.

---

## Academic Work

Five research threads with direct practitioner relevance.

**Microsoft RiSE** identifies Intent Formalization as the "grand challenge" of software engineering with AI — the gap between what a developer means and what a specification captures. 🟠 This is the theoretical framing behind SDD: closing the intent gap upstream is cheaper than debugging it downstream.

**METR study** produced the most important calibration finding of 2025: developers *felt* 20% faster with AI assistance; they were actually 19% slower. 🟢 The subjective experience of AI tools is a systematically unreliable performance signal. Measure; don't feel.

**ABC (Agent Behavioral Contracts)** proposes formal contracts for agent behavior — analogous to design-by-contract for functions. Tested across 1,980 sessions with under 10ms runtime overhead. 🟠 Still academic, but the closest formal analog to what AGENTS.md does informally.

**ExpeL** (Experience Learning) formalizes an experience loop: gather episodes → extract reusable patterns → apply patterns to new tasks. 🟠 Not coding-specific, but the architecture maps directly onto how self-improving agent harnesses should accumulate knowledge. [See Self-Improvement](./self-improvement.md).

**MECW** (Minimum Effective Context Window) found that models effectively use 1-2% of their advertised context window. 🟠 The implication for harness design: context curation is not optional. Semantic chunking, relevance filtering, and symbol-level reads exist to serve this constraint.

---

## People to Follow

**Simon Willison** (@simonw) — Django co-creator, author of the 12-chapter *Agentic Engineering Patterns* guide. The most thorough public practitioner writing on agent system design. When he publishes something, read it. 🟠

**Mario Zechner** (@badlogicgames) — libgdx author, Pi creator. Primary source on Pi internals, subagent architecture, and the `createBranchedSession` fork-context pattern. 🟠

**Andrej Karpathy** (@karpathy) — The autoresearch loop and the chess ELO benchmark are his. His public writing on autonomous experimentation is the clearest existence proof that self-improving systems work at scale today. 🟢

**Harrison Chase** (@hwchase17) — LangChain founder. His framing: "learning at the harness layer produces compounding returns." Worth following for visibility into what the LangChain ecosystem is converging on. 🟠

---

## Open Questions

- Does the METR slowdown effect persist after teams adopt SDD workflows, or is it a symptom of spec-less generation?
- At what agent fleet size does pi-council-style consensus checking become faster than single-model iteration?
- Will AGENTS.md formalization under Linux Foundation produce a stable v1 spec, or fragment into tool-specific dialects?
- How does effective context utilization (1-2% MECW) interact with very long agent trajectories — does it degrade further?

---

*[← Index](./index.md) · [Specification →](./specification.md) · [Verification →](./verification.md) · [Self-Improvement →](./self-improvement.md)*
