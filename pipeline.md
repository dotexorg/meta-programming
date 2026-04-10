# Pipeline

A structured workflow consistently outperforms throwing more context at a problem. This page details the six-stage pipeline, how orchestrators and workers divide responsibility, and the economic mechanics of forking versus spawning fresh agents.

## Why Process Beats Context

The intuition that "more information → better output" is wrong at scale. In a 5-run pilot on the same production task, a structured pipeline produced a first-attempt pass at $6.63. Adding a full codemap cost $9.99 and still failed review. Raw generation without review came in at $8.45 — correct, but unverified. The full data: [see Experiments](./experiments.md#experiment-1-process-vs-context). 🟢

ETH Zurich confirmed the same pattern independently across 138 real-world tasks: auto-generated context reduced success by 3%, while human-written boundaries improved it by 4%. 🟡

The structured pipeline won on all three dimensions: cost, correctness, and confidence. The lesson is not that context is useless, but that *how you sequence work* determines whether that context gets used well.

A full pipeline run on a larger task — 24 files refactored into 7 domain modules — produced 8 commits, 253 tests, and 0 regressions for approximately $5.50 🟢. Review agents had a 0% first-attempt pass rate: both failures were deterministically checkable (missing imports, type errors). Adding a pre-flight check before AI review eliminated the problem and became a permanent fixture. [See Experiments](./experiments.md#experiment-3-pipeline-end-to-end).


## The Six Stages

The pipeline maps cleanly onto a research-to-delivery arc:

**Scout** — a lightweight agent (Haiku-class) reads the codebase, identifies relevant files, and produces a focused briefing. It never writes anything. Its job is to compress signal so downstream stages don't pay full-context prices for orientation.

**Spec** — a structured specification is written before any implementation begins. In one tech-lead run, a clarifying question during spec review revealed the feature being designed was already configured in the system — the agent asked "should I check if webhook routing already exists?" and it did. The entire workstream was cancelled before a line of code was written. 🟢 [See Specification](./specification.md) for how specs are structured.

**Plan** — the spec is decomposed into atomic, sequenced tasks with explicit acceptance criteria. Task ordering matters: dependencies are made explicit so workers don't block each other and so parallelism opportunities are visible upfront.

**Workers** — individual subagents execute tasks in isolation. Context does not bleed between workers: in one run, a Task 6 that ran 106 turns did not degrade Task 7, which started clean 🟡. This is the core mechanism that makes large-scale parallel work reliable.

**Review** — a separate agent with no implementation history reads the output and verifies it against the spec. A reviewer with shared context is compromised; a reviewer with clean context is honest. [See Verification](./verification.md) for review patterns and pre-flight requirements.

**Lessons** — after merge, a final agent extracts what changed, what failed, and what rules should be updated. This closes the loop from execution back into process. [See Self-Improvement](./self-improvement.md).

This structure parallels what Claude Code's internal Coordinator Mode calls Research → Synthesis → Implementation → Verification (CC Ch.10). The important difference: our pipeline runs each stage in a separate process with a context reset, rather than as an in-process memory shift within a single long session. 🟢

**Not every task needs the full pipeline.** The six stages above are the maximum — for multi-file features that touch multiple domains. Most work needs a subset:

| Task | Stages needed |
|------|---------------|
| Bug fix with known cause | Worker → Review |
| Single-file feature | Spec → Worker → Review |
| Multi-file feature, one domain | Spec → Plan → Workers → Review |
| Cross-domain refactor | Scout → Spec → Plan → Workers → Review → Lessons |

The rule: use the minimum pipeline that solves the problem. Every stage adds latency, cost, and coordination surface. A scout is pointless if you already know which files are involved. A spec is overkill for a typo fix. Lessons extraction only pays off if the task was novel enough to learn from. Harrison Chase frames this as the same tradeoff: "simple task → minimal harness, complex workflow → batteries-included pipeline."



## The Orchestrator Rule

The orchestrator never reads files. It delegates everything.

This is not a style preference — it's an architectural constraint. In the pi subagent model, the orchestrator is a *skill* (running inside the parent process), and executors are *agents* (separate processes). The subagent tool is unavailable inside another subagent, which means orchestration must happen from the top level 🟢 (discovered through failure).

An orchestrator that reads files defeats itself in two ways. First, it accumulates context that belongs to workers, crowding out its coordination capacity. Second, it becomes a bottleneck: every file read is a sequential operation in the one process that should stay free for routing decisions.

The practical rule: if the orchestrator's message contains file contents, something is wrong. It should contain task descriptions, dependencies, and acceptance criteria — nothing else. See [Principles](./principles.md) §1 and §3.


## Coordinator Patterns

Not every task warrants a fresh agent. The continue-vs-spawn decision is the most consequential call the coordinator makes.

**Continue** (fork the parent) is correct when the next task is a *continuation* — reviewing what the parent just did, extracting memory from a completed session, or running a second pass on the same artifact. The fork shares context cheaply and maintains narrative continuity.

**Spawn** (clean agent) is correct when the next task is *independent* — a worker in a parallel batch, an exploration on a different part of the codebase, or any task that should not be influenced by what came before. Clean context is not a cost; it's a feature.

The failure mode is spawning clean agents for continuation tasks (they lack the context they need) or forking for independent tasks (they inherit context that poisons their reasoning). The decision should be made explicitly per task, not set once for the whole pipeline.

Model selection per stage matters. In one pipeline run, Opus as orchestrator + Haiku for scouts + Sonnet for workers cost $2.50 total and passed review. The same task with Sonnet at every stage cost $6.66 and failed review — Sonnet as orchestrator made worse decomposition decisions that cascaded into worker failures. 🟢 Model selection per stage is part of coordinator design, not an afterthought.


## Fork vs. Clean Agent

A fork is not a copy — it's a continuation from a shared prefix. When you fork a Claude agent, the child receives a byte-identical system prompt, tool list, and model assignment. Any specialization happens through the user-message directive only. Everything before the fork point is shared state.

A clean agent starts from scratch. No system prompt inheritance, no tool list, no prior turns. It costs more to orient (the scout stage exists to minimize this cost), but it cannot be contaminated.

The rule is simple:

| Task type | Agent type |
|---|---|
| Review parent's work | Fork |
| Extract memory from session | Fork |
| Independent worker in parallel batch | Clean |
| Exploration / open-ended research | Clean |
| Coordinator subworker | Clean |

Getting this wrong is expensive in both directions. A fork that was meant to be clean carries invisible priors. A clean agent doing continuation work has to re-discover everything the parent already knew.


## Fork Cache Mechanics

The economics of forking are driven by Anthropic's prompt caching model. When a forked child sends its first request, the prefix — system prompt plus all prior turns — is byte-identical to the parent's last request. Anthropic matches this prefix and returns a `cache_read` rather than processing it again.

On a 100K+ token context, the fork's first call costs roughly 10% of what a cold-start call would cost. In multi-agent runs with heavy forking, we observe ~90% `cache_read` ratios on the inherited prefix 🟡 — though we haven't formally benchmarked this; it's based on API response headers across several pipeline runs.

The practical implication: don't kill the orchestrator between tasks. A long-running parent agent accumulates cache. Every child forked from it inherits that cache and starts near-free. A child spawned clean pays full price for its initial orientation — reading files, loading context, processing the system prompt. The orchestrator's "idle" time between dispatches is not waste; it's keeping the cache warm.

The condition for cache hits is strict: system prompt, tool definitions, and model must be byte-identical. A single character change in the system prompt breaks the match. This is why tool and prompt standardization across pipeline stages is an economic decision, not just an aesthetic one. See [Principles](./principles.md) §5.


## Subagent Economics

The cheapest subagent is a skill. A skill (stored in `.claude/skills/` or `.pi/agent/skills/`) runs inside the parent's process, shares the parent's cache, and pays zero spawn overhead. An agent is a separate process: it starts cold, reloads tools, and rebuilds context from scratch.

Measured across repeated invocations, a skill runs approximately **28× cheaper** than an equivalent agent call 🟢. This is not a marginal difference — it changes whether a pattern is economically viable at scale.

The tradeoff is isolation. Skills share the parent's context and therefore its failure modes. Agents are isolated and therefore safe for tasks where contamination or runaway token consumption is a risk. The decision tree:

- Task needs isolation (long-running, risky, parallel) → Agent
- Task is lightweight and repeatable (search, format, summarize) → Skill
- Task is a one-off continuation → Fork

Industry data supports the agent-at-scale trajectory: Stripe processes 1,000+ merged PRs per week through a closed-loop agent system (git → lint → type-check → test → error-in-context → retry) backed by an MCP server with 400+ tools. Subagent support is now standard across Claude Code, Codex, Gemini CLI, and Cursor — the main value being root context preservation and isolating token-heavy operations.

The Azure multi-agent taxonomy identifies five patterns: Sequential, Concurrent, Group Chat, Handoff, and Magentic. The practical rule that holds across all of them: use the minimum complexity that solves the problem. Every coordination layer adds latency, cost, and failure surface. Augment Code's single-writer rule for hotspot files and sequential merge strategy is a concrete instance of this principle — verification becomes the bottleneck, not execution.


## Open Questions

- **Pre-flight as pipeline primitive.** Review agents failed on first attempt 100% of the time in [Experiment 3](./experiments.md#experiment-3-pipeline-end-to-end). Both failures were deterministically checkable — missing imports, type errors. We added a pre-flight step manually and it eliminated the problem. The question is whether pre-flight should be a formal pipeline stage (between Workers and Review) rather than an ad-hoc addition. We haven't tested whether pre-flight catches enough to justify always running it versus running it conditionally.
- **Human-in-loop placement.** In one debugging session, the critical breakthrough came from the user asking a direct question outside the pipeline — not from agent delegation. The agent had been looping on the wrong hypothesis for 3 iterations before the user intervened. 🟡 Where does human judgment fit inside a structured pipeline without becoming a bottleneck? We don't have a pattern for "agent requests human input at step N" that doesn't break the autonomous flow.
