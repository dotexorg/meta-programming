# Landscape: who else is doing this (April 2026)

A ground-level map of April 2026, which tools have traction, which experiments blew up in production, and whose work shows up in our own thinking. Labels are honest. Nothing is here because it sounds important.

## Trends: what shifted in Q1 2026

Four things moved at once. Each would have been a year's conversation on its own.

Spec-driven development went from niche to default. spec-kit: 79,000 stars. Kiro: an entire IDE built around requirements → design → tasks. Tessl: skill registry with security scoring. Fowler documented the space in October 2025. GitHub's thesis is blunt: "code is the last-mile approach." Specs are the real artifact. The industry is catching up. 🟡

Agent-first IDEs replaced toolbars. The old model: editor with AI sidebar. The new model: agents are primary, editor is a view. Parallel agents, marketplace extensions, local↔cloud handoff. Baseline now, not differentiators. 🟠

OTel is becoming the observability layer. GenAI SIG defines schemas for LLM calls, agent steps, tool executions. Datadog, Honeycomb, New Relic support them natively. LangChain, CrewAI, AG2 emit OTel spans. Promptfoo added OTLP tracing. Agents are production services now. The tooling says so. 🟡

Self-improvement loops without model access. Three tools launched in one week (skill-loop, selfwrite, iterate). All focused on improving instructions, not weights. One practitioner auto-promoted 13 error patterns to behavioral rules in two weeks. Just logging. Just counting. The pattern is real. 🟠

## Cursor 3: agent-first IDE (Apr 2, 2026)

Cursor rebuilt from scratch. Not an upgrade. A rewrite.

The previous version was VS Code with Claude bolted on. Version 3 treats agents as the primary primitive. The editor is a view into what agents are doing, not the other way around. 🟡 Parallel agents in the sidebar. Multi-workspace. Cloud/local handoff. A marketplace for MCPs, skills, and subagents.

Default model: Composer 2 (Kimi K2.5), 73.7 on SWE-bench multilingual, 61.3 on CursorBench. "Fleets of agents work autonomously to ship improvements". Their words, not ours. 🟡 The gap between "AI-assisted" and "agent-led" is now visible in shipping product.

## Amazon vibe coding failure: a case study

Four Sev-1 incidents in 90 days. 21,000 agents at 80% weekly usage. $2B savings target. One outage: 6 hours, ~6.3 million lost orders. 🟡

The failure was structural. Amazon laid off ~30,000 employees and scaled AI code generation simultaneously. Less review capacity, more volume. Matt Garman drew the line: "vibe coding" (blind acceptance) vs "augmented coding" (AI plus oversight). Amazon built the creation layer. The verification layer stayed the same size. 🟡

SDD is partly a response to this. Spec-driven workflows force humans to articulate intent before code runs. The question isn't whether AI should write code. It's whether anyone in the loop still understands what the code is supposed to do. Amazon answered that. The answer was no.

## Key tools worth knowing

**spec-kit** is GitHub's 5-command SDD workflow: constitution → specify → plan → tasks → implement. Twenty-plus agents, 79,000 stars. The open-source complement to Kiro. 🟡

**DSPy** (Stanford, 33,000 stars) shifts the abstraction from prompt strings to program structure. The optimizer (BootstrapFewShot, MIPROv2, GEPA) automatically improves prompts against a metric you define. It's the closest open-source implementation of a closed-loop Layer 3 system. Programming agents rather than prompting them. 🟡

**Promptfoo** was acquired by OpenAI in March 2026 for $86M. 350,000 developers, 130,000 MAU, 25% Fortune 500 adoption before the acquisition. Trajectory assertions, red teaming, 3-layer testing, now with OTLP tracing. The acquisition is a signal: agent evaluation is now a platform concern. 🟡

**AGENTS.md** has crossed into formal infrastructure. The Agentic AI Foundation accepted it as a standard; 60,000+ repos use it. What started as informal practice (write a markdown file explaining how to work with your codebase) is now versioned, portable, and readable by both humans and agents. All major context systems (CLAUDE.md, SKILL.md, DECISIONS.md) converge on the same pattern. That convergence didn't happen by coordination. It happened because the format works. 🟡

## Semantic edit: the unsolved problem

The edit tool failure rate is real and documented. Agents express edits as text replacements, which break on whitespace drift, formatting changes, and minor refactors. Three approaches are trying to fix this at the AST level. 🟠

**aiCoder** (199 stars) lets the LLM generate a code snippet and merges it via AST rather than text diff. Handles the common case. JavaScript only. **Ki Editor** (891 stars) takes the more radical position. It's a structural code editor that operates on the AST directly, with no text layer. No agent API yet, which is the main constraint. **Relace Instant Apply** is API-only: semantic diffs at 4,300+ tokens/second with roughly 40% token reduction compared to full-file replacement. 🟠

None of these are production-ready for general agent use. tree-sitter is the foundational library all three build on. The problem is meaningful. Until it's solved, every edit-heavy agent workflow accumulates silent error risk at the margins. [see Experiments](./experiments.md)

## Pi ecosystem (March 2026)

The extension list has grown past what's easy to track. The additions with real adoption: 🟠

**pi-autoresearch** (2,288 stars): Karpathy's autoresearch workflow as a Pi extension. MAD confidence scoring. **pi-council**: multi-model ensemble: Claude, GPT, Gemini, Grok weigh in independently. **pi-codemap**: Aider's RepoMap (PageRank on dependency graphs) for Pi.

Q1 architecture shift: all built-in tools now work as custom tools in extensions. Extensions can replace any built-in. An upcoming API change separates business logic from UI. Agents calling agents, no terminal required.

The skills layer matters here. tech-lead, spec-to-plan, post-change-review, research-discover. These encode workflows as reusable programs, not prompts. That distinction grows as the programs get longer.

## Academic: what's worth reading

Without fine-tuning, agents can still accumulate experience: gather working examples, extract transferable patterns, apply them to new tasks. That's the ExpeL framework (Andrew Zhao, 2023). It's the theoretical basis for how instruction-based self-improvement works. 🟡

Verbal reflection with episodic memory raised HumanEval performance from 80% to 91% in the Reflexion paper (NeurIPS 2023). The mechanism: agents evaluate their own outputs, write a reflection, and store it as episodic memory for future runs. Simple. It works. 🟡

ADAS (ICLR 2025) automates the design of agentic systems: the agent designs its own pipeline structure. Gödel Agent (ICLR 2025) adds recursive self-modification with confidence gating. Both are research-stage, but they're where the "self-improving agent" concept gets formalized with rigor rather than just described. 🟡

DKB (January 2026) is directly relevant if you're building code navigation: deterministic AST graphs beat both vector RAG and LLM-generated knowledge graphs on code understanding tasks. The graph is static, reproducible, and cheap to compute. 🟡

On benchmarks: SWE-bench Verified 2026 has Verdent at 76.1%, Cursor/Codeium around 35–40%, Copilot at 12.3%. But the Lossfunk critique holds. Frontier models score 85–95% on standard benchmarks and fail on unseen languages. Build your own eval before trusting any universal number. 🟡

## People worth following

**Simon Willison** (@simonw). The most rigorous public reference on agentic engineering. 12+ chapters, ongoing. Testable positions: tests are free and mandatory, harness > model, hoard working examples. No hype. 🟠

**Mario Zechner** (@badlogicgames). Built Pi. 84 content posts per week during active periods. His critique: when agents self-praise, human review becomes theater. Coined "slop debt." Built Terminal Bench 2.0, then immediately criticized it as benchmaxxed. Builder and critic in one person. Rare. 🟠

**Andrej Karpathy** (@karpathy). Three steps ahead. Autoresearch: 700 commits in two days, −11% validation loss. Current thinking: memory should be trainable via RL, Git is wrong for collaborative autoresearch, Tab → Agent → Parallel → Teams isn't the end. 🟠

**Harrison Chase** (@hwchase17). LangChain. Most widely deployed agent orchestration. Worth following for the pragmatic take: what does production agent infrastructure actually require? OTel spans emitted natively. That tells you where expectations have landed. 🟠

## Open questions

**When does observability become overhead?** OTel GenAI SIG is maturing. Schemas still evolving. At what complexity does full tracing pay off versus `console.log`? No data. We haven't run the comparison.

**How far does AST edit generalize?** aiCoder: JavaScript. Ki Editor: editor-level. Relace: API-only. No solution covers the full agent edit workflow across languages. Clear problem. No clear solution.

**Does spec-kit's adoption mean fewer failures?** Amazon argues for SDD. But the evidence is directional. Spec-first teams fail differently, not necessarily less. A proper before/after on incident rates? Doesn't exist in public.
