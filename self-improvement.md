# Self-Improvement

Agents that improve through language — updating rules, extracting lessons, and pruning bad memories — without touching model weights. This is the operational core of meta-programming.

## Why Language, Not Weights

Fine-tuning is expensive, slow, and opaque. A model that rewrites its own instructions improves on the timescale of a session, costs the price of a few inference calls, and leaves an auditable trace in plain text files.

The difference is not theoretical. 🟢 In three controlled A/B tests, an 8KB structured knowledge base transformed agent behavior from conventional wisdom to evidence-based reasoning. Without the KB, a generic agent recommended giving code maps to agents ("yes, code map is useful, detailed specs are better"). With the KB, the same model contradicted that advice with specific evidence from prior experiments. On a production task, the agent without KB proposed a technical migration; the agent with KB proposed an architectural redesign. The difference was not code quality — it was thinking level. Accumulated linguistic knowledge changed what the agent tried, not just how it executed.

The pattern holds beyond our experiments. Implicit judgment can be made explicit at scale: 4,668 PR review comments became 150 behavioral rules the agent can load and follow (Pydantic, open source). 🟡 Three independent skill self-improvement projects shipped in the same week — `skill-loop`, `selfwrite`, and `iterate` — each arriving at the same closed-loop architecture without coordination. 🟠 When practitioners independently converge on a pattern, the pattern is real.

Four distinct types of self-improvement exist: self-reflection, self-generated data, self-adapting models, and self-improving code agents (NeurIPS 2025 taxonomy). 🟡 The architecture described on this page combines type 1 (reflection via reviewer feedback) with elements of type 4 (lesson-extractor updating rules).

## Our Closed Loop

The pipeline's final stage — lesson extraction — closes the feedback loop. After each feature, a dedicated agent reviews what broke, what was unexpectedly hard, and what assumption was wrong. The output is atomic, generalizable rules saved to the knowledge base. 🟢

In practice: after a 24-file refactoring pipeline (Experiment 3), the lesson-extractor produced two patterns — "grep for barrel imports when moving types" and "check wildcard re-export conflicts when splitting shared modules." Both were the exact failure categories that caused the reviewer to fail three times before passing. 🟢 Those patterns now load into future pipeline runs, preventing the same class of error from recurring.

The loop:

```
Agent → action → tests
  IF fail:
    extract error + lesson → save to episodic store
    next attempt loads relevant episodes
  IF 3+ same error:
    synthesize permanent rule → add to L1/L2
  IF rule applied but task still fails:
    decrement rule score
    IF score < threshold → demote or delete
```

The verification signals — test results, linter output, user corrections — are what make extraction possible. Without reliable feedback, the agent extracts from noise. And even with reliable feedback, extraction can go wrong: a lesson that was correct for one context may be overfitted to that specific task. If the lesson "always use barrel re-exports" is extracted from a case where barrels happened to work, it becomes a landmine in a project where barrels cause circular dependencies. Demotion scoring (see Memory Scoring below) is the defense — lessons that hurt on application lose score and eventually get pruned. [See Verification](./verification.md) for why feedback infrastructure determines the self-improvement ceiling.

## Memory Hierarchy

Five levels of persistent context exist, each with a different scope and promotion threshold. 🟢 The table below uses Claude Code's implementation as the reference — it is the most fully documented production agent architecture — but the pattern generalizes. Any agent with file-based rules, scoped overrides, cross-session extraction, within-session notes, and periodic consolidation follows the same hierarchy, regardless of what the files are called.

| Level | What it does | Example (Claude Code) |
|---|---|---|
| L1 | Static rules, always loaded | `CLAUDE.md` (or `AGENTS.md`, project config) |
| L2 | Path-scoped overrides | `.claude/rules/` with glob patterns |
| L3 | Cross-session facts, derivability-filtered | `extractMemories` — 4 types |
| L4 | Within-session structured notes | Session Memory — 9 sections |
| L5 | Periodic consolidation and pruning | `autoDream` — triggers at 24h + 5 sessions |

L3 is the critical layer. The derivability test asks: *can this fact be derived from the codebase itself?* If yes, it doesn't need to be remembered — it can be re-derived on demand. Only facts that require experience (preferences, patterns, past decisions) pass through. L5 consolidation prevents memory bloat: older session notes are merged and compressed before they accumulate into noise.

We operate at L1 today, with our KB serving as a manual L3. The gap between these levels — L2 path-scoped rules, automated L3 extraction, L4 session memory, L5 consolidation — is the implementation roadmap. [See Pipeline](./pipeline.md) for how worker isolation maps to context reset at each level.

Two always-loaded files — not five levels — can be enough to close the most expensive gap. 🟢 After forgetting the same workflow five times in a single session, we reduced the architecture to `persona.md` (~1KB, rewritten each wrap by the current model) and `rules.md` (~500 tokens, manually edited, never touched by automation). The session journal was eliminated: platform-native JSONL session history already captures what the journal duplicated. Each `/wrap` rewrites `persona.md` from scratch — ~15 bullets, medium detail, current decisions and next steps — while `rules.md` holds sticky workflow conventions and key paths that shouldn't drift session to session. Mapped to the CC hierarchy: `rules.md` ≈ L1 (stable behavioral constraints), `persona.md` ≈ L3 (session-derived facts), platform session files ≈ L4. The design rule: automate only the levels where imprecision is recoverable.

## Extraction ≠ Promotion

Not every observed fact deserves permanent storage. Promotion thresholds should scale with storage permanence.

We encode this principle in our evidence hierarchy: 🟢 experiments we ran ourselves (🟢) carry more weight than trusted external sources (🟡), which carry more weight than community claims (🟠). The same asymmetry applies to agent memory — a pattern observed in three of our pipeline runs is a stronger candidate for promotion than a technique mentioned in a paper we haven't tested. This isn't bias; it's calibration. The agent that promoted everything equally would drown in noise.

Extraction quality correlates directly with base model capability. 🟡 Weaker models can extract per-session facts reliably — what happened, what failed, what was decided. They fail at cross-task generalization: identifying that a failure in one domain implies a rule applicable to a different domain. Claude Code's implementation reflects this: `extractMemories` uses the current model, while `/insights` (cross-session synthesis) routes explicitly to Opus.

Learned heuristics outperform few-shot prompting once the agent has accumulated enough experience. 🟡 In controlled trials, heuristic-guided agents scored +7.8% over ReAct baselines (ERL, Mar 2026) — suggesting that self-improvement compounds past a threshold rather than degrading as rules accumulate. This is the empirical argument for investing in a proper extraction pipeline: the returns are non-linear.

The practical principle: use cheap extraction liberally within sessions, use strong extraction conservatively across sessions. Demote as aggressively as you promote — memories that fail on application should lose score. [See Principles](./principles.md) for Principle 6 on this tradeoff.

## Automatic Rule Updates from User Corrections

Behavioral rules can be updated automatically from user corrections, with no external infrastructure required. 🟢 In Claude Code, a background hook runs after every five messages and checks whether the user corrected the agent's behavior during that window. If corrections are found, it rewrites the skill file to reflect what the agent should have done. The mechanism is platform-specific; the principle — observe corrections, update rules, apply next time — is universal.

This is a minimal closed loop: behavior → correction → rule update → next invocation uses updated rule. The limitation is that it only captures explicit user corrections — silent failures pass through unnoticed. Pair it with a verification stage that surfaces failures the user didn't explicitly flag [see Verification](./verification.md).

## The ExpeL Framework

The self-improvement loop can be formalized into three stages: experience gathering, insight extraction, and task inference. 🟡 Validation on HotpotQA and ALFWorld shows the approach compounds — performance improves as the insight library grows.

**Stage 1 — Experience Gathering:** the agent solves tasks and saves full trajectories, including failed attempts. Failures are as valuable as successes because they carry error signals.

**Stage 2 — Insight Extraction:** an LLM pass over the trajectory batch extracts generalizable rules. The extraction prompt asks: *what patterns across these attempts predict success or failure?*

**Stage 3 — Task Inference:** future tasks load both the extracted insights and relevant past trajectories as few-shot context.

Our lesson-extractor is a production adaptation of this pattern. 🟢 The gap between the research framework (short, clean trajectories on QA benchmarks) and production coding (long, messy sessions with file edits and test runs) is real — but the three-stage structure holds. We gather experience through pipeline runs, extract lessons through a dedicated post-merge agent, and load relevant lessons into future workers.

## Practitioner Patterns

Three patterns emerge from real-world deployments — including our own:

**Error-driven rules.** After each pipeline run, extract structured failure data: what happened, why, what should have happened. Count patterns across runs. At 3+ occurrences, synthesize a behavioral rule and promote it. 🟢 Our lesson-extractor follows this pattern — the two rules from Experiment 3 (barrel imports, wildcard conflicts) emerged from exactly this process. A separate practitioner deployment reports 13 active rules generated from 211 indexed memories after two weeks, with learned rules carrying more weight than static instructions because the agent treats them as empirically validated. 🟡

**Structured wrap-up.** The extraction moment, not the session itself, is what creates persistent improvement. 🟡 A four-phase end-of-session routine enforces it: Ship (commit and push) → Remember (review session lessons, promote to memory) → Review (flag mistakes worth extracting) → Publish. Without a deliberate extraction step, lessons evaporate with session context.

**Four-file memory split.** Separating lessons, errors, decisions, and todos by file prevents category pollution and enables targeted drift detection. 🟡 After each edit, a drift-detection pass checks for content inconsistent with the current state. Over 100 sessions on a production Java codebase: by session 20 the agent knows your patterns; by session 50, the codebase better than fresh context. Errors are never repeated.

**Self-patching skills.** Skills can rewrite their own failing steps from recorded failure data, without a human editing them. 🟡 The `/evolve` command in Claude Recall analyzes which steps produced errors across past sessions and rewrites those steps in-place. This closes a gap in the CC `skillImprovement.ts` pattern: the post-sampling hook only fires when a skill is currently active. Retrospective `/evolve` runs against any skill with a failure history, regardless of when it last ran.

**Debugging-loop RAG.** Persistent memory wired into the debugging loop — not just planning — compounds quality as the codebase ages. 🟡 One production deployment captures each resolved bug as a retrieval entry: the next time a similar failure appears, the relevant fix surfaces automatically. No manual curation step; the debugging history is the training set.

## Self-Improving Tools

Three tools implement self-improvement as a first-class concern:

**skill-loop:** A tool that runs a four-stage loop against any skill file — observe → inspect → amend → evaluate. 🟠 It exposes the loop as a programmatic interface (via MCP, the emerging standard for tool integration), so improvement can be triggered automatically, not just by end-of-session hooks.

**Stanford Meta-Harness** automates the harness itself, not just the agent. Rather than manually tuning the evaluation loop, it treats harness configuration as an optimization target. 🟡 A 6x performance gap between optimized and unoptimized harnesses on the same agent confirms that self-improvement infrastructure matters as much as the agent itself.

**DSPy** treats prompts as learnable parameters. Optimizers like BootstrapFewShot, MIPROv2, and GEPA search over prompt variants given a training set and scoring function. 🟡 When the underlying model changes, you recompile rather than manually re-engineer prompts. This is the closest existing implementation to automated cross-session memory management — the prompt *is* the extracted knowledge, and the optimizer is the extractor.

For a full catalog of tools in this space, [see Landscape](./landscape.md).

## Memory Scoring

Memory becomes self-correcting when every unit of knowledge carries a score that updates on use. Our KB implements this through `helpful` and `harmful` feedback on individual bullets — a pattern formalized in the ACE framework (Agentic Context Engineering). 🟢

The scoring formula is simple: `effectiveness = (helpful - harmful) / (total + 1)`. The +1 (Laplace smoothing) prevents noise from single votes on new entries. Bullets that consistently help rise in retrieval priority; bullets that mislead fall and are eventually pruned.

Individual episodes carry an effectiveness score updated on each use: +0.3 on success, −0.5 on failure. 🟡 The asymmetric weighting is deliberate — a single misleading memory can cascade into multiple wrong decisions, so the penalty must exceed the reward. Score below 0.3 triggers deprioritization.

Episodic reflections can accumulate into a higher-order policy layer. Meta-Policy Reflexion builds predicate-style rules with confidence weights. 🟡 Two enforcement modes: SOFT biases token probabilities toward rule-consistent actions; HARD blocks actions that violate active rules outright. Hard blocking outperforms soft-only enforcement — preventing rule violations that soft biasing merely discourages.

Production deployments show the pattern works at scale with simple stacks. Output quality saturates at roughly seven governed memories per entity — adding more provides no measurable benefit. 🟡 This ceiling emerged from multi-agent research (Personize.ai, 2026): 99.6% fact recall, 50% token reduction from progressive delivery, zero cross-entity leakage on 500 adversarial queries. A separate deployment across 232 tasks with zero failures used four components: an extraction daemon polling logs, Haiku extracting facts, a markdown vault with vector search, and an injection hook surfacing the top-2 relevant entries per prompt. 🟡 The complexity ceiling is low — returns come from the feedback loop, not the infrastructure.

## Open Questions

- **Extraction timing:** end-of-session batch vs. inline after each tool call. Inline catches more signal but adds latency to every step.
- **Rule conflict resolution:** when two promoted rules contradict each other, which wins? Current implementations use recency; confidence weighting is unexplored in practice.
- **Cross-agent sharing:** rules extracted by one agent instance — are they transferable to another agent on the same codebase? No published evidence yet.
- **Extraction under adversarial input:** a user who deliberately provides incorrect corrections could poison the skill file. No defense mechanism exists in current implementations.
- **KB scaling:** At ~935 bullets, retrieval quality is partially addressed. A search v2 implementation — threshold-based filtering, relevance scoring, and a top-15 cap — reduced broad-query noise from 214 results to 5 actionable hits. 🟢 Narrow queries now work reliably; the tier files and progressive disclosure handle broad exploration. The remaining gap is cross-bullet linking: related findings stored under different tags don't surface together unless the query happens to match both. At the current scale this is manageable; at 1,500+ bullets it will require graph-based retrieval or explicit cross-references.
