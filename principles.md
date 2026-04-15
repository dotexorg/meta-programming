# 6 principles + how to apply them

Six principles extracted from A/B tests and production failures. Three maturity levels show which to implement first.

## The 6 principles

### 1. Process > context

Structured workflow beats more information. Adding code maps, background files, or richer context doesn't reliably improve results. Process does. 🟢

The proof is direct. In a 5-run A/B test refactoring a production EventBus (509 TypeScript files, 53K lines, DDD architecture), the winning configuration wasn't the most-informed one. Tech-lead pipeline without a code map: $6.63, correct architecture, first-try review pass. Same pipeline with a code map added: $9.99, failure requiring a fix. The map cost $3.36 extra and introduced a failure. 🟢 ETH Zurich independently tested the same phenomenon across 138 real-world tasks and three models: auto-generated context files reduced success by 3% and increased cost by 20%. Human-written boundaries improved success by 4%. 🟡

The mechanism: code maps give the agent a fast path through the codebase. It follows the path instead of exploring. When the map doesn't capture the full picture (it never fully does) the agent misses scope. SWE-bench confirms the broader point: a custom task harness adds 10% over standard scaffolding with the same underlying model. Same model, different process. 🟡

See [Pipeline](./pipeline.md) for the full workflow.

### 2. Boundaries are not optional

Without explicit constraints, agents fail identically to raw prompts. Not slower. Not more expensively. Identically. 🟢

In a direct ablation (Telegram bot feature addition), two consecutive failures without a spec. First run corrupted existing prompt handling; second misunderstood the word "reply." Both failures were deterministic. They'd have happened every time. Adding `DO NOT modify classifyTicket` to the spec prevented both. Three lines eliminated two distinct failure types. 🟢

The minimum viable spec has three parts: DO (what to build + acceptance criteria), DO NOT (files and patterns to leave alone), GLOSSARY (any term with room for interpretation, e.g., "fast" = < 200ms p95). The industry has converged on this structure: spec-kit (79K GitHub stars), Kiro, and Tessl all enforce some variant of it. 🟠

See [Specification](./specification.md).

### 3. Exploration vs exploitation

Navigational hints hurt. Requirements don't. The distinction matters. 🟢

Telling an agent where to look ("start with grep X, check file Y") narrows the search space before the agent understands the problem. It follows the path, misses adjacent context, builds to the path rather than the problem. Telling it what to build, why, and what to leave alone gives it room to discover scope. The codemap result in Experiment 1 is direct evidence: without the map, the agent grepped broadly, found the full picture, produced correct architecture. With the map, it took the fast path and missed scope. 🟢⚪

ETH Zurich measured the same pattern: auto-generated context files reduced task success by 3% across 138 tasks. Small delta that compounds when the task requires finding non-obvious solutions. 🟡

Claude Code handles this structurally via separate agent types with different tool permissions. The explore agent can't call edit tools. Our formulation is explicit rather than structural, which makes it portable. 🟡

### 4. Separate builder from reviewer

Agents praise their own work. This isn't a model quality issue. It's a structural property of single-agent review. 🟢

Anthropic documents "self-evaluation bias" explicitly. In a 133-cycle study across four models (BSWEN), each model had different blind spots: GPT caught security issues and non-idiomatic patterns; Claude caught architectural problems. A race condition surfaced only under multi-model review. Invisible to both models reviewing their own output. 🟡 The failure mode runs deeper than generous self-assessment. Higher thinking levels produce a soft sycophancy pattern: the model formally states it won't violate a rule, then implements the violation framed as a "workaround". Passing a surface check while substantively failing (Experiment 8). 🟢

The scale of the consequence: Amazon deployed 21K agents with 80% team adoption in 90 days, then hit 4 Sev-1 incidents in the following 90 days, including a 6-hour outage. Velocity scaled faster than verification. 🟠 Verification isn't optional overhead. It's what keeps this from happening.

See [Verification](./verification.md).

### 5. Atomic tasks with context reset

One task → verify → commit → reset → next. No exceptions. 🟢

A 106-turn task in Experiment 3 didn't contaminate subsequent tasks. Because we reset between them. Without the reset, errors that slip through review compound in later tasks; the agent tries to correct them and often makes things worse. 🟢 Effective context degrades well before the nominal limit: complex reasoning can collapse at 10–40% utilization depending on task type (MECW, Chroma 2025; confirmed by practitioners at Sentry and in production 1M-context sessions). 🟡 Long sessions don't get more done. They get noisier.

Context reset also defends against provider-side model degradation. When model quality drops silently (as in the April 2026 Opus incident (-67% thinking depth, 80× cost increase), an externalized plan written to disk keeps the task anchored even as the model's own reasoning deteriorates (Experiment 9). 🟢

Tasks spanning 30+ files should be split into 2–3 subtasks before starting, not mid-session. 🟢⚪

### 6. Extraction ≠ promotion

Per-session extraction and cross-session promotion are different operations with different cognitive demands. Running them on the same model level pollutes the knowledge base. 🟡

Extraction asks: what specifically happened in this session? Factual summarization. A cheap model handles it well. Promotion asks: what generalizable rule does this represent? Synthesis from specific cases to general patterns correlates directly with model capability (OpenReview 2025). 🟡 Run promotion with a weak model and the KB fills with case-specific noise masquerading as universal rules.

Claude Code separates them architecturally: `extractMemories` uses whatever model is running; `/insights` (cross-session promotion) routes to Opus. The split reflects the actual cognitive load of each task. 🟡

See [Self-Improvement](./self-improvement.md).

## Level 1: minimum viable process (day one)

Three habits that eliminate most failures before you build any pipeline.

**DO / DO NOT / GLOSSARY on every task.** Three lines before the prompt. DO: what to build + when it's done. DO NOT: which files, APIs, and patterns to leave untouched. GLOSSARY: any term that could be interpreted two ways. In the ablation, both consecutive failures traced to missing one of these three. 🟢 Don't write paragraphs. Write three lines.

**Commands, not prose.** "We value clean code" produces zero behavior change. `run: npm test -- --coverage && assert coverage > 80%` works. Across 10+ runs per pattern, prose instructions don't move the needle; executable commands do. 🟠 Every behavioral requirement goes as a verifiable command or condition.

**Specify WHAT, not WHERE.** No "start with grep X" or "look at file Y." Describe the outcome; leave the path to the agent. Navigational hints trigger exploitation over exploration. The agent follows the path instead of finding the right one (Principle 3). 🟢

## Level 2: pipeline (first week)

Separate roles, sequential execution, context reset between tasks.

**Scout reads, workers implement.** One exploration pass before any implementation. The scout reads the codebase, writes a structured report, caches it by git commit hash. Workers get the task plus the scout report. Not the full codebase. The orchestrator never reads files directly. 🟢

**Deterministic checks before LLM review.** `tsc`, `npm test`, linter. Run these first. They catch whole classes of errors without burning tokens. But they don't catch logic bugs. 🟢 In Experiment 5, `tsc` passed on code with a corrupted `JIRA_SERVICE_USER` assignment. A runtime logic error invisible to the type checker. Deterministic checks are necessary, not sufficient.

**Builder and reviewer in separate sessions.** Builder commits and stops. Reviewer starts fresh with no prior context. Ideally different models. GPT reviewing Claude's output catches failures Claude self-review misses, and vice versa. 🟢🟡

**One task, one commit, one reset.** Verify after every task. Commit. Start the next in a clean context. The reset is structural, not optional. 🟢

## Level 3: learning loop (first month)

The agent improves without fine-tuning. Most teams skip this layer entirely.

**Extract lessons after every feature.** End of session: what broke, what was unexpectedly hard, what assumption was wrong. Write it down. A cheap model (Sonnet, Haiku) handles this. It's factual summarization. Don't promote yet. 🟢🟡

**Promote carefully with a strong model.** Cross-session promotion (turning session lessons into general rules) requires judgment. Use the strongest available model. Cheap promotion produces rules that sound universal but are actually case-specific. Over time, this fills your spec files with noise that future sessions inherit and can't easily clean up. 🟡

**Spec before code, always.** In Experiment 4, the agent proposed a barrel re-export strategy during spec review. Asked to explain it, the agent recognized it as an anti-pattern and removed it. Without spec review, that decision would have been built, merged, and lived in production. Spec is the cheapest place to catch errors. Nothing has been written yet. 🟢

**KB as working memory, not documentation.** In a direct A/B (Experiment 6), a generic Sonnet answered "give it a code map" when asked how to handle a complex refactor. The same model with an 8KB structured KB answered "careful, exploration vs exploitation paradox" and cited the tradeoff with evidence. The difference wasn't code quality. It was thinking level. 🟢

## Anti-patterns

Five failure modes with measured consequences.

**"Make it good."** No DO, no DO NOT, no GLOSSARY. Fails identically to a raw prompt. Two consecutive deterministic failures in Experiment 2. Model capability doesn't matter; boundaries are load-bearing, and removing them collapses the structure. 🟢

**Code map as default context.** Experiment 1 run #5 (with codemap): $9.99, failure. Run #4 (same pipeline, no map): $6.63, first-try pass. The map added $3.36 and introduced a failure. Use maps for orientation. Remove them for implementation. 🟢

**Self-review.** Agents rate their own broken code as excellent. A separate session with fresh context catches errors the implementation session misses, even with the same model. With a different model, coverage is better. Neither is optional; both are better than the same session reviewing itself. 🟢

**Monster tasks.** 50+ files in one session don't get refactored. They get half-refactored, with the rest hallucinated. Decompose until each task has a verifiable output and a deterministic acceptance test. If you can't write the acceptance test, the task is still too large. 🟢⚪

**Trust without verification.** Amazon's 6-hour outage (21K agents, 4 Sev-1 incidents in 90 days) traced to velocity scaling faster than verification. 🟠 LinearB measured the baseline risk across 8.1M PRs: AI-generated code receives 1.7× more review revisions than human code. 🟡 Verification isn't process overhead. It's what keeps scale from compounding errors.

## Where we go beyond CC

Claude Code answers "how to build an agent system." These principles answer a different question: how to use one effectively. The two aren't in conflict, but they're not the same conversation.

| Dimension | Claude Code | Our contribution |
|---|---|---|
| Framework name | LMP implicit (`skillImprovement.ts`, `extractMemories`) | Named, 3 layers defined, empirically tested |
| Process design | Coordinator mode. One built-in pattern | A/B tested 5 configurations, proved Process > Context |
| Exploration vs exploitation | Structural (agent types with different tool permissions) | Explicit principle, portable across tools and providers |
| KB as working memory | Observations about user and project | Structured research that changes how the agent reasons |
| Multi-model review | Anthropic-only | Cross-model review (GPT reviewing Claude) as explicit pattern |
| Multi-provider | Anthropic API only | Principles generalize to Gemini, GPT, Ollama |

CC's production engineering is solid. But production-proven isn't optimal. Memory consolidation prunes by file size and line count rather than by relevance or quality. A different design choice than our bullet system with scored retrieval. Hooks exist but mostly handle permissions, not quality gates. There's no formalized guidance on when to use coordinator mode vs fork vs a clean agent spawn. That meta-layer is where these principles live. 🟡⚪

The honest summary: use CC (or any capable agent tool) for execution. Use these principles to design the process around it.

## Open questions

**Automated degradation detection.** The April 2026 Opus incident showed silent provider-side quality changes: thinking depth dropped 67%, costs spiked 80×, stop-hook violations went from 0 to 173. All while the same CLAUDE.md was in context. We have `trajectory.py` and the relevant metrics (read:edit ratio, thinking depth, retry count). What doesn't exist: automated alerting when metrics cross thresholds, so degradation gets caught in real time rather than discovered after it damages output. This blocks shipping a reliable pipeline in production without manual monitoring. 🟢

**Layer 2 intent language.** The principles above express WHAT and WHY in natural language. The next layer is machine-verifiable intent. Not "don't modify classifyTicket" as a text rule, but as an executable check that confirms compliance post-task. ContextCov (U Washington, Feb 2026) turns passive AGENTS.md into active executable checks; IntentLang formalizes intent as 7 structured elements with an XML intermediate representation. The external theory is mapped. Our own implementation doesn't exist yet. This blocks fully closing the loop described in [Self-Improvement](./self-improvement.md). 🟡
