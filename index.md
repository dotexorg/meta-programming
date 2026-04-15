<!-- NAV
- [Meta-Programming](index.md)
- [Specification](specification.md)
- [Context Engineering](context-engineering.md)
- [Pipeline](pipeline.md)
- [Verification](verification.md)
- [Self-Improvement](self-improvement.md)
- [Principles](principles.md)
- [Landscape](landscape.md)
-->

# Linguistic Meta-Programming

How you structure an agent's memory, specs, and feedback loops matters more than which model you pick. This site documents why — and what to do about it.

**Linguistic Meta-Programming (LMP)** is a framework for improving coding agents through linguistic feedback — specs, reviews, lessons, rules — without touching model weights. No fine-tuning. The agent gets better because it accumulates structured knowledge about what works and what doesn't, session after session. We built and tested this on production codebases. This site is what we found.

---

## The progression: from vibe coding to meta-programming

Three phases. Most teams are stuck in the first.

**Phase 1: Vibe coding.** Describe what you want, get code. Works for throwaway scripts. Falls apart in production — AI-written code creates 1.7× more review revisions than human code (LinearB, 8.1M PRs, 2026). 🟡 That's assuming someone reviews it at all. Half don't — only 48% of developers verify AI output before shipping (Sonar, 2026, 1,000+ respondents). 🟡

**Phase 2: Agentic engineering.** Add orchestration — scouts, reviewers, multi-agent pipelines. The tools matured: Claude Code, Cursor 3, Copilot. But after 10 months and 878 merged PRs, Microsoft Research found the bottleneck had moved. Not typing speed — knowledge, judgment, and the ability to articulate what you actually want built. 🟡 The model stopped being the variable. You did.

Velocity scaled. Verification didn't. Amazon mandated Kiro across 21K agents, laid off 30K reviewers, then hit 4 Sev-1 incidents in 90 days — a 6-hour outage, 6.3M lost orders. 🟠

**Phase 3: Meta-programming.** The agent's operating context — memory, specs, rules, feedback loops — becomes the primary engineering artifact. Change the harness, not the model. Stanford proved this in March 2026: same LLM, different scaffolding, SOTA results. 🟡 SWE-bench: custom scaffolding alone adds +10%. Same model underneath. 🟡

You can't delegate your way to a better agent — you have to build the system around it that makes every session compound on the last.

## The thesis: natural language is code, thinking is architecture

Writing specs, structuring memory, and encoding feedback isn't prep work you do before the real engineering starts. It is the engineering.

LMP formalizes this into three layers (next section). The industry is converging on the same idea without coordinating. 60,000 repos now ship AGENTS.md — a cross-tool standard for agent instructions under the Linux Foundation. 🟠 GitHub's spec-kit hit 79K stars: "the lingua franca of development moves to a higher level, and code is the last-mile approach." 🟠

Academia named it. Microsoft Research: "Intent Formalization" — a grand challenge (March 2026). 🟡 Tsinghua's NLAH went further: harness behavior externalized as a portable executable artifact, independent of the model underneath. 🟡 DSPy treats prompts as learnable parameters and optimizes them automatically. Prompts as code. Literally. 🟡

The implementations are already live. Pydantic converted 4,668 PR review comments into ~150 AGENTS.md rules. 🟠 Claude Code rewrites SKILL.md from user corrections every fifth message. 🟡 Cursor's review agent: 78% issue resolution, self-improving from PR activity. 🟡

Everyone is building the same thing. Nobody agreed on a name.

## Three layers

### Layer 1: persistent model of self

Not facts about your project. Epistemology — how to decompose tasks, which trade-offs to prefer, what signals to trust.

Without persistent memory, an agent solves every problem the same way it solved it last time. Groundhog Day. We tested this directly: a generic Sonnet answered "yes, give it a code map" — standard advice, universally repeated. The same model loaded with an 8KB synthesis from our research answered "careful — exploration vs exploitation paradox," citing evidence from five prior experiments. 🟢 Same model. Same temperature. Same task. The difference wasn't code quality — it was thinking level.

Claude Code arrived at the same logic independently. `extractMemories` captures four types of per-session patterns. `/insights` promotes the strongest ones cross-session — using Opus specifically. Why? Cheap promotion creates rule pollution — you end up with dozens of low-signal rules that actively degrade future performance. Generalization needs the strongest model you have. 🟡 [Self-improvement](./self-improvement.md) covers the full mechanism.

### Layer 2: personal intent language

"Refactor the auth module" means ten different things to ten different agents. Formal specs avoid this ambiguity but take longer to write than the code itself. Intent language sits between: domain vocabulary plus constraints plus a shared model of the codebase that the agent carries forward across sessions.

The minimum viable version is three lines:

- **DO**: what to build, acceptance criteria
- **DO NOT**: what to leave alone
- **GLOSSARY**: terms that mean something specific in your codebase

Without these, our agent failed identically to a raw prompt — twice in a row, completely different failure modes each time (Experiment 2). 🟢 Three lines. That's all it took.

Commands change behavior; prose doesn't — verified across 10+ controlled runs. 🟠 Why? Agents pattern-match on imperative verbs and explicit conditions. Vague aspirations have nothing to match against. `run: npm test && assert coverage > 80%` gives them a concrete target.

[Specification](./specification.md) covers patterns, maturity levels, and the academic convergence around Intent Formalization.

### Layer 3: closed loop

Worker fails → reviewer catches it → lesson extractor writes a KB bullet → next worker gets the lesson in context. Each cycle narrows the gap between what you meant and what the agent produced.

This is the hardest layer to get right. DSPy is the closest production implementation — prompts treated as learnable parameters, optimized via BootstrapFewShot and MIPROv2 across hundreds of runs. 🟡 QodoAI does it differently — turning code review data into better agent skills through the same feedback structure. 🟡 Both prove the concept. Our implementation is in progress.

One critical distinction, confirmed independently and verified in Claude Code's source: extraction (per-session — "what specifically failed") works with cheap models. Promotion (cross-session — "what generalizable rule does this represent") needs the strongest available. Mix these up and you pollute the KB with noise that degrades future performance. 🟡

[Self-improvement](./self-improvement.md) has the details.

## How this was built

Nine experiments across 70+ sessions, generating 940+ atomic findings. One consistent benchmark: a 509-file TypeScript API on Cloudflare Workers with DDD architecture. Each experiment isolated one variable, measured cost and outcome, ran multiple times.

**Process beats context.** A spec→plan→review pipeline outperformed raw prompting on identical tasks at lower cost. Adding a code map actually made things worse — gave the agent a shortcut that killed exploration ($6.63 pipeline vs $9.99 pipeline-plus-codemap, 5 A/B runs). 🟢 ETH Zurich confirmed: auto-generated context reduced success by 3% across 138 real-world tasks while human-written boundaries improved it by 4%. 🟡 Two independent datasets. Same result.

**Boundaries prevent the majority of failures.** Three lines — DO, DO NOT, GLOSSARY — turned two consecutive raw-prompt failures into a first-attempt pass. 🟢

**Review must be separate.** Agents score their own work as excellent even when broken. Cross-model review catches failure modes neither model catches alone (GPT finds security issues Claude misses; Claude catches architecture GPT misses). 🟢🟡

**The edit tool is the main bug source.** Agents understand tasks correctly but physically break code during editing — a 7.1% error rate post-fix, all model mistakes, all self-recovering in one retry. 🟢 The meta-lesson was humbling. We spent two hours building statistical analysis, checking GitHub issues, constructing an elaborate causal story. Root cause? Our own extension bypassing the tool pipeline.

**Thinking level is a compliance dial — for one model version.** Claude 4.5-Nov at medium thinking = rule-compliant. At high thinking = conviction-mode, actively pushing back on instructions it disagrees with. Not gradual. Binary. Claude 4.6 doesn't have this dial at all — behavior is volatile across runs, same settings, different outcomes. 🟢

**Model degradation happens silently.** An April 2026 Opus incident: thinking depth dropped 67%, cost spiked 80×, conventions ignored despite an unchanged CLAUDE.md sitting right there in context. The pipeline survived. Externalized plans compensated for reduced thinking capacity. Output stayed above baseline. But the degradation went undetected for hours. 🟢

All 9 experiments with full methodology, data tables, and reproduction steps: [see Experiments](./experiments.md).

## How to read evidence levels

Every factual claim on this site carries a marker:

| Marker | Meaning |
|--------|---------|
| 🟢 **Proven** | We ran the experiment. A documented bullet exists with task, cost, and result. |
| 🟡 **Trusted source** | Anthropic, Microsoft Research, peer-reviewed papers, large-scale empirical studies. |
| 🟠 **Community reports** | Widely observed, not independently verified. GitHub stars, practitioner reports. |
| 🔴 **Unverified** | Heard it, didn't check. Rare — flagged only when worth noting. |
| ⚪ **Opinion** | Our synthesis. A conclusion we drew, not a measurement. |

🟢 leads. Always. 🟡 validates. 🟠 and ⚪ support — but never anchor a claim.

When 🟢 and 🟡 point in the same direction — as they do for process-over-context, mandatory boundaries, and separate review — that's as solid as findings get in a field this young.

## What's inside

| Page | What it covers |
|------|---------------|
| [Specification](./specification.md) | DO/DO NOT/GLOSSARY, the three-line minimum, SDD maturity, AGENTS.md as industry standard, Intent Formalization. |
| [Context engineering](./context-engineering.md) | Token budgets, why code maps hurt, progressive disclosure, scout caching, prompt caching mechanics. |
| [Pipeline](./pipeline.md) | Scout → Questions → Spec → Plan → Workers → Reviewer → Lessons. Architecture, costs, failure modes, when to skip stages. |
| [Verification](./verification.md) | Deterministic pre-flight before LLM review, cross-model review, the Amazon case study, why self-evaluation fails. |
| [Self-improvement](./self-improvement.md) | The closed loop: extraction vs promotion, KB as working memory, Claude Code's memory architecture, open gaps. |
| [Principles](./principles.md) | Six battle-tested principles with experiment references. Three maturity levels. Anti-patterns that cost real money. |
| [Landscape](./landscape.md) | Who else is doing this as of April 2026. DSPy, Cursor 3, AGENTS.md, spec-kit, and where it's all heading. |
