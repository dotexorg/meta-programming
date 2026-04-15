# Pipeline: Scout → Spec → Plan → Workers → Review → Lessons

A structured workflow outperforms adding more information. By a measurable margin. Six stages, clear role separation, and deliberate cache strategy are what make multi-agent pipelines work in production.

## Why process beats context

Giving an agent more context when it gets something wrong is the natural instinct. It is also wrong.

We ran five variants on the same production EventBus refactor (509 TypeScript files, DDD architecture, Cloudflare Workers) to measure what actually drives correctness.

| Run | Setup | Cost | Architecture | Review |
|-----|-------|------|-------------|--------|
| #1 | Raw prompt + code map | $2.84 | ❌ Wrong | — |
| #2 | Raw prompt, no map | $8.45 | ✅ Correct | — |
| #3 | Detailed spec + code map | $9.17 | ✅ Correct | — |
| #4 | **Pipeline, no map** | **$6.63** | **✅ Correct** | **Pass, first try** |
| #5 | Pipeline + code map | $9.99 | ✅ Correct | Fail → fix |

The pipeline run (#4) was the cheapest correct solution and the only one that passed review on the first attempt. Adding a code map to that same pipeline (#5) cost 51% more and introduced a review failure. The map gave the agent a fast path to the highest-ranked files, which meant it stopped exploring and missed scope. Without the map, it grepped broadly, found the full picture, and built accordingly. [See Experiments](./experiments.md#experiment-1-process-vs-context).

Auto-generated context reduces success by 3% and increases cost by 20%; human-written boundaries improve success by 4%, across 138 real-world tasks and three models. Same mechanism at a different scale: pre-loaded navigation shortcuts substitute for actual exploration.

The pipeline adds roughly 10× the latency of a raw prompt for simple changes. But trace a full week of work: two raw failures per feature, one pipeline success. The pipeline reaches working code faster. The overhead isn't in the pipeline. It's in the failures it prevents.

## The pipeline: what each stage does

The six stages map onto a research-to-delivery arc. Not every task needs all of them.

| Task | Minimum stages |
|------|---------------|
| Bug fix, known cause | Worker → Pre-flight |
| Single-file change | Spec → Worker → Review |
| Multi-file feature | Spec → Plan → Workers → Review |
| Cross-domain refactor | Scout → Spec → Plan → Workers → Review → Lessons |

**Scout.** A lightweight agent (Haiku-class) reads the codebase and writes a structured briefing. It never touches anything. Its job is to compress signal so downstream workers don't pay full-context prices for orientation. In our architecture, scout output is cached by git commit hash, so if nothing changed, the briefing is free.

**Spec.** A structured specification is written before any implementation begins. In one tech-lead run, a clarifying question during spec review revealed the feature being designed was already configured in the system. What looked like a multi-day implementation reduced to a one-line config change. Without spec review, a raw agent on the same task implemented the entire feature from scratch, duplicating existing work. [See Specification](./specification.md) for structure.

**Plan.** The spec is decomposed into atomic, sequenced tasks with explicit acceptance criteria. Ordering matters: dependencies become explicit, so workers don't block each other and parallelism opportunities are visible upfront.

**Workers.** Individual subagents execute tasks in isolation. Context does not bleed between workers: a Task 6 that ran 106 turns did not degrade Task 7 in one pipeline run, because Task 7 started clean. This isolation is the mechanism that makes sequential multi-task work reliable at scale.

**Review.** A separate agent with no implementation history reads the output and checks it against the spec. Review and implementation sharing context is the most common pipeline mistake. Self-evaluation bias is real. Agents rate their own work highly even when it is broken (Anthropic, 2026). A clean reviewer is an honest reviewer. [See Verification](./verification.md) for layered pre-flight requirements.

**Lessons.** After merge, a final agent extracts what changed, what failed, and what patterns should persist. Per-session extraction is cheap enough for a Haiku-class model. Cross-session promotion (deciding which patterns generalize) needs a stronger model; cheap promotion produces rule pollution. This stage closes the loop: execution experience becomes structured knowledge that changes how future workers reason. [See Self-Improvement](./self-improvement.md).

This parallels Claude Code's Coordinator Mode (Research → Synthesis → Implementation → Verification). The difference: CC runs these as in-process phase shifts within a long session; our pipeline runs each as a separate process with a context reset. Both work. Clean processes are simpler to debug.

## Orchestrator never reads files

The orchestrator delegates. It does not explore.

If the orchestrator's messages contain file contents, something is wrong. It should route work: task descriptions, dependencies, acceptance criteria. Nothing else.

In our architecture the orchestrator is a skill running in the parent process; workers are separate agent processes. Pi's subagent tool is unavailable inside another subagent, so orchestration must happen from the top level. We learned this the hard way. tech-lead couldn't delegate to scout until we restructured it as a skill.

An orchestrator that reads files defeats itself. It fills context with details that belong to workers. It becomes a sequential bottleneck. Every read is a turn spent on work that should be parallelized.

This rule degrades under pressure. One debugging session: the orchestrator called scout five times (the skill says "NEVER call scout twice"), then started reading files directly. Caught mid-session. The pattern is consistent, when delegation can't solve the problem, the orchestrator reverts to direct exploration. Sometimes that's correct. Novel debugging may need human collaboration, not longer chains.

## Coordinator patterns: continue vs spawn fresh

The continue-vs-spawn decision is the most consequential call the coordinator makes.

**Continue** (fork the parent) works when the next task is a continuation of the same artifact: reviewing what the parent just wrote, extracting memory from a completed session, running a second pass on the same output. Fork shares context cheaply and maintains narrative continuity.

**Spawn fresh** (clean agent) works when the next task is independent: a worker in a parallel batch, a verification pass that needs fresh eyes, any task that should not be influenced by prior context. Clean context is a feature, not a cost.

The failure modes are symmetric. A fork that should have been clean carries invisible priors. The worker "knows" things from the parent's exploration that bias its approach. A clean agent doing continuation work has to rediscover everything the parent already built.

Model selection per stage is part of the coordinator decision, not an afterthought. A pipeline run with Haiku scouts and Sonnet workers cost $4.77 total and found the root cause. A solo Sonnet session on the same task (no pipeline structure) burned $6.66+ and failed, looping on wrong hypotheses for ~86K tokens before the user intervened. The savings came from the structure, not just the cheaper scouts.

## Fork vs clean agent: when to share context, when to start fresh

A fork is not a copy. It's a continuation from a shared prefix. When you fork an agent, the child receives a byte-identical system prompt, tool list, and model assignment. Specialization happens through the user-message directive only. Everything before the fork point is shared state.

A clean agent starts cold. No inherited system prompt, no tool list, no prior turns. It pays more to orient, but it cannot be contaminated.

| Task type | Agent type |
|---|---|
| Review parent's work | Fork |
| Extract memory from session | Fork |
| Independent worker, parallel batch | Clean |
| Open-ended exploration | Clean |
| Verification pass | Clean |

Contamination is the more dangerous direction. A fork carrying stale priors solves the wrong problem confidently. A clean agent doing continuation work signals its ignorance through questions. Visible. Correctable. When uncertain, default to clean.

## Fork cache mechanics

Forking runs on Anthropic's prompt cache. The child's first request shares a byte-identical prefix with the parent. Same system prompt, same prior turns. The API matches, returns `cache_read`, skips reprocessing. On 100K+ tokens of context, the fork's first call costs ~10% of cold start.

Claude Code's fork implementation preserves this by design: all children receive identical placeholder tool results across every fork, and only the final user-message directive differs per worker. This is what makes multi-worker fan-outs economically viable. The expensive prefix is paid once by the parent and shared by all children.

The condition is strict. Byte-identical. System prompt, tool definitions, model. All must match exactly. One character of whitespace? Full reprocessing. Different model on a fork? Cache gone. This is why standardizing prompts across pipeline stages isn't style. It's economics.

A long-running orchestrator keeps the cache warm. Idle time between dispatches isn't waste. It's the shared prefix that makes the next fork cheap.

## Subagent economics

A skill runs in the orchestrator's process, shares its cache, and pays zero spawn overhead. An agent is a separate process: it starts cold, reloads tools, and rebuilds context from scratch.

The cache math is concrete. Fourteen subagents processing 50K tokens each generate 700K `cache_create` tokens and zero `cache_read`. The same work done in one long session with context sharing produces an 11:1 read-to-create ratio. The parallel workload costs an order of magnitude more in cache terms than the sequential session, before accounting for spawn overhead.

This is not an argument against subagents. Isolation and parallelism justify the cost for complex tasks. But for lightweight, repeatable operations (search, format, summarize), a skill is almost always the right choice.

The decision:

- Task needs isolation, is long-running, or risks runaway token consumption → Agent
- Task is lightweight and repeatable → Skill
- Task is a natural continuation of the parent → Fork

Verification is often where the model breaks down, not execution. Augment Code's single-writer rule for hotspot files and sequential merge strategy reflects this: when verification is the bottleneck, adding more parallel execution makes things worse, not better. Azure's multi-agent taxonomy names five patterns (Sequential, Concurrent, Group Chat, Handoff, Magentic) and applies the same rule across all of them: use the minimum complexity that solves the problem reliably.

## Settled questions

**Pre-flight is a permanent fixture.** In [Experiment 3](./experiments.md#experiment-3-pipeline-end-to-end), review agents failed on first attempt every time. Zero first-attempt pass rate. Both failures were deterministically checkable: missing imports, type errors. Adding `tsc --noEmit` before the review agent ran eliminated both. Pre-flight is now part of every pipeline run. The remaining scope question: in [Experiment 5](./experiments.md#experiment-5-edit-tool-bottleneck), `tsc` passed on code with a runtime logic error. A `||` operator split across lines by the edit tool, invisible to the compiler. Deterministic pattern checks for common edit-tool artifacts are candidates for additional pre-flight steps; false-positive rate is unmeasured.

## Open questions

- **Human-in-loop placement.** One debugging session: the breakthrough came from the user asking a direct question. Not from delegation. Three wrong hypotheses. User intervened. Fixed. We have no pattern for "agent requests human input at step N" that doesn't break autonomous flow. Where should the intervention threshold sit? No data.

- **Pipeline crash recovery.** Worker dies mid-task, progress is lost. Context reset between workers is a feature. But it means no checkpoint within a task. CC's PreCompact/PostCompact hooks address orchestrator memory loss, not worker crashes. Checkpointing is undesigned. The experiment: write worker state to `.pi/pipeline-state/` after each phase, test recovery from simulated kills.
