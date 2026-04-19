# What Changed: April 17, 2026

For readers who already went through the site. One entry per page, only what's new. Skip the rest.

The big shifts this round: Layer 2 got rewritten, Opus 4.7 shipped with a silent repricing, and four academic papers put numbers on claims we'd been making from experience.

---

## The three biggest changes

**Layer 2 reformulation.** "Personal Intent Language" is gone. It was aspirational placeholder that read well and wasn't real. The replacement is **Intent Form Selection**: pick the form (NL, AGENTS.md, SKILL.md, DO/DO NOT, rules, hooks) that matches task complexity, reliability requirement, and horizon. Not a new language to invent. A decision tree for the forms that already exist. See [index → Layer 2](./index.md#layer-2-intent-form-selection) and [principles → open questions](./principles.md#open-questions).

**Opus 4.7 silent repricing (Apr 16).** Nominal pricing unchanged at $5/$25. Real cost +33–50% on the same workload: new tokenizer inflates tokens 1.25–1.35×, `budget_tokens` silently reroutes to `task_budget` without erroring, `xhigh` is the new default, self-verification consumes +15% output tokens. Cache stays cold for 2–4 weeks post-migration. Reliable configs remain Opus 4.5-Nov HIGH and Sonnet 4.6 HIGH; Opus 4.7 is not yet on the reliable list. New [landscape → Opus 4.7 and the silent repricing](./landscape.md#opus-47-and-the-silent-repricing) section.

**Four papers landed in four weeks.** Empirical numbers now exist for claims that used to be experiential:

| Paper | What it measured | Where it lives now |
|-------|-----------------|-------------------|
| Spec Gap (ICN2, arxiv 2603.24284) | 89→56% single-agent pass rate as spec detail drops. 58→25% multi-agent. 16pp coordination + 11pp info asymmetry. AST conflict detector Δ=0pp. Full spec recovers ceiling. | [specification](./specification.md), [pipeline](./pipeline.md), [principles](./principles.md) |
| Context Engineering (Swift North, arxiv 2604.04258) | Five-role context package (Authority/Exemplar/Constraint/Rubric/Metadata). Iterations 3.8→2.0, first-pass 32→55%. | [specification](./specification.md) |
| SLUMP (Purdue, arxiv 2603.17104) | Specs drift during sessions. External state tracker (ProjectGuard) recovers 90% faithfulness gap. | [specification](./specification.md) |
| Cognitive Companion (IBM, arxiv 2604.13759) | Four states: ON_TRACK / LOOPING / DRIFTING / STUCK. LLM-based detector cuts repetition 52–62% at 11% overhead. | [verification](./verification.md) |

---

## Per-page deltas

### [index.md](./index.md)

- Layer 2 rewritten from "Personal Intent Language" to "Intent Form Selection." Intent gap quantified: 32% of production LLM failures.
- KB stat updated: 9 → 10 experiments, ~430 → ~1,000 bullets.

### [specification.md](./specification.md)

- Spec Gap numbers added as empirical backbone under "Why boundaries aren't optional."
- Intent Gap 4-layer model (immediate / final / background / autonomy). 32% production failure rate. RLHF optimises for helpfulness → training signal points at the gap.
- Assumption propagation named as core failure mode. CodeRabbit 470 repos: 1.7× more bugs, 75% more logic errors, 8× I/O bugs in AI code.
- Context Engineering 5-role package added to "Minimum viable spec" section.
- SLUMP / ProjectGuard added under Open Questions.

### [pipeline.md](./pipeline.md)

- **New section: "Parallel decomposition has a measured tax."** 16pp coordination + 11pp info asymmetry floor on parallel workers sharing state. Scope of finding spelled out: does not apply to sequential pipelines or role-specialised agents. u/thurn2 quote: "agent teams = expensive subagents with better marketing."
- **New section: "The pipeline's advantage shrinks as models improve."** Behavioral Drivers (NCSU, 9,374 trajectories): successful behaviour is agent-determined, framework effect narrows with each model generation. What to measure going forward.
- Parallel decomposition added to Settled Questions.

### [verification.md](./verification.md)

- Expectation-Realisation Gap (arxiv 2602.20292) sharpened METR finding: +24% expected, −19% measured, 43pp calibration error.
- **New section: "Four cognitive states in a stuck agent."** Cognitive Companion paper formalises perseveration detection. LLM-based detector (API-accessible, 11% overhead) vs probe-based (requires open weights). Production baselines (LangGraph `recursion_limit`, AutoGen termination, OpenHands StuckDetector) named as rule-based and missing semantic drift.

### [self-improvement.md](./self-improvement.md)

- **New section: "Chase's three learning layers."** Model / Harness / Context from LangChain's continual learning framework. Orthogonal to LMP: LMP is about what form, Chase is about where in the stack. OpenClaw explicitly mapped as harness layer. Hot-path vs offline memory updates.
- New practitioner: `adelaidasofia/claude-performance`, measurement-driven CLAUDE.md with rule retirement (add → measure → retire). "A static rule is a wish. A measured rule is a system."
- New practitioner: Homunculus plugin, pattern observation daemon auto-writes skills/hooks/commands.
- WebXSkill paper added: skill = parameterized action + NL guidance. +9.8 / +12.9 pts on WebArena / WebVoyager.

### [context-engineering.md](./context-engineering.md)

- **New section: "The AGENTS.md cliff."** Paper (arxiv 2602.11988) quantifies context file cliff: success drops past 500 lines. Sweet spot 200–300 actionable lines. ~70% adherence even under budget. Hooks for rules that must hold every time, context files are advisory.
- KB size updated: 429 → 1,000+ bullets, synthesis 8KB → 12KB.

### [principles.md](./principles.md)

- Principle #1 (Process > Context): Behavioral Drivers caveat added. Pipeline alpha shrinks as models strengthen. What to track over time.
- Principle #2 (Boundaries): Spec Gap academic experiment added as quantitative backing for the ablation.
- Open Questions rewritten: Layer 2 "intent language" → "form selection measurement problem." New open question: does pipeline alpha shrink monotonically.

### [landscape.md](./landscape.md)

- **New top section: "Opus 4.7 and the silent repricing."** Tokenizer tax, `budget_tokens` deprecation, `xhigh` default, +33.5% real cost.
- **New section: "April 2026: the spec-driven research wave."** Spec Gap, Context Engineering, SLUMP, AGENTS.md paper, Intent Gap, all consolidated.
- **New section: "Harrison Chase's three-layer framing."** Traces as shared substrate.
- **New section: "GitLab's five autonomy levels."** L1–L5 ladder. Warning about skipping to L4/L5 without verification infrastructure; maps to Amazon's Sev-1 pattern.
- **New section: "Seven anti-patterns from production telemetry."** TechDebt.guru / GitClear / CodeRabbit numbers.
- Armin Ronacher added to "People worth following."
- Rule-retirement loop added to self-improvement trend.

### [playbook.md](./playbook.md)

*(Updated one day earlier, Apr 16)*

- Rule 1 strengthened with 70% adherence number and hook distinction.
- Rule 4 backed with Spec Gap quantification (89→56, 58→25, full spec recovers ceiling).
- Rule 5 strengthened with Intent Gap 32% production failure stat and XY problem framing.
- Rule 8 gets four-state taxonomy reference (ON_TRACK / LOOPING / DRIFTING / STUCK).
- Rule 15 closes with rule-retirement pattern from `claude-performance`.

### [references.md](./references.md)

- 10 new academic papers indexed (17b–17j): Spec Gap, Context Engineering, SLUMP, Intent Gap, Behavioral Drivers, Cognitive Companion, AGENTS.md, Expectation-Realisation Gap, WebXSkill.
- 6 new community / vendor sources (30b–30f, 35b–35d): Chase, GitLab, TechDebt.guru, Cursor best practices, Opus 4.7 migration, `claude-performance`, Homunculus, u/thurn2.
- Experiment 10 added: Opus 4.7 release analysis.
- People section updated: Chase moved from 🟠 to 🟡, Ronacher added.

---

## What was settled (moved out of Open Questions)

- **Parallel decomposition of shared-state code**: quantified and settled, don't do it. Sequential workers sharing a spec artefact is the working pattern. ([pipeline](./pipeline.md#settled-questions))
- **Layer 2 direction**: settled on form selection, not language invention. Open question moved to "which form for which task" measurement. ([principles](./principles.md#open-questions))

## What's newly open

- **Does pipeline alpha shrink monotonically as models improve?** Behavioral Drivers finding implies yes. No longitudinal data yet. ([landscape](./landscape.md#open-questions), [principles](./principles.md#open-questions))
- **Hot-path memory during a session.** Our KB extraction is offline-only. Chase's framing puts a name on the gap; we don't have an implementation. ([self-improvement](./self-improvement.md))
- **Spec faithfulness under emergent specification.** SLUMP's ProjectGuard pattern says enforce "every worker re-reads the spec." We have persistent artefacts but no systematic re-check. ([specification](./specification.md#open-questions))

---

*Previous changelogs: none. This is the first one. Future rounds: one file per sync date, appended below.*
