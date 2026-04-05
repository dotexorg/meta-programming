# Self-Improvement

Agents that improve through language — updating rules, extracting lessons, and pruning bad memories — without touching model weights. This is the operational core of meta-programming.

## On This Page

- [Why Language, Not Weights](#why-language-not-weights)
- [Memory Hierarchy](#memory-hierarchy)
- [Skill Improvement via Post-Sampling Hooks](#skill-improvement-via-post-sampling-hooks)
- [The ExpeL Framework](#the-expel-framework)
- [Extraction ≠ Promotion](#extraction--promotion)
- [Practitioner Implementations](#practitioner-implementations)
- [Self-Improving Tools](#self-improving-tools)
- [Memory Systems](#memory-systems)
- [Open Questions](#open-questions)

---

## Why Language, Not Weights

Fine-tuning is expensive, slow, and opaque. A model that rewrites its own instructions is something different: it improves on the timescale of a session, costs the price of a few inference calls, and leaves an auditable trace in plain text files.

The practical evidence is accumulating fast. In March 2026 alone, three independent skill self-improvement projects shipped in one week — `skill-loop`, `selfwrite`, and `iterate` — each arriving at the same closed-loop architecture without coordination. ⚪ When practitioners independently converge on a pattern, the pattern is real. The Pydantic team codified 4,668 PR review comments into 150 explicit behavioral rules, turning implicit judgment into instructions the agent can load and follow. 🟡 This is Language Model Programming at open-source scale.

NeurIPS 2025 taxonomy identifies four types of self-improvement: self-reflection, self-generated data, self-adapting models, and self-improving code agents. 🟠 The architecture described on this page combines type 1 (reflection) with elements of type 4 (code agent adaptation).

---

## Memory Hierarchy

Claude Code exposes five levels of persistent context, each with a different scope and promotion threshold. 🟢

| Level | Mechanism | Scope |
|---|---|---|
| L1 | `CLAUDE.md` — always loaded | Static rules, project-wide |
| L2 | `.claude/rules/` — path-scoped | Directory or feature area |
| L3 | `extractMemories` — cross-session facts | 4 types, derivability-filtered |
| L4 | Session notes — 9 structured sections | Current session only |
| L5 | `autoDream` consolidation | Triggers at 24h + 5 sessions |

L3 is the critical layer. The derivability test asks: *can this fact be derived from the codebase itself?* If yes, it doesn't need to be remembered — it can be re-derived on demand. Only facts that require experience (preferences, patterns, past decisions) pass through. L5 consolidation prevents memory bloat: older session notes are merged and compressed before they accumulate into noise.

---

## Skill Improvement via Post-Sampling Hooks

The `skillImprovement` hook runs after every five messages and checks whether the user corrected the agent's behavior during that window. If corrections are found, it rewrites `SKILL.md` to reflect what the agent should have done. 🟢

This is a minimal closed loop: behavior → correction → rule update → next invocation uses updated rule. The loop requires no external infrastructure beyond the skill file itself. The limitation is that it only captures explicit user corrections — silent failures pass through unnoticed. Pair it with a verification stage that surfaces failures the user didn't explicitly flag [see Verification](./verification.md).

---

## The ExpeL Framework

ExpeL (Experience + Learning) formalizes the self-improvement loop into three stages. 🟠 Published research, validated on HotpotQA and ALFWorld — not coding agents.

**Stage 1 — Experience Gathering:** the agent solves tasks and saves full trajectories, including failed attempts. Failures are as valuable as successes because they carry error signals.

**Stage 2 — Insight Extraction:** an LLM pass over the trajectory batch extracts generalizable rules. The extraction prompt asks: *what patterns across these attempts predict success or failure?*

**Stage 3 — Task Inference:** future tasks load both the extracted insights and relevant past trajectories as few-shot context. Performance compounds as the insight library grows.

The gap between ExpeL's research results and a production coding agent is real — HotpotQA trajectories are short and clean, coding sessions are long and messy. The pattern is sound; the engineering work to close that gap is not trivial.

---

## Extraction ≠ Promotion

Not every observed fact deserves permanent storage. Promotion thresholds should scale with storage permanence.

OpenReview 2025 data shows that extraction quality correlates directly with base model capability. 🟠 Weaker models can extract per-session facts reliably — what happened, what failed, what was decided. They fail at cross-task generalization: identifying that a failure in one domain implies a rule applicable to a different domain. Claude Code's own implementation reflects this: `extractMemories` uses the current model, while `/insights` (cross-session synthesis) routes explicitly to Opus.

The practical principle: use cheap extraction liberally within sessions, use strong extraction conservatively across sessions. Demote as aggressively as you promote — memories that fail on application should lose score. [See Principles](./principles.md) for Principle 6 on this tradeoff.

---

## Practitioner Implementations

Three real-world deployments, each converging on the same write-from-failure architecture:

**Rory Teehan's error-driven rules.** The agent maintains a structured error log with three fields per entry: what happened, why it happened, what should have happened. A background pass counts error patterns; at 3+ occurrences, it synthesizes a behavioral rule and promotes it to L1. After two weeks: 13 active rules generated from 211 indexed memories. 🟡 The key finding: learned rules carry more weight than static instructions written by humans, because the agent treats them as empirically validated rather than hypothetical.

**jonathanmalkin's wrap-up skill.** A four-phase end-of-session routine: Ship (commit and push) → Remember (review session lessons, promote to memory) → Review (flag mistakes worth extracting) → Publish. 🟡 Self-reported as "the one I actually use every single session." The enforced review step is the mechanism — without a deliberate extraction moment, lessons evaporate with session context.

**Claude Recall.** Four files: lessons, errors, decisions, and todos. After each edit, a drift-detection pass checks whether any file's content has become inconsistent with the current state. 🟡 Tested over 100 sessions on a production Java codebase. Reported outcome: "By session 20 Claude knows your patterns, by session 50 — the codebase better than any fresh context. Errors are never repeated." The four-file split enforces separation of concerns: lessons (generalizable) stay separate from decisions (project-specific), preventing category pollution.

---

## Self-Improving Tools

Three tools implement self-improvement as a first-class concern:

**skill-loop (stylusnexus):** An MCP server that runs a four-stage loop against any skill file — observe → inspect → amend → evaluate. 🟡 The first concrete closed loop for skill self-improvement with external observability. The MCP interface means the loop can be triggered programmatically, not just by end-of-session hooks.

**pi-autoresearch (2,288★):** Karpathy's autoresearch pattern as a Pi extension. The loop: edit → commit → run experiment → log result → keep or revert → repeat. 🟠 Designed for research automation but the loop structure maps directly to skill refinement: each iteration is a controlled experiment on agent behavior.

**Iris:** Closed-loop agent with integrated self-improvement. Trajectories feed directly into rule synthesis. Not yet publicly documented, but referenced in multiple practitioner threads as the most complete implementation of the full pipeline. 🔴

For a full catalog of tools in this space, [see Landscape](./landscape.md).

---

## Memory Systems

**ACE (Agentic Context Engineering):** Structures memory as a pipeline with four roles — Generator produces candidates, Reflector evaluates them against recent behavior, Curator manages the store by score, Playbook serves relevant context at inference time. 🟠 Each bullet (unit of knowledge) carries an ID, embedding, and two counters: helpful and harmful. Freshness decays over time. This makes the memory self-correcting: bullets that consistently help rise; bullets that mislead fall and are eventually pruned.

**Session memory and episode pruning.** Individual episodes carry an effectiveness score updated on each use: +0.3 on success, −0.5 on failure. Score below 0.3 triggers deprioritization. 🟠 The asymmetric weighting (failure costs more than success gains) is deliberate — a single misleading memory can cascade into multiple wrong decisions, so the penalty must exceed the reward.

**MPR (Meta-Policy Reflexion).** Episodic reflections accumulate into a Meta-Policy Memory of predicate-style rules with confidence weights. 🟠 Two enforcement modes: SOFT biases token probabilities toward rule-consistent actions; HARD (Rule Admissibility Checks) blocks actions that violate active rules outright. Empirically, HAC outperforms soft-only enforcement — hard blocking prevents rule violations that soft biasing merely discourages.

**DSPy** takes a different angle: it treats prompts themselves as learnable parameters. Optimizers like BootstrapFewShot, MIPROv2, and GEPA search over prompt variants given a training set and scoring function. 🟠 When the underlying model changes, you recompile rather than manually re-engineer prompts. This is the closest existing implementation to automated L3 memory management — the prompt *is* the extracted cross-session knowledge, and the optimizer is the extractor.

---

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

---

## Open Questions

- **Extraction timing:** end-of-session batch vs. inline after each tool call. Inline catches more signal but adds latency to every step.
- **Rule conflict resolution:** when two promoted rules contradict each other, which wins? Current implementations use recency; confidence weighting is unexplored in practice.
- **Cross-agent sharing:** rules extracted by one agent instance — are they transferable to another agent on the same codebase? No published evidence yet.
- **Extraction under adversarial input:** a user who deliberately provides incorrect corrections could poison the skill file. No defense mechanism exists in current implementations.
