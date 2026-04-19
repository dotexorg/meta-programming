# Specification: Telling the Agent What You Mean

A spec is a contract, not a description. This page covers what makes that contract work: from the three lines that prevent most failures to the full spectrum of spec-driven development, and what the industry has converged on as the baseline.

## Why boundaries aren't optional

Removing scope boundaries from a task doesn't simplify it. It hands the agent a coin flip where both outcomes are wrong.

We tested this directly. The same Telegram bot task (add a support reply feature) failed twice without a pipeline. The first run corrupted an existing prompt handler. The second misunderstood "reply" entirely and built the wrong thing. Adding a spec with explicit constraints, including one directive (`DO NOT modify classifyTicket`), turned the third attempt into a first-pass success. The agent didn't need more capability. It needed a contract.

The failure mode has a name: the *intent gap*: the semantic distance between what a developer means and what the system does. It isn't a model quality problem. The model interprets ambiguity correctly; it just picks an interpretation. Every underspecified instruction is an invitation to guess, and agents guess with high confidence.

Production data says this is not an edge case. Intent misalignment accounts for roughly 32% of dissatisfactory LLM responses in the wild — the largest single failure category, ahead of hallucination and refusal combined. Only 67% of these get resolved even after the user corrects the agent. Multi-turn conversations make it worse by about 30 percentage points against single-turn. Every user input sits on four layers: the immediate text, the final goal behind it, background desiderata the user never stated ("don't switch language, don't rewrite the API surface"), and the user's right to make a suboptimal choice. Production LLMs are fluent at the first layer, mediocre at the second, and violate the third routinely. The training signal points directly at the gap: RLHF optimises for helpfulness (answer) and productivity (don't clarify). The spec is the only place that encodes the layers the model wasn't trained to ask about.

The more dangerous variant is assumption propagation. The agent misunderstands one requirement early ("auth tokens are user-scoped" when they're org-scoped), builds an internally consistent solution on the faulty premise, three commits later the system "works" but breaks in production. Code reviews catch syntactic issues and style drift; they rarely catch an internally consistent solution to the wrong problem. CodeRabbit's analysis across 470 repositories puts AI-generated code at 1.7× more bugs than human, 75% more logic errors, and 8× more I/O performance bugs. Glossary and DO NOT directives aren't stylistic preference. They're how you prevent an agent from building the wrong thing with full confidence.

Even a fast-path prompt with no boundaries fails identically to a raw prompt. Speed doesn't compensate for direction. [Principles](./principles.md) covers Principle 2 in full.

Controlled academic work quantified this. A 2026 paper from ICN2 Barcelona tested four spec detail levels (full docstrings and doctests, through bare function signatures) across 51 class-generation tasks, with both single-agent and multi-agent setups. Single-agent pass rate dropped from 89% at full spec to 56% at bare signatures. Multi-agent dropped from 58% to 25% on the same tasks. The gap between single and multi is 25–39 percentage points of additional loss from coordination alone. Decomposition of that gap: 16 points from coordination overhead (lack of shared decisions) and 11 points from information asymmetry between workers. Independent effects, additive. The recovery is the counter-intuitive part: adding an AST-based conflict detector at 97% precision shifts the numbers by zero. Restoring the full spec to the merger agent recovers the 89% ceiling. **Specification is both the cause of coordination failure and the sufficient instrument of recovery.** Spend on the spec, not on reconciliation.

## The minimum viable spec

Three sections. That's the entire entry price.

**DO**: what the agent is allowed to build or change. Scoped tightly. One cohesive feature. If you need three sentences to describe it, you have two specs. Acceptance criteria go here: "Task is complete when all existing tests pass and the new endpoint responds within 200ms p95."

**DO NOT**: named prohibitions. Files, functions, modules, contracts. `DO NOT modify the public API surface of UserService`. Explicit prohibitions consistently outperform vague guidance. "Be careful with database migrations" produces no measurable behavior change. `DO NOT run migrations automatically` does.

**GLOSSARY**: terms that mean different things in different codebases. "Ticket" in a support system isn't "ticket" in a project tracker. Ambiguous terms inside a spec aren't ambiguous. They're latent bugs.

This structure came from failure analysis, not theory. Prose instructions get skimmed under task pressure. Contradictory priorities create thrash. Vague directives produce nothing. The DO/DO NOT/GLOSSARY format is the minimum surface that eliminates the most common failure modes without requiring a full requirements document.

Sizing matters. A 900–1600 token sweet spot applies to structured quick-dev specs. Below 900, ambiguity risk rises. Above 1600, early constraints start behaving like background context. Technically present, actively deprioritized. A spec is a program. Treat it like one.

An academic formalization of this space names five roles that a complete context package needs to carry: Authority (who decides), Exemplar (concrete examples), Constraint (hard limits), Rubric (success criteria), Metadata (versioning and provenance). 200 documented agent interactions across four tools showed incomplete context triggers 72% of iteration cycles. A structured package against a baseline cut iterations from 3.8 to 2.0 and raised first-pass acceptance from 32% to 55%. DO/DO NOT/GLOSSARY handles three of the five roles directly — authority implicit in the author, exemplars and rubrics are the parts most specs skip. When boundaries alone don't recover quality, those are the usual missing pieces.

## The intent formalization spectrum

The central unsolved problem in AI-assisted development isn't writing specs. It's validating them without an oracle. The developer is the oracle. Most won't validate carefully. Microsoft Research named this the grand challenge of *Intent Formalization*.

The spectrum runs from informal to formal:

**Vibe coding**: no spec, agent interprets freely. Fast for throwaway exploration, unreliable for anything repeatable.

**AGENTS.md**: first structured layer. Commands organized by task type, explicit done criteria. Where most teams should start, and many should stay.

**SDD (Spec-Driven Development)**: functional specs written before code, kept as living documents. The spec is the source of truth; code is derived from it.

**DSL synthesis**: a domain-specific language from which correct code is mechanically generated. Currently academic outside narrow domains. The trajectory is clear; the tooling isn't there yet.

One result worth sitting with: auto-generated context reduced task success by 3%, while human-written boundaries improved it by 4%. LLMs writing their own constraints produces agents that do a lot of plausible-looking work in the wrong direction. Specs have to come from people who understand intent.

Interactive test-driven formalization offers a practical middle path: the agent generates postconditions from natural language, the developer validates before any code runs. "Remove duplicates" has at least two distinct valid interpretations. The postcondition forces a commit to one of them upfront.

## SDD tools and Fowler's maturity levels

Teams mature through three levels:

**Spec-first**: spec written before coding, discarded after. The entry point. Even a 15-minute writing session before handoff changes the failure rate, because it forces the developer to confront ambiguity before the agent does. Kiro (AWS) targets this level explicitly: "Use Specs for complex features. Use Vibe for quick exploratory coding."

**Spec-anchored**: spec kept after the task, living in the repo, governing future changes. Spec-kit (GitHub, 79K★) is built for this: five commands: `constitution → specify → plan → tasks → implement`, coordinating 20+ agents including Claude Code, Copilot, Cursor, and Gemini. The spec travels with the codebase; every agent that touches it works from the same contract.

**Spec-as-source**: only the spec is authored by humans; code is a generated artifact. Tessl's target. The most ambitious position and the hardest to retrofit onto anything existing. Don't start here.

Velocity scales faster than verification. That's the failure mode Kiro's internal mandate at Amazon made explicit: 80% weekly usage across 21,000 agents, 4 Sev-1 incidents in 90 days. Spec-first development prevents building the wrong thing quickly. It doesn't guarantee the right thing was built correctly. That's a [Verification](./verification.md) problem.

Spec review is cheaper than code review. In practice: when an agent proposed a barrel re-export strategy during spec review, the developer asked what that was; the agent recognized it as an anti-pattern and removed it before touching a single file. Zero-cost correction. The same catch at PR review costs hours of context reconstruction and broken test runs.

## AGENTS.md as the coordination standard

AGENTS.md is now in 60,000+ public repositories. The Linux Foundation governs it as the cross-tool coordination protocol, with Anthropic, Google, Microsoft, and OpenAI as Platinum members. It works across Claude Code, Codex, Cursor, GitHub Copilot, Amp, Windsurf, and Devin. The spec travels with the repository, not the tool.

Implicit reviewer judgment can be systematically codified. Pydantic analyzed 4,668 pull request comments and extracted 150 AGENTS.md rules. Knowledge that lived in senior engineers' heads became a machine-readable contract. Every contributor (human or agent) starts with the same baseline. One spec, written once, applied wherever the code goes.

Controlled testing across 10+ runs per pattern identifies what actually changes agent behavior:

What gets ignored: prose paragraphs, ambiguous directives like "be careful with migrations," contradictory priorities without explicit ordering, style guides without enforcement commands.

What works: exact commands (`ruff check --select D`), task-organized sections that separate coding from review from release, explicit done criteria with verifiable conditions ("task is complete when: all tests pass, no TypeScript errors, PR description includes migration notes"), numbered priorities when conflicts are likely.

One instruction that's consistently missing from AGENTS.md files: surface ambiguity. Agents in non-interactive mode resolve ambiguity silently by default. Task resolve rate drops from 48.8% to 28% without an explicit directive to ask. "Ask when uncertain" is a behavioral requirement, not a social norm. Write it in.

## Constrained natural language: commands, not prose

The space between free-form prose and formal code has a 60-year history. COBOL, SQL, Simula all occupy it. Each decade brought abstractions that made programming more cognitively accessible. LLMs may be the bridge that finally makes constrained natural language a first-class programming interface, acting as a compiler for structured intent.

The practical consequence: AGENTS.md should be written in commands, not prose. The difference is sharper than it sounds.

**Prose:** "Please try to avoid making changes to files that aren't directly related to the task."

**Command:** `DO NOT modify files outside src/features/ unless the task explicitly names them.`

Prose invites interpretation. A command defines behavior. The agent either executes it or violates it. There's no third state called "I think this probably means." Ambiguity is a feature for human communication and a fatal flaw for executable systems.

This extends beyond AGENTS.md. [Layer 2 in the LMP framework](./index.md) is constrained natural language specific to your project: your stack, your patterns, your conventions. AGENTS.md is the generic version. A personal intent language is AGENTS.md that evolved from your corrections over time, specific enough that it can't be swapped into another team's codebase without rewriting half of it.

## Open questions

**Spec validation without a practical oracle.** Interactive test-driven formalization works if developers validate generated postconditions. Most won't, not consistently. The open question: is there a minimal feedback loop that captures validation signal without requiring active developer discipline? Every current approach (postcondition generation, disambiguation prompts, test-first formalization) assumes the developer will engage at validation time. We haven't found a setup that doesn't.

**Emergent specification faithfulness.** Purdue's SLUMP work (Specification Loss Under eMergent sPecification) formalises a distinct failure: specs that emerge during a session rather than being written upfront drift from the original problem statement as the conversation extends. Their proposed fix — an external state tracker (ProjectGuard) that every worker checks against — recovers 90% of the faithfulness gap and cuts severe failures from 72 to 49 on their benchmark. Our `spec.md` plus `plan.md` is a weaker version of the same idea: persistent artefacts between workers. We don't systematically enforce "every worker re-reads the spec before acting." That's the missing enforcement step.

**Spec decay.** A spec written at project start drifts from the implementation as features accumulate. Implementation outpaces the update cadence. No current tool detects the drift automatically. The mismatch surfaces only when a new agent reads a stale spec and builds for the wrong target. What triggers a spec update, and who owns that process, remains unresolved.

**Cross-agent spec inheritance.** When a coordinator spawns worker agents, how much of the coordinator's spec should each worker inherit? Boundary constraints? Glossary only? Full DO/DO NOT? Currently each orchestrator decides independently, with no standard. An agent inheriting too little scope fails differently than one inheriting too much, but we don't have a principled model for where to draw the line.

*See also: [Pipeline](./pipeline.md) (how specs feed into execution), [Verification](./verification.md) (how spec violations surface at runtime. [Principles](./principles.md), Principle 2 in full.*
