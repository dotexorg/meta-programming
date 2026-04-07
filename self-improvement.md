# Self-Improvement

Agents that improve through language — updating rules, extracting lessons, and pruning bad memories — without touching model weights. This is the operational core of meta-programming.

## Why Language, Not Weights

Fine-tuning is expensive, slow, and opaque. A model that rewrites its own instructions improves on the timescale of a session, costs the price of a few inference calls, and leaves an auditable trace in plain text files.

The practical evidence is accumulating fast. In March 2026 alone, three independent skill self-improvement projects shipped in one week — `skill-loop`, `selfwrite`, and `iterate` — each arriving at the same closed-loop architecture without coordination. ⚪ When practitioners independently converge on a pattern, the pattern is real. Implicit judgment can be made explicit at scale: 4,668 PR review comments became 150 behavioral rules the agent can load and follow (Pydantic, open source). 🟡 This is meta-programming at production scale.

Four distinct types of self-improvement exist: self-reflection, self-generated data, self-adapting models, and self-improving code agents (NeurIPS 2025 taxonomy). 🟡 The architecture described on this page combines type 1 (reflection) with elements of type 4 (code agent adaptation).

## Memory Hierarchy

Five levels of persistent context exist, each with a different scope and promotion threshold. 🟢

| Level | Mechanism | Scope |
|---|---|---|
| L1 | `CLAUDE.md` — always loaded | Static rules, project-wide |
| L2 | `.claude/rules/` — path-scoped | Directory or feature area |
| L3 | `extractMemories` — cross-session facts | 4 types, derivability-filtered |
| L4 | Session notes — 9 structured sections | Current session only |
| L5 | `autoDream` consolidation | Triggers at 24h + 5 sessions |

L3 is the critical layer. The derivability test asks: *can this fact be derived from the codebase itself?* If yes, it doesn't need to be remembered — it can be re-derived on demand. Only facts that require experience (preferences, patterns, past decisions) pass through. L5 consolidation prevents memory bloat: older session notes are merged and compressed before they accumulate into noise.

## Skill Improvement via Post-Sampling Hooks

Behavioral rules can be updated automatically from user corrections, with no external infrastructure required. 🟢 The `skillImprovement` hook runs after every five messages and checks whether the user corrected the agent's behavior during that window. If corrections are found, it rewrites `SKILL.md` to reflect what the agent should have done.

This is a minimal closed loop: behavior → correction → rule update → next invocation uses updated rule. The limitation is that it only captures explicit user corrections — silent failures pass through unnoticed. Pair it with a verification stage that surfaces failures the user didn't explicitly flag [see Verification](./verification.md).

## The ExpeL Framework

The self-improvement loop can be formalized into three stages: experience gathering, insight extraction, and task inference. 🟡 Validation on HotpotQA and ALFWorld shows the approach compounds — performance improves as the insight library grows.

**Stage 1 — Experience Gathering:** the agent solves tasks and saves full trajectories, including failed attempts. Failures are as valuable as successes because they carry error signals.

**Stage 2 — Insight Extraction:** an LLM pass over the trajectory batch extracts generalizable rules. The extraction prompt asks: *what patterns across these attempts predict success or failure?*

**Stage 3 — Task Inference:** future tasks load both the extracted insights and relevant past trajectories as few-shot context.

The gap between this research and production coding agents is real — HotpotQA trajectories are short and clean, coding sessions are long and messy. The pattern is sound; the engineering work to close that gap is not trivial.

## Extraction ≠ Promotion

Not every observed fact deserves permanent storage. Promotion thresholds should scale with storage permanence.

Extraction quality correlates directly with base model capability. 🟡 Weaker models can extract per-session facts reliably — what happened, what failed, what was decided. They fail at cross-task generalization: identifying that a failure in one domain implies a rule applicable to a different domain. Claude Code's implementation reflects this: `extractMemories` uses the current model, while `/insights` (cross-session synthesis) routes explicitly to Opus.

Learned heuristics outperform few-shot prompting once the agent has accumulated enough experience (Allard et al., ERL, Mar 2026). 🟡 In controlled trials, heuristic-guided agents scored +7.8% over ReAct baselines — suggesting that self-improvement compounds past a threshold rather than degrading as rules accumulate. This is the empirical argument for investing in a proper extraction pipeline: the returns are non-linear.

The practical principle: use cheap extraction liberally within sessions, use strong extraction conservatively across sessions. Demote as aggressively as you promote — memories that fail on application should lose score. [See Principles](./principles.md) for Principle 6 on this tradeoff.

## Practitioner Implementations

Three real-world deployments converge on the same write-from-failure architecture:

**Error-driven rules.** Learned rules carry more weight than static instructions because the agent treats them as empirically validated rather than hypothetical. 🟡 An error log with three fields per entry — what happened, why it happened, what should have happened — feeds a background pass that counts error patterns. At 3+ occurrences, it synthesizes a behavioral rule and promotes it to L1. After two weeks: 13 active rules generated from 211 indexed memories.

**Structured wrap-up.** The extraction moment, not the session itself, is what creates persistent improvement. 🟡 A four-phase end-of-session routine enforces it: Ship (commit and push) → Remember (review session lessons, promote to memory) → Review (flag mistakes worth extracting) → Publish. Without a deliberate extraction step, lessons evaporate with session context.

**Four-file memory split.** Separating lessons, errors, decisions, and todos by file prevents category pollution and enables targeted drift detection. 🟡 After each edit, a drift-detection pass checks for content inconsistent with the current state. Over 100 sessions on a production Java codebase, the outcome compounds: by session 20 the agent knows your patterns; by session 50, the codebase better than fresh context. Errors are never repeated.

## Self-Improving Tools

Three tools implement self-improvement as a first-class concern:

**skill-loop:** An MCP server that runs a four-stage loop against any skill file — observe → inspect → amend → evaluate. 🟡 The MCP interface means the loop can be triggered programmatically, not just by end-of-session hooks. This is the first concrete closed loop for skill self-improvement with external observability.

**Stanford Meta-Harness** automates the harness itself, not just the agent. Rather than manually tuning the evaluation loop, it treats harness configuration as an optimization target. 🟡 Reported results show a 6x gap between optimized and unoptimized harnesses on the same agent — confirming that self-improvement infrastructure matters as much as the agent itself.

**DSPy** treats prompts as learnable parameters. Optimizers like BootstrapFewShot, MIPROv2, and GEPA search over prompt variants given a training set and scoring function. 🟡 When the underlying model changes, you recompile rather than manually re-engineer prompts. This is the closest existing implementation to automated L3 memory management — the prompt *is* the extracted cross-session knowledge, and the optimizer is the extractor.

For a full catalog of tools in this space, [see Landscape](./landscape.md).

## Memory Systems

Memory becomes self-correcting when every unit of knowledge carries a score that updates on use. 🟡 ACE (Agentic Context Engineering) structures this as a pipeline with four roles — Generator produces candidates, Reflector evaluates them against recent behavior, Curator manages the store by score, Playbook serves relevant context at inference time. Each bullet carries an ID, embedding, and two counters: helpful and harmful. Freshness decays over time. Bullets that consistently help rise; bullets that mislead fall and are eventually pruned.

Individual episodes carry an effectiveness score updated on each use: +0.3 on success, −0.5 on failure. 🟡 Score below 0.3 triggers deprioritization. The asymmetric weighting is deliberate — a single misleading memory can cascade into multiple wrong decisions, so the penalty must exceed the reward.

Episodic reflections can accumulate into a higher-order policy layer. MPR (Meta-Policy Reflexion) builds a Meta-Policy Memory of predicate-style rules with confidence weights. 🟡 Two enforcement modes: SOFT biases token probabilities toward rule-consistent actions; HARD (Rule Admissibility Checks) blocks actions that violate active rules outright. Hard blocking outperforms soft-only enforcement — preventing rule violations that soft biasing merely discourages.

## Closed-Loop Architecture

The pattern shared across all implementations:

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

The feedback pipeline closes the loop [see Pipeline](./pipeline.md). Verification signals — test results, linter output, user corrections — are what make extraction possible. Without reliable feedback, the agent extracts from noise. 🟢

## Open Questions

- **Extraction timing:** end-of-session batch vs. inline after each tool call. Inline catches more signal but adds latency to every step.
- **Rule conflict resolution:** when two promoted rules contradict each other, which wins? Current implementations use recency; confidence weighting is unexplored in practice.
- **Cross-agent sharing:** rules extracted by one agent instance — are they transferable to another agent on the same codebase? No published evidence yet.
- **Extraction under adversarial input:** a user who deliberately provides incorrect corrections could poison the skill file. No defense mechanism exists in current implementations.
