# Landscape: who else is doing this

A ground-level map of April 2026, which tools have traction, which experiments blew up in production, and whose work shows up in our own thinking. Labels are honest. Nothing is here because it sounds important.

## Opus 4.7 and the silent repricing

The biggest release of the quarter arrived on April 16, 2026. Anthropic shipped Opus 4.7 with +10.9 points on SWE-bench Pro (64.3% vs 53.4% on 4.6), a new `xhigh` effort tier as the default for coding and agentic work, a file-system-level memory tool, and self-verification that runs before output commits. The pricing page said "unchanged" at $5 input / $25 output per million tokens.

The pricing was unchanged. The real cost wasn't. The new tokenizer maps the same English prose to about 1.25× more tokens, code and JSON to 1.35×. `budget_tokens` still accepts the old value but now silently routes through `task_budget`, which behaves differently — migration code that looked idempotent loses its cost ceiling without raising an error. Adaptive-only thinking means the model decides how much to spend, and the self-verification pass consumes another ~15% of output tokens before commit. One modelled workload ran $118K/month through 4.6 and $157.5K through 4.7 at identical behavior — a 33.5% increase on "unchanged" pricing. The prompt cache also resets cold for two to four weeks after migration, because the tokenizer change invalidates every prior cache entry.

The practitioner response has converged on route-by-workload rather than upgrade-wholesale. Cost-sensitive paths stay on 4.6 or Sonnet 4.6. Hard reasoning tasks move to 4.7 where the accuracy delta justifies the spend. Explicit effort control isn't available anymore — `xhigh` is what you get, and the budget mechanism is opaque. For teams that depended on `budget_tokens` as a cost cap, that's a breaking change dressed as a minor version.

This is the pattern to expect from now on: provider-side changes arrive as release events, not just runtime drift. Silent degradation — which used to mean "the same API returns worse answers" — now also means "the same call costs 35% more."

## Trends: what shifted in Q1 2026

Four things moved at once. Each would have been a year's conversation on its own.

Spec-driven development went from niche to default. spec-kit: 79,000 stars. Kiro: an entire IDE built around requirements → design → tasks. Tessl: skill registry with security scoring. Fowler documented the space in October 2025. GitHub's thesis is blunt: "code is the last-mile approach." Specs are the real artifact. The industry is catching up.

Agent-first IDEs replaced toolbars. The old model: editor with AI sidebar. The new model: agents are primary, editor is a view. Parallel agents, marketplace extensions, local↔cloud handoff. Baseline now, not differentiators.

OTel is becoming the observability layer. GenAI SIG defines schemas for LLM calls, agent steps, tool executions. Datadog, Honeycomb, New Relic support them natively. LangChain, CrewAI, AG2 emit OTel spans. Promptfoo added OTLP tracing. Agents are production services now. The tooling says so.

Self-improvement loops without model access. Three tools launched in one week (skill-loop, selfwrite, iterate). All focused on improving instructions, not weights. One practitioner auto-promoted 13 error patterns to behavioral rules in two weeks. Just logging. Just counting. The pattern is real. By April, the same approach shipped with a rule lifecycle: `adelaidasofia/claude-performance` reads session JSONLs, computes six effectiveness metrics, writes rules into CLAUDE.md when a metric falls below target, re-measures weekly, and **retires rules when the metric stabilises without them**. "A static rule is a wish. A measured rule is a system." The loop includes a delete step, which is the part most instruction-based self-improvement frameworks haven't bothered to ship.

## April 2026: the spec-driven research wave

Three papers landed in four weeks that turned scattered practitioner claims about specifications into measured effects.

The Specification Gap (ICN2 Barcelona, arxiv 2603.24284) ran 51 class-generation tasks across four spec detail levels, with both single-agent and multi-agent setups. Single-agent passes dropped from 89% at full spec to 56% at bare signatures. Multi-agent parallel decomposition held a structural gap of 25–39 additional points on top — 16 from coordination overhead, 11 from information asymmetry, independent and additive. An AST conflict detector at 97% precision moved the number by zero. Restoring the full spec to the merging agent recovered the 89% ceiling. The paper quantifies what practitioners have been saying for a year: specification is both the cause of coordination failure and the sufficient instrument of recovery.

Context Engineering (Swift North AI Lab, arxiv 2604.04258) formalised a five-role package: Authority, Exemplar, Constraint, Rubric, Metadata. 200 documented interactions across Claude, ChatGPT, Cowork, Codex; incomplete context triggered 72% of iteration cycles. A structured package against baseline cut iterations from 3.8 to 2.0 and raised first-pass acceptance from 32% to 55%. This is DO/DO NOT/GLOSSARY extended to explicit roles. The missing pieces from a minimal spec are usually Exemplar (a working code sample) and Rubric (explicit success criteria) — naming them makes the gap visible.

SLUMP (Purdue, arxiv 2603.17104) named a distinct failure: specifications that emerge during a session drift from the original problem statement as the conversation extends. Their mitigation — an external project-state layer called ProjectGuard that every worker checks against — recovered 90% of the faithfulness gap and cut severe failures from 72 to 49 on their benchmark. The persistent-artefact pattern (our `spec.md` plus `plan.md`) is the weak version; "every worker re-reads the spec" is the strong version. Most production pipelines stop at the weak one.

AGENTS.md got its own counter-intuitive result (arxiv 2602.11988): context files actively reduce SWE-bench success rates past 500 lines. Cliff drop, not gradual. The sweet spot sits at 200–300 actionable lines. The authors' framing: "not giving the model a mental model, giving it a compliance checklist." Practitioners have reported roughly 70% adherence to context file rules independently — the academic data lines up.

The Intent Gap (Microsoft Research + tianpan.co) put a number on the failure rate: intent misalignment accounts for ~32% of dissatisfactory LLM responses in production, the largest single category, ahead of hallucination and refusal combined. Every user input carries four layers (immediate text, final goal, unstated background constraints, user autonomy). Production LLMs are fluent at the first, mediocre at the second, violate the third regularly. RLHF optimises for helpfulness and productivity — the training signal points at the gap, not away from it. The practical implication: restate-back during planning isn't politeness; it's the cheapest available counter-measure.

## Harrison Chase's three-layer framing

LangChain's Harrison Chase published a "continual learning for AI agents" framework that names three layers where improvement happens, orthogonal to our LMP framing. Model (weights, catastrophic forgetting), Harness (code plus always-present instructions — Claude Code, Pi), Context (CLAUDE.md, skills, mcp.json). Most discussions only address the model layer; harness and context are where practitioners actually compound returns. Chase explicitly maps OpenClaw as "Pi plus some other scaffolding" — harness layer, not a model fork.

Two related points from the same post. First, hot-path versus offline memory updates — hot-path runs while the agent is on the core task, offline runs as a batch job over recent traces. CC's `autoDream` is offline; `extractMemories` is mid-session. Our KB lesson extraction is offline-only; hot-path during a session is a gap. Second, traces are the shared substrate for all three learning layers — model training, harness improvement, context updates. LangSmith is LangChain's positioning play here. Our equivalent is Pi session JSONLs plus `trajectory.py`. The investment in trace analysis isn't tactical; it's the substrate everything else rides on.

## GitLab's five autonomy levels

GitLab published a practitioner playbook that names a ladder instead of a pattern.

L1 is autocomplete. L2 is pair-programming: the human designs and reviews, the agent writes. L3 is conductor: a tight feedback loop where the human stays in every decision. L4 is orchestrator: multiple async agents, human sets direction and validates outcomes. L5 is harness: the human sets architecture and quality bar, the agent does everything between.

The warning matters more than the ladder: "skipping to L4 or L5 without the right infrastructure produces unreliable output and amplifies technical debt." Amazon's Sev-1 pattern reads as exactly that failure mode — mandated L4/L5 deployment at 21,000-agent scale with L2-level verification infrastructure. GitLab's five-principle framing is the complementary piece: failing test before every feature, fix the environment not the prompt, constraints are multipliers, repo is single source of truth, ask the agent to challenge you. The environment fix is the one that compounds. Prompt fixes persist until the context compacts. A CI gate persists across every future session.

## Seven anti-patterns from production telemetry

TechDebt.guru catalogued what happens when copilot usage meets production code without the infrastructure above. Seven patterns, measured across GitClear and CodeRabbit data:

Accept-and-Forget — suggestions accepted without line-by-line review. Tab-Tab-Tab Syndrome — rapid acceptance, 40% higher defect density. Context Blindness — ignores project architecture. Dependency Sprawl. Test Scaffolding Decay. Documentation Displacement. Style Drift. GitClear's headline figure: AI-generated code has a 55% higher revert rate within two weeks. CodeRabbit's analysis across 470 repositories: AI code produces 1.7× more bugs than human, 75% more logic errors, 8× more I/O performance bugs, 1.5–2× more security bugs. The pattern is consistent across measurement methodology.

The takeaway isn't "agents are worse." It's that the 40% defect bump and the 55% revert rate are the default when verification infrastructure doesn't scale with generation volume. Teams that invested in the verification layer before scaling generation show the opposite pattern.

## Cursor 3: agent-first IDE

Cursor rebuilt from scratch. Not an upgrade. A rewrite.

The previous version was VS Code with Claude bolted on. Version 3 treats agents as the primary primitive. The editor is a view into what agents are doing, not the other way around. Parallel agents in the sidebar. Multi-workspace. Cloud/local handoff. A marketplace for MCPs, skills, and subagents.

Default model: Composer 2 (Kimi K2.5), 73.7 on SWE-bench multilingual, 61.3 on CursorBench. "Fleets of agents work autonomously to ship improvements". Their words, not ours. The gap between "AI-assisted" and "agent-led" is now visible in shipping product.

## Amazon vibe coding failure: a case study

Four Sev-1 incidents in 90 days. 21,000 agents at 80% weekly usage. $2B savings target. One outage: 6 hours, ~6.3 million lost orders.

The failure was structural. Amazon laid off ~30,000 employees and scaled AI code generation simultaneously. Less review capacity, more volume. Matt Garman drew the line: "vibe coding" (blind acceptance) vs "augmented coding" (AI plus oversight). Amazon built the creation layer. The verification layer stayed the same size.

SDD is partly a response to this. Spec-driven workflows force humans to articulate intent before code runs. The question isn't whether AI should write code. It's whether anyone in the loop still understands what the code is supposed to do. Amazon answered that. The answer was no.

## Key tools worth knowing

**spec-kit** is GitHub's 5-command SDD workflow: constitution → specify → plan → tasks → implement. Twenty-plus agents, 79,000 stars. The open-source complement to Kiro.

**DSPy** (Stanford, 33,000 stars) shifts the abstraction from prompt strings to program structure. The optimizer (BootstrapFewShot, MIPROv2, GEPA) automatically improves prompts against a metric you define. It's the closest open-source implementation of a closed-loop Layer 3 system. Programming agents rather than prompting them.

**Promptfoo** was acquired by OpenAI in March 2026 for $86M. 350,000 developers, 130,000 MAU, 25% Fortune 500 adoption before the acquisition. Trajectory assertions, red teaming, 3-layer testing, now with OTLP tracing. The acquisition is a signal: agent evaluation is now a platform concern.

**AGENTS.md** has crossed into formal infrastructure. The Agentic AI Foundation accepted it as a standard; 60,000+ repos use it. What started as informal practice (write a markdown file explaining how to work with your codebase) is now versioned, portable, and readable by both humans and agents. All major context systems (CLAUDE.md, SKILL.md, DECISIONS.md) converge on the same pattern. That convergence didn't happen by coordination. It happened because the format works.

## Semantic edit: the unsolved problem

The edit tool failure rate is real and documented. Agents express edits as text replacements, which break on whitespace drift, formatting changes, and minor refactors. Three approaches are trying to fix this at the AST level.

**aiCoder** (199 stars) lets the LLM generate a code snippet and merges it via AST rather than text diff. Handles the common case. JavaScript only. **Ki Editor** (891 stars) takes the more radical position. It's a structural code editor that operates on the AST directly, with no text layer. No agent API yet, which is the main constraint. **Relace Instant Apply** is API-only: semantic diffs at 4,300+ tokens/second with roughly 40% token reduction compared to full-file replacement.

None of these are production-ready for general agent use. tree-sitter is the foundational library all three build on. The problem is meaningful. Until it's solved, every edit-heavy agent workflow accumulates silent error risk at the margins. [see Experiments](./experiments.md)

## Pi ecosystem

The extension list has grown past what's easy to track. The additions with real adoption:

**pi-autoresearch** (2,288 stars): Karpathy's autoresearch workflow as a Pi extension. MAD confidence scoring. **pi-council**: multi-model ensemble: Claude, GPT, Gemini, Grok weigh in independently. **pi-codemap**: Aider's RepoMap (PageRank on dependency graphs) for Pi.

Q1 architecture shift: all built-in tools now work as custom tools in extensions. Extensions can replace any built-in. An upcoming API change separates business logic from UI. Agents calling agents, no terminal required.

The skills layer matters here. tech-lead, spec-to-plan, post-change-review, research-discover. These encode workflows as reusable programs, not prompts. That distinction grows as the programs get longer.

## Academic: what's worth reading

Without fine-tuning, agents can still accumulate experience: gather working examples, extract transferable patterns, apply them to new tasks. That's the ExpeL framework. It's the theoretical basis for how instruction-based self-improvement works.

Verbal reflection with episodic memory raised HumanEval performance from 80% to 91% in the Reflexion paper. The mechanism: agents evaluate their own outputs, write a reflection, and store it as episodic memory for future runs. Simple. It works.

ADAS automates the design of agentic systems: the agent designs its own pipeline structure. Gödel Agent adds recursive self-modification with confidence gating. Both are research-stage, but they're where the "self-improving agent" concept gets formalized with rigor rather than just described.

DKB is directly relevant if you're building code navigation: deterministic AST graphs beat both vector RAG and LLM-generated knowledge graphs on code understanding tasks. The graph is static, reproducible, and cheap to compute.

On benchmarks: SWE-bench Verified 2026 has Verdent at 76.1%, Cursor/Codeium around 35–40%, Copilot at 12.3%. But the Lossfunk critique holds. Frontier models score 85–95% on standard benchmarks and fail on unseen languages. Build your own eval before trusting any universal number.

## People worth following

**Simon Willison** (@simonw). The most rigorous public reference on agentic engineering. 12+ chapters, ongoing. Testable positions: tests are free and mandatory, harness > model, hoard working examples. No hype.

**Mario Zechner** (@badlogicgames). Built Pi. 84 content posts per week during active periods. His critique: when agents self-praise, human review becomes theater. Coined "slop debt." Built Terminal Bench 2.0, then immediately criticized it as benchmaxxed. Builder and critic in one person. Rare.

**Andrej Karpathy** (@karpathy). Three steps ahead. Autoresearch: 700 commits in two days, −11% validation loss. Current thinking: memory should be trainable via RL, Git is wrong for collaborative autoresearch, Tab → Agent → Parallel → Teams isn't the end.

**Harrison Chase** (@hwchase17). LangChain. Most widely deployed agent orchestration. Worth following for the pragmatic take: what does production agent infrastructure actually require? OTel spans emitted natively. His April 2026 continual-learning post put a practical framework on the model/harness/context split that most discussions collapse into "the model."

**Armin Ronacher** (@mitsuhiko). Advocate for `lat.md` as knowledge-graph-in-markdown. Shipped multi-edit tooling for Pi. Direct critic of agent anti-patterns: "agents are hard to resist, build shit you regret." Builds the tools and documents the failure modes in the same feed.

## Open questions

**When does observability become overhead?** OTel GenAI SIG is maturing. Schemas still evolving. At what complexity does full tracing pay off versus `console.log`? No data. We haven't run the comparison.

**How far does AST edit generalize?** aiCoder: JavaScript. Ki Editor: editor-level. Relace: API-only. No solution covers the full agent edit workflow across languages. Clear problem. No clear solution.

**Does spec-kit's adoption mean fewer failures?** Amazon argues for SDD. But the evidence is directional. Spec-first teams fail differently, not necessarily less. A proper before/after on incident rates? Doesn't exist in public.

**Does pipeline alpha shrink monotonically as models improve?** NCSU's 9,374-trajectory study showed framework effects diminishing with stronger LLMs. If the trend is monotonic, most pipeline infrastructure becomes legacy overhead on a five-year horizon — baked into the model instead. If the trend is non-monotonic (structure still buys something at long horizons or specific task types), the investment compounds. We have no longitudinal data across model generations yet.
