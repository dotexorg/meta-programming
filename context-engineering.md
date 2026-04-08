# Context Engineering

The context window is the most constrained resource in agentic AI. Understanding how it degrades, how to fill it strategically, and how to keep it warm across sessions is what separates reliable pipelines from ones that quietly collapse under load.

Examples throughout this documentation use [Pi](https://github.com/mariozechner/pi), an open-source coding agent framework. The patterns apply to any agent setup — Claude Code, Cursor, Copilot, or custom pipelines. Where a technique is tool-specific, we note it.

## The Window Is Smaller Than You Think

The effective context window for a working agent is a fraction of the advertised maximum — and the fraction depends on task complexity. The MECW study (Chroma, 2025) tested 18 frontier models and found that every one degrades with context growth. 🟡 Complex reasoning tasks — code generation, multi-step planning — collapse earliest. Simple retrieval survives longer. Practitioners report the threshold independently: "dumb zone begins at ~6%" of context (mitsuhiko, Sentry); at 40% utilization on a 1M-token model, initial analysis and instructions are effectively forgotten (observed in production sessions). 🟡

This isn't a bug. It's the nature of attention-based models — n² relationships mean more tokens = thinner attention on each one. The practical implication: treat your context budget as a scarce resource from the first token, not a fallback concern when things break. The failure mode is **silent degradation**, not visible errors — the model keeps generating plausible-looking output while operating on wrong assumptions baked in from dozens of turns ago.

The answer isn't a bigger window. It's context discipline.

## Progressive Disclosure

Loading everything upfront is a tax you pay on every request. Progressive disclosure is the practice of loading context at the moment it becomes relevant — not before.

Meta-Programming structures knowledge in three tiers: 🟡

- **Tier 1 (~8K tokens, always loaded):** Persona, core directives, project conventions. This is the invariant prefix — it never changes and is always present. Keeping it stable is a prerequisite for effective prompt caching.
- **Tier 2 (contextually loaded):** Domain patterns, architecture guides, relevant specs for the current task. Loaded when a task's classification matches the domain.
- **Tier 3 (on-demand retrieval):** Historical decisions, deep theory, reference implementations. Fetched only when an agent explicitly needs them.

The goal is a context that's *minimal but sufficient* — not a dump of everything that might be useful. Auto-generated context actually **hurts agent performance by 3%**, while human-written boundaries improve it by 4% (ETH Zurich, Feb 2026, 138 real-world tasks). 🟡 The insight is counterintuitive but consistent: more context isn't free. Every token you add is a token the model has to process and attend to. The principle is minimal effective context — only include information the agent cannot discover on its own.

[See Principles](./principles.md) — Principle 1 (Process > Context) covers why structured process beats raw context volume.

## Scout Caching

Each subagent in a pipeline reads the codebase from scratch. A pipeline with 7 workers pays for the same files 7 times — in one real run, 4+ scout calls on a single feature consumed 1.2M tokens on file reads alone. 🟢 Scout caching eliminates this: one agent reads the full repository once, writes a structured report, and all subsequent workers consume the report instead of the raw codebase.

Invalidation is by **git commit hash**. If the hash hasn't changed, the cached report is valid. When the codebase changes, the scout re-runs and rewrites the cache. This gives you:

- One cold read per commit, not per agent
- Consistent codebase understanding across all workers in a pipeline
- Orchestrators that stay lightweight and focused on coordination

The tradeoff: scout reports are summaries. An agent that gets a scout report instead of raw source is working from an abstraction. For most tasks this is fine — and faster. For tasks that need exact line-level access, workers can request specific files on demand from the scout report's index.

The optimal approach is slicing: don't give every worker the full scout report. Pass only the sections relevant to their task. A full 60K-token report compresses to ~8K relevant tokens — a 4.5× saving without quality loss. 🟢

[See Pipeline](./pipeline.md) for how scout output flows through the full orchestration model.

## Prompt Caching

Prompt caching works on **prefix matching**: the system prompt, tools, and initial messages must be byte-identical to get a cache hit. A cache hit cuts time to first token by 13–31% and cost by 45–80% (PwC, *Don't Break the Cache*, 500+ agent sessions). 🟡

The practical rules:

**Put dynamic content at the end.** The cache prefix must be stable. Any dynamic content — timestamps, session IDs, per-request state — breaks the prefix match if it appears early. Keep it at the tail of the messages array.

**Avoid dynamic function calling in the prefix.** Tool results that vary per-call should not appear in the stable prefix. Claude Code handles this with `defer_loading` — deferred tools are excluded from the system-prompt prefix entirely. Only always-needed tools appear in the prefix, preserving cacheability across sessions. 🟡

**Model switches are full cache misses.** The cache doesn't survive switching between models mid-session. If your pipeline swaps models based on task complexity — say, routing simple tasks to a smaller model — every switch starts cold. Design your routing logic around this: cluster similar model calls together rather than interleaving them.

**Extend the cache TTL.** The default 5-minute TTL is too short for iterative development workflows. Extending it to 1 hour produces a noticeable improvement in cache hit rates for multi-turn sessions. 🟢 In Pi, this is `PI_CACHE_RETENTION=long`; Claude Code achieves similar persistence through its sticky latch mechanism. Check your provider's TTL options — this is free performance.

## Fork vs Clean Agent

Multi-agent pipelines have a structural cache problem: each subagent is a separate API session with a cold cache. A pipeline running 14 subagents with 50K tokens each generates 700K tokens of `cache_create` on every run. The fork pattern is the primary mitigation.

**Fork** means the child agent inherits the parent's full conversation prefix byte-for-byte — same system prompt, same tools, same message history. Only the final directive differs. Because the API cache key is the entire prefix, a forked child gets ~90% `cache_read` instead of 100% `cache_create`. 🟡 The implementation trick: all fork children receive identical placeholder tool results, with only the last text block containing the per-child directive. This maximizes prefix overlap across siblings.

**Clean agent** means a fresh process with no shared context. Zero cache sharing, but complete isolation — no risk of context contamination from the parent's session.

When to use which:

- **Fork** for tasks that need parent context: code review (reviewer needs to see what was built), memory extraction (needs full conversation), side questions, post-turn analysis
- **Clean** for independent work: implementation workers (need spec, not history), exploration (focused search without parent noise), tasks where a fresh perspective matters

The tradeoff is real: forking is cheaper but leaks context. A reviewer that forks from the builder session inherits the builder's assumptions — potentially undermining the independence that makes separate review valuable. Design your pipeline with both patterns, not one.

[See Pipeline](./pipeline.md) for the full orchestration model.

## Token Optimization

Beyond caching, there are three levers for reducing token spend:

**Code maps: useful but dangerous.** Tools like Aider's RepoMap use tree-sitter to build ranked AST maps — files, symbols, call relationships — in a fraction of the tokens a full read costs. But code maps can actively hurt: in our A/B test, the code-map run cost 51% more ($9.99 vs $6.63) and failed, while the pipeline without a map passed on the first attempt. 🟢 The mechanism: a map gives the agent a fast path that discourages exploration, leading it to miss scope it didn't know it needed. Use maps for navigation-heavy tasks where the target is known; avoid them for open-ended implementation.

**Exclusion patterns.** Analogous to `.gitignore`, tools like `.claudeignore` exclude paths from context assembly. Build artifacts, `node_modules`, generated files, test fixtures with large datasets — none of these belong in the context window. A well-maintained exclusion list cuts effective token cost per task substantially. 🟡

**Finite Brain Model.** Every knowledge resource declares its token cost upfront. Before a context rebuild, a budget dashboard shows what's being loaded and what it costs. Graceful degradation: if a resource would push the budget over threshold, it loads at reduced fidelity — full body → outline → catalog listing. 🟡 This keeps context assembly predictable rather than letting it balloon silently.

[See Specification](./specification.md) — specs themselves are a form of context compression. A good spec loads a task's full intent in ~2K tokens rather than spreading it across a conversation.

## How Claude Code Assembles Context

Claude Code's context assembly is structured in two dimensions: **5 tiers of priority** and **4 layers of compression**.

The 5-tier system controls what enters the context at all — project memory, session memory, inline instructions, tool results, and inferred context each fill different roles. Higher tiers shadow lower ones when content conflicts.

When the session runs long, Claude Code applies **4-layer compression** in sequence: 🟡

1. **Snip** — drops low-priority tool outputs and intermediate scratchpad
2. **Microcompact** — condenses repetitive message patterns
3. **Collapse** — merges consecutive messages of the same role
4. **Auto-compact** — full context rebuild from a structured compact prompt

The compact prompt that emerges from auto-compact has 9 defined sections, and critically: **all user messages are preserved verbatim**. The model's responses get compressed; the user's instructions don't. This asymmetry is intentional — user intent is the signal, model output is the work product.

Understanding this pipeline matters for prompt design. Instructions that appear only in assistant turns risk being compressed away. Instructions that need to survive long sessions should appear in user turns or in the stable system prefix — not buried in the middle of a long assistant response.

## Open Questions

- **Optimal Tier 2 classification latency.** Contextual loading requires classifying the incoming task before assembling context. How fast can this classification be without sacrificing accuracy? Is a dedicated lightweight classifier worth the added complexity?
- **Code map staleness.** A cached code map is cheap but diverges from the actual codebase after commits. What's the right invalidation strategy — same git-hash approach as scout, or incremental diff?
- **Compression fidelity.** Claude Code's auto-compact preserves user messages verbatim. For pipelines where agent-to-agent messages are the primary signal (not user input), does this asymmetry work against us?
