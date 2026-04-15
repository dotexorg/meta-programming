# Self-improvement: agents that get better without fine-tuning

Fine-tuning costs weeks and tens of thousands of dollars per cycle. Linguistic self-improvement costs a dollar. This is the closed loop at the core of meta-programming: rules extracted from session failures, lessons promoted to permanent conventions, memories consolidated across months of work. All in natural language, without changing a single weight.

## Why language beats fine-tuning

Fine-tuning captures behavioral shifts durably. It also produces changes opaque to inspection, requires labeled failure data at scale, and runs weeks behind the feedback it addresses. Most agent behaviors worth fixing don't clear that bar. They're one-session discoveries: a naming convention preference, a do-not-touch boundary, a specific error pattern in async middleware.

The question isn't whether the linguistic approach can replace fine-tuning for large-scale behavioral shifts (it can't). The question is whether it handles the 90% of cases that aren't large-scale shifts. A rule update takes effect immediately at the cost of a dollar. That's the comparison.

The behavioral difference is measurable. In three controlled A/B tests, loading an 8KB knowledge base changed not what the agent executed but what it *tried*. Without the KB, a Sonnet model answered "yes, code maps improve orientation". The conventional-wisdom answer. With the KB loaded, the same model flagged the exploration-exploitation tradeoff, citing a specific experiment where a code map increased cost by 51% and caused the run to fail. On a production refactor the gap was starker: KB-less agent proposed a technical migration ("replace JSON parsing"); KB agent proposed an architectural redesign: three-type router with tool-calling. The KB didn't grant new capabilities. It changed the reference frame.

Four types of self-improvement exist in current AI research: self-reflection, self-generated data, self-adapting models, and self-improving code agents. The patterns on this page combine type 1 (reflection through reviewer feedback) with type 4 (rule updates from session experience). No weights change at any stage.

## Five levels of memory

Claude Code operates five concurrent memory systems with different scopes and time horizons. No level replaces another. Each serves a different TTL and granularity.

| Level | What | Scope | Automated? |
|-------|------|-------|-----------|
| L1 | Static rules, always loaded | Permanent | Manual (CLAUDE.md) |
| L2 | Path-scoped rules (`paths:` frontmatter) | Permanent, scoped | Manual (.claude/rules/) |
| L3 | Cross-session facts, derivability-filtered | Days to weeks | Auto (extractMemories) |
| L4 | Within-session structured notes | One session | Auto (Session Memory) |
| L5 | Periodic consolidation | → L3 | Auto (autoDream) |

All five background processes fire at startup and run fire-and-forget off the main thread: `initMagicDocs + initSkillImprovement + initExtractMemories + initAutoDream + autoUpdateMarketplacesAndPlugins`. None block the conversation.

### L3: extractMemories

After each session, a forked agent reads the conversation history and writes structured facts into four buckets. *User*: role, goals, preferences. *Feedback*: corrections and confirmations. *Project*: ongoing work state. *Reference*: pointers to external systems.

A derivability test filters noise. Any fact re-derivable from the codebase itself doesn't need storage. Git history is authoritative for "we added a payment module." Only experience-dependent knowledge passes through: the developer's preference for flat error handling, the team convention that broke in production, the specific sequence that causes CI to time out.

Confirmations count as signal alongside corrections. The quiet "yes, exactly" that accepts an unusual approach carries the same evidential weight as explicit pushback. Recording only failures biases the agent away from validated approaches it might otherwise abandon on a future task.

The fork shares the main conversation's prompt cache. Extraction is cheap by design.

### L4: session memory and compact protection

Session Memory tracks nine sections across a long conversation: Current State (marked CRITICAL), task spec, files in scope, workflow, errors and corrections, codebase docs, learnings, and a worklog. It triggers at 10K tokens with at least three tool calls, or when the session grows 5K tokens without a tool call. Budget: 12,000 tokens total, 2,000 per section.

Compaction is not memory loss. Three mechanisms protect state: Session Memory injects its summary into the compact transcript, all user messages survive verbatim, and a microcompact pass cleans only tool results (triggers at 180K tokens, keeps 40K). A `PreCompact` hook is the explicit save point. Critical pipeline state written there survives any compaction. Adding `NO_TOOLS_PREAMBLE` as the first line prevents tool calls during the compact itself.

### L5: autoDream

Every 24 hours (after at least five sessions) a forked read-only agent runs consolidation. Four phases: Orient (read MEMORY.md), Gather (recent logs plus drifted memories), Consolidate (merge entries, convert relative dates to absolute), Prune (enforce < 200 lines and < 25KB). A file-based mutex with PID ownership prevents concurrent runs.

The pruning criterion is size, not value. That's a gap. Entries removed by line count may carry more signal than entries retained by recency. Score-based pruning (which the ACE framework addresses) is a more principled approach. [See memory scoring below.](#memory-scoring)

## skillImprovement: rules from corrections

Every five user messages, a background hook scans the recent conversation for behavioral corrections. Requests to add or remove steps, preferences about how things should work, explicit pushback. These are the signals. When corrections are found, it proposes a `SkillUpdate[]` with the section to change, what to change, and why, then calls an LLM to rewrite the active SKILL.md in-place.

The scope is narrow by design: only the currently active skill, only messages since the last check. The rewrite is surgical. Not a full regeneration, but targeted updates to sections that received corrections.

The gap is symmetric. Silent failures pass through unnoticed. The hook only captures explicit corrections. A verification stage that surfaces failures the user never flagged closes this gap. [See Verification](./verification.md).

A companion mechanism (the Remember skill) periodically reviews auto-extracted memories and proposes promotions to permanent rules. Destination options: CLAUDE.md (project conventions), CLAUDE.local.md (personal preferences), team memory (cross-repo shared), or auto-memory (keep ephemeral). All proposals are presented before any changes apply.

### Magic Docs: knowledge that updates itself

Any markdown file prefixed with `# MAGIC DOC: [title]` rewrites itself after being read. A background forked subagent updates information in-place. Not as a changelog appended to the bottom, but editing content where it lives. Italic text after the header provides custom instructions: what to prioritize, what to preserve, how aggressively to update.

The pattern extends beyond agent preferences. A project architecture doc that refreshes each time the scout reads it. A dependency map that adjusts when packages change. A test coverage summary that updates after each run. The implementation is a fork of a cheap model and a single header convention. The result is documentation that ages more slowly because it self-corrects.

## ExpeL: gather → extract → apply

The closed-loop pattern was formalized in ExpeL. Three stages.

**Gather.** The agent solves tasks and saves full trajectories, including failed attempts. A trajectory with three wrong approaches before a correct solution carries information a clean success doesn't: what the agent tried, what broke, why the final approach worked.

**Extract.** An LLM pass over trajectory batches finds patterns: approaches that consistently fail, approaches that succeed, failure modes repeating across different task types. Output is natural-language rules. Not code, not embeddings. Text the agent reasons about directly.

**Apply.** Future task runs load extracted insights and relevant past trajectories as few-shot context. The insight library grows with each run.

Performance compounds as the library grows, not degrades. On HotpotQA and ALFWorld, ExpeL agents improve with each batch of trajectories. The gain comes from pattern recognition across tasks. Not single-task reflection, which saturates quickly.

Our pipeline implements a production adaptation. After each feature merge, a dedicated lesson-extractor reviews what broke, what was unexpectedly hard, and what assumption was wrong. After the 24-file type refactoring in Experiment 3, the extractor produced two rules: "grep for barrel imports when moving types" and "check wildcard re-export conflicts when splitting shared modules." Both matched exactly the failure categories that caused the reviewer to fail three times before passing. Those rules now load into future pipeline runs automatically. [Full pipeline details](./pipeline.md).

## Extraction ≠ promotion

Knowing what happened this session is not the same as knowing what generalizes across sessions. These are different jobs that require different model quality. Running them with the same model is a false economy.

Per-session extraction (what failed, what was decided, what needs to be remembered) is cheap. A Haiku-class model handles it reliably. Cross-session synthesis (recognizing that a failure in authentication implies a rule applicable to async middleware generally) is expensive. It requires the strongest model available. Claude Code makes this concrete: `extractMemories` runs on whatever model is currently active; `/insights`, which analyzes all sessions and generates `claude_md_additions`, routes explicitly to Opus.

The `/insights` command surfaces repetition across the full history. An instruction appearing in two or more sessions is a "PRIME candidate" for promotion to CLAUDE.md. Friction patterns (misunderstood requests, wrong approaches, rejected actions, excessive changes) accumulate into behavioral signals that no single session can surface alone.

Cheap promotion produces rule pollution. A weaker model extracting cross-session generalizations makes plausible mistakes: rules that hold in one context, fail in adjacent ones, and accumulate without demotion. The defense is aggressive demotion — rules with sub-50% success over ten applications should be removed, not just deprioritized.

Extract liberally within sessions. Promote conservatively across sessions. Demote as aggressively as you promote.

This is [Principle 6](./principles.md). The one principle that touches all the others, because without the feedback loop, none of the others compound.

## Practitioner patterns

Self-improvement moved from research into production across 2025-2026. Three deployments show what the pattern looks like at different scales and from different angles.

**Structured error logging.** Rory Teehan's system captures three fields per error: what happened, why it happened, what should have happened instead. When the same pattern appears three or more times, a rule synthesizes and promotes automatically. After two weeks: 13 error patterns became 13 active rules, backed by 211 indexed memories. The notable observation: learned rules carry more weight than initial static instructions. They're treated as empirically validated rather than author-asserted.

**Extraction as end-of-session discipline.** A four-phase wrap-up skill enforces the extraction step that's easiest to skip. Ship (commit and push) → Remember (review lessons, promote to memory) → Review (flag mistakes worth extracting) → Publish. The extraction moment (not the session itself) creates persistent improvement. Without a deliberate end-of-session step, lessons evaporate with context. The practitioner reports running this without exception every single session.

**Four-file memory with drift detection.** Separating lessons, errors, decisions, and todos into distinct files prevents category pollution. Across 100 sessions on a production Java codebase: by session 20 the agent knows your coding patterns; by session 50, it knows the codebase better than a new hire would. The `/evolve` command patches a gap in `skillImprovement.ts`. Instead of requiring the skill to be currently active, it runs retrospectively against any skill with recorded failures.

## Self-improving tools

**pi-autoresearch** (2,288 stars) implements Karpathy's autonomous research loop as a Pi extension. The pattern: run an experiment, feed results to the next run, iterate without human intervention per cycle. The chess example: starting at expert ELO, reaching top-50 grandmaster through 70+ sequential autonomous experiments.

**chudi.dev's self-improving RAG** wires persistent memory into the debugging loop, not just planning. Each resolved bug becomes a retrieval entry; when a similar failure appears later, the fix surfaces automatically. The debugging history functions as the training set, growing continuously without a separate curation step.

**skill-loop** runs a four-stage programmatic cycle against any skill file: observe → inspect → amend → evaluate. Exposed via MCP, so improvement can trigger automatically on any event rather than waiting for session end.

Three independent self-improvement projects (`skill-loop`, `selfwrite`, and a third loop-based tool) shipped in the same week in March 2026 without coordination. Each arrived at the same closed-loop architecture independently. That convergence is its own data point.

## Memory scoring

Memory becomes self-correcting when each unit carries a score that updates on use. The ACE framework (Agentic Context Engineering) formalizes this with hybrid retrieval: embedding similarity plus effectiveness score plus freshness determines what surfaces. Not just recency or keyword match. Grow-and-refine cycles apply semantic dedup (cosine ≥ 0.85 = likely duplicate), prune negative-score entries, and enforce section size limits.

Our KB implements a simplified version. Each bullet accepts helpful/harmful feedback, retrieval priority tracks effectiveness, and `research_refine` prunes entries with persistently negative scores. Episode weights are asymmetric: +0.3 on success, −0.5 on failure. One misleading memory can cascade into multiple wrong decisions. The penalty exceeds the reward deliberately.

Output quality saturates at roughly seven governed memories per entity. Adding more provides no measurable benefit, confirmed across 500 adversarial queries with 99.6% fact recall, 50% token reduction from progressive delivery, and zero cross-entity leakage. The ceiling is low. Returns come from the feedback loop and scoring quality, not from storing more entries.

A separate production deployment ran 232 tasks with zero failures using four components: extraction daemon polling logs, Haiku extracting facts, a markdown vault with vector search, and an injection hook surfacing the top-2 relevant entries per prompt. Simple infrastructure. Working loop.

## Open questions

**Extraction timing.** We run extraction at session end in batch. Inline extraction after each tool call would capture silent failures that resolve before the user notices. The signal exists but never gets logged. The tradeoff: latency on every tool call, plus a live-context extraction prompt rather than a session summary. No measurement exists for signal gain vs latency cost. The breakeven point is unknown.

**Rule conflict resolution.** Two promoted rules can contradict each other for the same situation. Current implementations default to recency. Newer rule wins. Confidence-weighted resolution is conceptually clear: a rule with 90% success over 30 applications should beat a newer rule with three. The unsolved case is two rules with equal confidence and opposite recommendations. No implementation handles this reliably.

**Cross-agent rule transfer.** Rules extracted by one agent instance. Do they transfer to a second agent on the same codebase without encoding agent-specific context artifacts? The file format is portable; whether the generalizations hold is unknown. No published transfer success rates exist.

**Automatic promotion outside Claude Code.** The `/insights` mechanism is readable in CC source: Opus model, cross-session aggregation, frequency threshold of 2+ sessions as the PRIME candidate signal. Replicating it independently requires session history in a compatible format and a coordination mechanism to prevent duplicate promotions across concurrent sessions. The threshold is clear. The plumbing (for tools other than CC) is not.
