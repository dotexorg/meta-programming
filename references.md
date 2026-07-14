# References

This page collects tools, sources, and experiment details behind the [Meta-Programming](index.md) documentation. Tools come first: runnable artifacts worth installing, forking, or reading. Then sources backing the claims in narrative pages, then our own experiments.

## Evidence Levels

| Marker | Level | Meaning |
|--------|-------|---------|
| 🟢 | Proven | Our experiment, our data, measured result |
| 🟡 | Trusted source | Anthropic, Microsoft Research, peer-reviewed — we read the primary source |
| 🟠 | Community reports | Widely observed, not independently verified by us |
| 🔴 | Unverified | Heard, not checked |
| ⚪ | Opinion | Our synthesis, reasoned but not proven |

---

## Tools

Tools are organised by pipeline layer (Spec → Edit → Navigate → Verify → Recover → Self-improve). The list is narrow rather than exhaustive: each entry is something we observed shipping, evaluated as a research signal, or use ourselves. License/access labels appear when they affect adoption decisions.

### Spec / Form

- **[AGENTS.md](https://agents.md)** — cross-tool agent context standard. 60,000+ repositories, Linux Foundation / Agentic AI Foundation governed, co-signed by Anthropic, Google, Microsoft, Cursor. The lowest-friction form for cross-session agent contracts. 🟡
- **[spec-kit](https://github.com/github/spec-kit)** — GitHub's five-command SDD workflow: constitution → specify → plan → tasks → implement. 79K stars, supports 20+ agent runtimes. Maps to Fowler's spec-anchored maturity level. 🟡
- **[Kiro](https://kiro.dev)** — Amazon IDE built around requirements → design → tasks. Specs live in project root and evolve with the codebase. Spec-first maturity, beta access. 🟡
- **[Tessl](https://tessl.io)** — spec-as-source aspiration: only the specification is human-edited, code is a generated artifact. Earliest commercial example of Fowler's level-three maturity. 🟠
- **[Pydantic AI](https://github.com/pydantic/pydantic-ai)** — distilled 4,668 PR comments into 150 AGENTS.md rules. Concrete example of compiling engineering taste into agent instructions. 🟡

### Edit / Apply

- **[Cursor](https://cursor.sh)** — apply-model architecture (planner emits intent, applier executes the diff). Plan Mode (Shift+Tab) for research → questions → plan → approve → build. Commercial, free tier. 🟡
- **[Aider](https://aider.chat)** — architect mode separates planning from edit application. Pioneered the architect/editor split now reproduced across the apply-model layer. Open source. 🟠
- **[Morph Fast Apply](https://morphllm.com)** — semantic diff format optimised for AI output. ~4300 tok/s, cuts apply-step token usage ~40% on Claude 3.5/3.7. 🟠
- **[Relace Instant Apply](https://relace.ai)** — alternative apply model in the same pattern as Morph. Inspired by Cursor's apply model. 🟠
- **[Augment Code](https://augmentcode.com)** — single-writer rule for hotspot files, sequential merge strategy at scale. Commercial. 🟡

### Navigate / Discover

- **[agent-lsp](https://github.com/blackwell-systems/agent-lsp)** — Go MCP server bridging Language Server Protocol into the agent tool surface. 56 tools, 30 CI-verified language servers. Reproducible benchmark across five codebases (15K–319K LOC) finds 92–99% of grep matches on symbol references are false positives; structured navigation runs 5–34× more token-efficient than grep-and-read. Ships speculative execution and phase enforcement primitives. 🟡

### Verify / Gate

- **[Promptfoo](https://github.com/promptfoo/promptfoo)** — trajectory assertions: `tool-used`, `tool-args-match`, `tool-sequence`, `step-count`, `goal-success`. 350K developers. Acquired by OpenAI March 2026 for $86M. 🟡
- **[OpenTelemetry GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)** — semantic conventions for agent observability: `gen_ai.chat`, `agent.invoke`, `tool.execute`. Native support in Datadog, Honeycomb, New Relic. 🟡

### Recovery loops

- **[lopi](https://github.com/konjoai/lopi)** — Rust + tokio Ralph-loop orchestrator. Plan → Implement → Test → Score → Fix → Retry with `git reset --hard` per failed attempt. Hard turn limits, diff-scope check, last-error injection, model routing escalation only after failure. Quality gate `LESSON_QUALITY_GATE = 0.6` skips lesson write below threshold. Single-imperative lesson form (≤200 chars). Most architecturally complete of the four shipped Ralph-loop reproductions. 🟡
- **[ralph-claude-code](https://github.com/gacabartosz/ralph-claude-code)** — Claude Code port of the recovery layer alone, with cost caps, worktree isolation, and MCP audit mode. Third independent shipped reproduction of the Ralph-loop pattern. 🟠
- **Geoffrey Huntley's Ralph blog** — named originator of the loop pattern. Reference reading: [Huntley personal blog](https://ghuntley.com) and [ai-assisted-software-development.com](https://ai-assisted-software-development.com). 🟠

### Self-improve / Memory

- **[DSPy](https://github.com/stanfordnlp/dspy)** — prompts as learnable parameters. BootstrapFewShot and MIPROv2 search language space automatically. 33K stars. 🟡
- **[CORE](https://github.com/RedPlanetHQ/core)** — temporal knowledge graph memory. Episodes → Entities → Statements with hybrid search (vector + BM25 + graph traversal). 88.24% on LoCoMo benchmark. CC plugin via SessionStart and Stop hooks. 3K stars, self-hostable. 🟠
- **[claude-performance](https://github.com/adelaidasofia/claude-performance)** — measurement-driven CLAUDE.md. Reads session JSONLs, computes six effectiveness metrics, writes behavioural rules when a metric falls below target, retires rules when the metric stabilises. Concrete implementation of the add → measure → retire lifecycle. 🟠
- **Homunculus plugin** (Reddit r/ClaudeAI) — observes user patterns, auto-generates skills, hooks, and commands when repetitive behaviour is detected. Probabilistic skills (50–80% fire rate), deterministic commands. Per-project state in `.claude/homunculus/`. 🟠

### Multi-layer platforms

- **[Claude Code](https://docs.anthropic.com/claude/docs/claude-code)** — Anthropic's terminal coding agent. Hooks-as-gates (PreToolUse / PostToolUse), skills, AGENTS.md, fork mechanics for cache reuse. 67% blind-quality win rate against Codex CLI in March 2026 benchmarks. 🟡
- **[Codex CLI](https://github.com/openai/codex)** — OpenAI's terminal coding agent. 68K stars, MIT-licensed. `npm i -g @openai/codex`. 🟡
- **[Cubic](https://cubic.dev)** — commercial review-loop platform. Closed-source, public benchmarks. 🟠

Each project above places a different bet on which pipeline layer matters most. The layer split is the legend: a tool's category tells you what assumption it made about where AI agents actually fail. Convergence-by-pattern is real — recovery loops, hooks-as-gates, AGENTS.md adoption — but the optimal stack for a given task profile remains an empirical question per team. ⚪

---

## Our Experiments

| # | Description | Key Finding | Pages |
|---|-------------|-------------|-------|
| 1 | EventBus refactor A/B (5 variants, 509 TS files, DDD, Cloudflare Workers) | Process beats information: Scout→Spec→Worker→Review pipeline ($8.45) outperformed raw context injection ($2.84 fail, $14.30 pass). Code maps hurt: $9.99 fail vs $6.63 pass without map. | [pipeline](pipeline.md), [principles](principles.md) |
| 2 | Telegram bot without spec (support reply feature) | Two consecutive deterministic failures without spec. First corrupted existing handler, second missed intent entirely. | [specification](specification.md), [principles](principles.md) |
| 3 | 24-file type refactor | 253 tests, zero regressions, $5.50 API cost. Review required three iterations (both failure types were missing pre-flight checks). 106-turn task didn't contaminate subsequent tasks due to context reset. | [index](index.md), [verification](verification.md), [pipeline](pipeline.md) |
| 4 | Spec review with barrel re-export | Agent proposed barrel re-export during spec review, recognized it was unnecessary when asked to explain. Spec review is cheaper than code review. | [specification](specification.md), [principles](principles.md) |
| 5 | Pipeline variant comparison | Structured pipeline: correct on first attempt, $8.45. Raw prompt: wrong, $2.84. Sequential with context: correct, $14.30. With code map: wrong, $9.99. | [pipeline](pipeline.md) |
| 6 | KB A/B test (generic vs structured) | Generic Sonnet said "give it a code map." KB-loaded agent flagged exploration-vs-exploitation paradox with prior session evidence before writing code. | [index](index.md), [principles](principles.md) |
| 7 | Edit tool investigation | Persistent error pattern traced to our own extension, not the platform. Post-fix benchmark: 7.1% errors, 1.1% data loss. | [index](index.md) |
| 8 | Model evaluation (thinking levels) | Thinking level acts as compliance-to-conviction dial. Soft sycophancy identified: agent says no while providing implementation. | [index](index.md) |
| 9 | Opus degradation incident (April 2026) | Read:Edit ratio dropped from 6.6 to 2.0, thinking depth fell 67%, costs spiked 80×. Three-day silent degradation with no API-side signal. | [verification](verification.md), [principles](principles.md) |
| 10 | Opus 4.7 release analysis (April 16, 2026) | Tokenizer inflation 1.25–1.35× tokens per request, `budget_tokens` silently rerouted to `task_budget`, `xhigh` as new default, self-verification +~15% output tokens. Modelled workload $118K → $157.5K (+33.5%) at unchanged nominal pricing. | [landscape](landscape.md), [verification](verification.md) |

---

## External Sources

### Large-Scale Studies

1. **LinearB (2026).** "The Real Impact of AI on Developer Productivity." 8.1M pull requests, 4,800 teams, 42 countries. AI-generated code: 1.7× more review revisions, 4.6× longer review wait, 32.7% acceptance rate (vs 84.4% human). Developers feel 20% faster; tasks take 19% longer end-to-end. 🟡
   - Referenced in: [index](index.md), [verification](verification.md), [principles](principles.md)

2. **Meta-Harness (Lee, Nair, Zhang, Lee, Khattab, Finn — Stanford + MIT, arxiv 2603.28052, March 2026).** End-to-end optimization of model harnesses. Agentic proposer searches harness code via filesystem: +7.7 points on online text classification with 4× fewer context tokens, +4.7 points on retrieval-augmented math (200 IMO-level), 76.4% on TerminalBench-2 (top auto-optimized system, vs a Claude Code baseline of 58.0). Correction: the widely-quoted "6× gap" is a figure the paper cites from SWE-bench Mobile, not a Meta-Harness measurement — do not attribute it to this study. 🟡
   - Referenced in: [index](index.md), [principles](principles.md)

3. **ETH Zurich / LogicStar (arxiv 2602.11988, Feb 2026), read firsthand.** 138 real-world tasks (CTXBENCH) across 3 models. Auto-generated context files: no significant success change (p=.87/.37) with +20–23% cost. Human-written boundaries: +2.4%, not statistically significant. Correction: earlier editions cited "−3% / +4%" — neither figure holds. Same paper as 17h below (dedup pending). 🟡
   - Referenced in: [specification](specification.md), [pipeline](pipeline.md), [principles](principles.md)

4. **Sonar (2026).** Survey of 1,000+ developers. Only 48% verify AI output before shipping. 🟡
   - Referenced in: [index](index.md)

5. **Microsoft Copilot Study (2026).** 10-month study, 878 pull requests. "The bottleneck moved from typing speed to knowledge, judgment, and ability to articulate tasks." 🟡
   - Referenced in: [index](index.md)

6. **BSWEN (2026).** 133 cycles, 42 development phases, four models in strict isolation. GPT caught Python security issues Claude missed. Claude caught architectural violations GPT normalized. Each model had different blind spots. 🟡
   - Referenced in: [verification](verification.md), [principles](principles.md)

7. **Bamberg/Heidelberg (2026).** Systematic analysis of 2,926 repositories across Claude Code, GitHub Copilot, Cursor, Gemini CLI, Pydantic AI. Converging on identical patterns independently. 🟡
   - Referenced in: [index](index.md)

### Research Papers

8. **Tsinghua NLAH (March 2026).** "Natural-Language Agent Harnesses." Harness behavior externalized as "a portable executable artifact in editable natural language." 🟡
   - Referenced in: [index](index.md)

9. **Microsoft Research RiSE (March 2026).** Lahiri et al. "Intent Formalization" named as a grand challenge for 2026. Intent gap: the semantic distance between what a developer means and what the system does. 🟡
   - Referenced in: [index](index.md), [specification](specification.md)

10. **ERL — Experiential Reflective Learning (Allard, Teinturier, Xing, Viaud, arxiv 2603.24639, March 2026).** Agents with heuristics extracted from prior trajectories outperformed ReAct baselines by +7.8% on the Gaia2 benchmark. Two-component framework: heuristic generation from task experience + retrieval-augmented execution for new tasks. "Heuristics provide more transferable abstractions than few-shot prompting." 🟡
    - Referenced in: [index](index.md)

11. **ExpeL (Andrew Zhao et al., 2023).** Experience Learning: three-stage self-improvement (act → reflect → extract). On HotpotQA and ALFWorld, ExpeL agents improve with each batch of trajectories. 🟡
    - Referenced in: [self-improvement](self-improvement.md), [landscape](landscape.md)

12. **Chroma 'Context Rot' (July 2025).** 18 frontier models tested (GPT-4.1, Claude 4, Gemini 2.5, Qwen3, others). Universal degradation with input length: 20-50% accuracy drop between 10K and 100K input tokens (NIAH low-similarity), >30% accuracy loss in mid-window positions across all 18 models. Coined "Context Rot" as continuous decline, not overflow. 🟡
    - Referenced in: [context-engineering](context-engineering.md), [principles](principles.md)

12b. **Paulsen MECW (arxiv Oct 2025, pub Jan 2026).** "Context Is What You Need: The Maximum Effective Context Window for Real World Limits of LLMs." Separate paper from Chroma; coined the term MECW. Effective context is task-dependent: simple retrieval tolerates ~3000-5000 tokens, complex operations (sort, summarize) collapse at 400-1200 tokens, some top models fail at as few as ~100 tokens on specific tasks. Effective window can be reduced "as much as 99%" of advertised MCW on worst-case structured tasks. 🟡
    - Referenced in: [context-engineering](context-engineering.md)

13. **Reflexion (NeurIPS 2023).** Verbal reflection with episodic memory raised HumanEval from 80% to 91%. 🟡
    - Referenced in: [landscape](landscape.md)

14. **ADAS (ICLR 2025).** Automated Design of Agentic Systems. The agent designs its own pipeline structure. 🟡
    - Referenced in: [landscape](landscape.md)

15. **Gödel Agent (ICLR 2025).** Recursive self-modification via confidence-based logic. 🟡
    - Referenced in: [landscape](landscape.md)

16. **DKB (January 2026).** Deterministic Knowledge Bases. AST graphs beat vector RAG and LLM-generated knowledge graphs for code navigation. 🟡
    - Referenced in: [landscape](landscape.md)

17. **AMBIG-SWE (ICLR 2026).** Benchmark for ambiguity detection in software engineering tasks. 🟡
    - Referenced in: [specification](specification.md)

17b. **Specification Gap (Chacón Sartori, ICN2 Barcelona, arxiv 2603.24284, March 2026).** 51 class-generation tasks across four spec detail levels, single and multi-agent. Single-agent: 89% → 56% as spec details are removed. Multi-agent: 58% → 25%. 16pp coordination cost plus 11pp information asymmetry, additive. AST conflict detector at 97% precision: Δ = 0pp. Restoring full spec recovers 89% ceiling. 🟡
    - Referenced in: [specification](specification.md), [pipeline](pipeline.md), [landscape](landscape.md)

17c. **Context Engineering (Calboreanu, Swift North AI Lab, arxiv 2604.04258, April 2026).** Five-role context package: Authority, Exemplar, Constraint, Rubric, Metadata. 200 documented interactions across four tools. Incomplete context triggered 72% of iteration cycles. Structured package: iterations 3.8 → 2.0, first-pass acceptance 32% → 55%. 🟡
    - Referenced in: [specification](specification.md), [context-engineering](context-engineering.md), [landscape](landscape.md)

17d. **SLUMP — Specification Loss Under eMergent sPecification (Purdue, arxiv 2603.17104, March 2026).** Specifications that emerge during a session drift from the original problem statement as conversation extends. ProjectGuard external state tracker recovers 90% of the faithfulness gap, cuts severe failures from 72 to 49 on benchmark. 🟡
    - Referenced in: [specification](specification.md), [landscape](landscape.md), [self-improvement](self-improvement.md)

17e. **Intent Gap (tianpan.co, April 10, 2026).** Intent misalignment accounts for ~32% of dissatisfactory LLM responses in production — largest single category. Four-layer user input model: immediate text, final goal, background desiderata, autonomy. Salesforce production: 58% single-turn success, 35% multi-turn. 67% resolution rate even after user correction. 🟡🟠
    - Referenced in: [specification](specification.md), [landscape](landscape.md)

17f. **Behavioral Drivers (Mehtiyev & Assunção, NCSU, arxiv 2604.02547, April 2026).** 9,374 agent trajectories across 19 agents. Trajectory structure discriminates success: "gather context before editing, invest in validation" is agent-determined, not task-adaptive. Framework effect shrinks with each generation of base model. 🟡
    - Referenced in: [pipeline](pipeline.md), [principles](principles.md), [landscape](landscape.md)

17g. **Cognitive Companion (Khan & Khan, IBM Dublin, arxiv 2604.13759, April 2026).** Four cognitive states: ON_TRACK, LOOPING, DRIFTING, STUCK. Two detector architectures: LLM-based companion (periodic structured prompt, −52–62% repetition, 11% overhead, API-accessible) and probe-based (linear classifier on hidden states layer 28, AUROC 0.84, requires open weights). 🟡
    - Referenced in: [verification](verification.md), [self-improvement](self-improvement.md)

17h. **AGENTS.md paper — ETH/LogicStar (arxiv 2602.11988), read firsthand 2026-07-12.** On CTXBENCH (138 issues / 12 repos) plus SWE-bench: LLM-generated context files show no significant success change (p=.87/.37) while adding +20–23% cost; §B finds file length has no effect on outcomes. Correction: earlier editions of this reference cited a "500-line cliff / 200–300 sweet spot / compliance-checklist quote" from this id — the paper contains none of them. 🟡
    - Referenced in: [specification](specification.md), [context-engineering](context-engineering.md), [landscape](landscape.md)

17i. **Expectation-Realisation Gap (Lobentanzer et al., arxiv 2602.20292, February 2026).** Extension of METR study. 16 developers expected +24% productivity from AI tools, measured −19%. 43-point calibration error. 🟡
    - Referenced in: [verification](verification.md)

17j. **WebXSkill (Microsoft + UNC, arxiv 2604.13318, April 2026).** Skill defined as parameterized action + natural-language guidance. +9.8 / +12.9 points on WebArena / WebVoyager against baseline. Concrete instance of Layer 2 intent form selection. 🟡
    - Referenced in: [index](index.md), [self-improvement](self-improvement.md)

18. **ACE Framework.** Agentic Context Engineering. Memory scoring: each unit carries a score that updates on use. Quality saturates at ~7 governed memories per entity across 500 adversarial queries. 🟡
    - Referenced in: [self-improvement](self-improvement.md)

19. **Pavlyshyn (Jan 2026).** History of constrained natural language in programming: COBOL, SQL, Simula. 60-year progression. 🟡
    - Referenced in: [specification](specification.md)

### Vendor Documentation & Posts

20. **Anthropic — "Building Agents with Skills."** Skills as zero-cost-until-invoked context units. Self-evaluation bias documented explicitly. Claude Code architecture: tiered context loading, compaction, coordinator mode. 🟡
    - Referenced in: [index](index.md), [context-engineering](context-engineering.md), [verification](verification.md), [principles](principles.md)

29. **Simon Willison (@simonw).** Rigorous public reference on agentic engineering. Tests are free and mandatory. Agents follow existing code patterns. 🟡🟠
    - Referenced in: [context-engineering](context-engineering.md), [landscape](landscape.md)

30. **Martin Fowler.** Spec progression: spec-first → spec-anchored → spec-as-source. Maturity curve mapping. 🟡
    - Referenced in: [index](index.md)

30b. **Harrison Chase — "Continual learning for AI agents" (LangChain blog, April 2026).** Three-layer framework: Model (weights), Harness (code + always-present instructions), Context (CLAUDE.md, skills, mcp.json). Hot-path vs offline memory update modes. Traces as shared substrate across all three layers. OpenClaw explicitly mapped as "Pi plus scaffolding" = harness layer. 🟡
    - Referenced in: [landscape](landscape.md), [self-improvement](self-improvement.md), [index](index.md)

30c. **GitLab AI-Assisted Development Playbook.** Five autonomy levels: L1 Baseline (autocomplete), L2 Pair, L3 Conductor, L4 Orchestrator, L5 Harness. Five principles: failing test before every feature, fix the environment not the prompt, constraints are multipliers, repo is single source of truth, ask the agent to challenge you. Warning: skipping to L4/L5 without infrastructure amplifies technical debt. 🟡
    - Referenced in: [landscape](landscape.md), [playbook](playbook.md), [principles](principles.md)

30d. **TechDebt.guru — 7 Copilot Anti-Patterns.** Accept-and-Forget, Tab-Tab-Tab Syndrome (40% higher defect density), Context Blindness, Dependency Sprawl, Test Scaffolding Decay, Documentation Displacement, Style Drift. GitClear: 55% higher revert rate within two weeks for AI code. CodeRabbit (470 repos): 1.7× more bugs, 75% more logic errors, 8× more I/O performance bugs. 🟡🟠
    - Referenced in: [landscape](landscape.md), [verification](verification.md)

30e. **Cursor Official Best Practices (April 2026).** Plan Mode (Shift+Tab) for research → questions → plan → approve → build. "Start over from a plan" preferred to mid-agent fixing. New conversation when context polluted. Save plans to `.cursor/plans/` for team docs and resumption. 🟡
    - Referenced in: [landscape](landscape.md), [pipeline](pipeline.md)

30f. **Opus 4.7 Migration Analysis (Anthropic platform docs + @badlogicgames + ravoid.com + dev.to).** Tokenizer inflation 1.0–1.35×, `budget_tokens` silently rerouted, `xhigh` as new default, self-verification +~15% output. Modelled workload: $118K/mo → $157.5K/mo (+33.5%) on identical behavior. Prompt cache cold for 2–4 weeks post-migration. 🟡
    - Referenced in: [landscape](landscape.md), [verification](verification.md)

### Community Reports

31. **Amazon Kiro deployment (March 2026).** 21,000 agents, 80% weekly usage mandate. 4 Sev-1 incidents in 90 days. March 5 outage: 6 hours, ~6.3M lost orders (99% US order drop). Internal Treadwell email cites "Gen-AI assisted changes" with "high blast radius." 90-day safety reset across 335 Tier-1 systems, mandatory two-person review. ~30,000 layoffs concurrent with AI scaling. Primary reporting: Financial Times (internal docs). Summary: [The Register, 10 Mar 2026](https://www.theregister.com/2026/03/10/amazon_ai_coding_outages/). 🟠
    - Referenced in: [index](index.md), [verification](verification.md), [principles](principles.md), [landscape](landscape.md)

32. **Self-improvement tools (March 2026).** Three independent projects (skill-loop, selfwrite, iterate) shipped in the same week without coordination. All focused on instruction improvement, not weight modification. 🟠
    - Referenced in: [self-improvement](self-improvement.md), [landscape](landscape.md)

33. **Edit tool failure rate.** Agents express edits as text replacements, which break on whitespace drift, formatting changes, and multi-cursor ambiguity. Documented across multiple tools. 🟠
    - Referenced in: [landscape](landscape.md)

34. **Spec sizing (900-1600 tokens).** Community-reported sweet spot for structured quick-dev specs. Below 900: ambiguity risk. Above 1600: tail instruction degradation. 🟠
    - Referenced in: [specification](specification.md)

35. **Rory Teehan.** Structured error logging: what happened, why, what should have happened. 🟡
    - Referenced in: [self-improvement](self-improvement.md)

35d. **u/thurn2 (Reddit r/ClaudeCode).** "Agent teams = expensive subagents with better marketing." Community consensus across multiple threads: communication overhead overwhelms team leader's context, idle notifications consume context, no proven benefit over simple subagent spawning for current implementations. 🟠
    - Referenced in: [pipeline](pipeline.md)

### People to Follow

36. **Andrej Karpathy** (@karpathy). Autoresearch: 700 commits in two days, −11% validation loss. Memory should be tree-structured, not flat. 🟠
37. **Mario Zechner** (@badlogicgames). Built Pi. When agents self-praise, human review becomes the bottleneck. 🟠
38. **Harrison Chase** (@hwchase17). LangChain. Model/Harness/Context three-layer continual learning framework. What does production agent orchestration actually look like at scale. 🟡
39. **Armin Ronacher** (@mitsuhiko). Advocate for `lat.md` (knowledge graph in markdown). Shipped multi-edit tooling for Pi. Direct critic of agent anti-patterns. 🟠

---

*This reference list is maintained alongside the documentation. Sources marked with evidence levels as used in the main text.*
