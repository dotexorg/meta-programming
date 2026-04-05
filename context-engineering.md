# Context Engineering

The context window is the most constrained resource in agentic AI. Understanding how it degrades, how to fill it strategically, and how to keep it warm across sessions is what separates reliable pipelines from ones that quietly collapse under load.

**On This Page**
- [The Window Is Smaller Than You Think](#the-window-is-smaller-than-you-think)
- [Progressive Disclosure](#progressive-disclosure)
- [Scout Caching](#scout-caching)
- [Prompt Caching](#prompt-caching)
- [Token Optimization](#token-optimization)
- [How Claude Code Assembles Context](#how-claude-code-assembles-context)

---

## The Window Is Smaller Than You Think

The MECW (Minimum Effective Context Window) paper makes a blunt claim: the effective context window for a working agent is **1–2% of the advertised maximum**. 🟠 **External** A 200K-token model doesn't give you 200K tokens of useful reasoning. Agent quality degrades well before you hit the hard limit — attention diffuses, earlier instructions get shadowed by later content, and the model begins making assumptions from stale context rather than fresh instructions.

This isn't a bug. It's the nature of attention-based models. The practical implication: treat your context budget as a scarce resource from the first token, not a fallback concern when things break. David Reis' piece ["Why AI Agents Get Worse Over Time"](https://davidreis.dev) identifies the failure mode clearly — context overload produces **silent degradation**, not visible errors. 🟠 **External** The model keeps generating plausible-looking output while operating on wrong assumptions baked in from messages 50 turns ago.

The answer isn't a bigger window. It's context discipline.

---

## Progressive Disclosure

Loading everything upfront is a tax you pay on every request. Progressive disclosure is the practice of loading context at the moment it becomes relevant — not before.

Meta-Programming structures knowledge in three tiers: 🟡 **Observed**

- **Tier 1 (~8K tokens, always loaded):** Persona, core directives, project conventions. This is the invariant prefix — it never changes and is always present. Keeping it stable is a prerequisite for effective prompt caching.
- **Tier 2 (contextually loaded):** Domain patterns, architecture guides, relevant specs for the current task. Loaded when a task's classification matches the domain.
- **Tier 3 (on-demand retrieval):** Historical decisions, deep theory, reference implementations. Fetched only when an agent explicitly needs them.

The goal is a context that's *minimal but sufficient* — not a dump of everything that might be useful. ETH Zurich found that auto-generated context actually **hurts agent performance by 3%**. 🟠 **External** The insight is counterintuitive but consistent: more context isn't free. Every token you add is a token the model has to process and attend to. The principle is minimal effective context — only include information the agent cannot discover on its own.

[See Principles](./principles.md) — Principle 1 (Process > Context) covers why structured process beats raw context volume.

---

## Scout Caching

The most expensive context operation is reading a codebase. Scout caching eliminates repeat reads by making it a one-time operation.

The pattern: a scout agent reads the full repository and writes a structured report. All subsequent worker agents receive the scout report, not the raw codebase. Orchestrators never read files directly. 🟡 **Observed**

Invalidation is by **git commit hash**. If the hash hasn't changed, the cached report is valid. When the codebase changes, the scout re-runs and rewrites the cache. This gives you:

- One cold read per commit, not per agent
- Consistent codebase understanding across all workers in a pipeline
- Orchestrators that stay lightweight and focused on coordination

The tradeoff: scout reports are summaries. An agent that gets a scout report instead of raw source is working from an abstraction. For most tasks this is fine — and faster. For tasks that need exact line-level access, workers can request specific files on demand from the scout report's index.

[See Pipeline](./pipeline.md) for how scout output flows through the full orchestration model.

---

## Prompt Caching

Anthropic's prompt cache works on **prefix matching**: the system prompt, tools, and initial messages must be identical to get a cache hit. A cache hit cuts TTFT (time to first token) by 13–31% and cost by 45–80%. 🟠 **External** (PwC, *Don't Break the Cache*)

The practical rules:

**Put dynamic content at the end.** The cache prefix must be stable. Any dynamic content — timestamps, session IDs, per-request state — breaks the prefix match if it appears early. Keep it at the tail of the messages array.

**Avoid dynamic function calling in the prefix.** Tool results that vary per-call should not appear in the stable prefix. Claude Code handles this with `defer_loading` — deferred tools are excluded from the system-prompt prefix entirely. Only always-needed tools appear in the prefix, preserving cacheability across sessions. 🟡 **Observed**

**Model switches are full cache misses.** The cache doesn't survive switching between models mid-session. If your pipeline swaps models based on task complexity — say, routing simple tasks to a smaller model — every switch starts cold. Design your routing logic around this: cluster similar model calls together rather than interleaving them.

**Extended TTL:** Setting `PI_CACHE_RETENTION=long` extends the default 5-minute TTL to 1 hour. This only applies to `api.anthropic.com` — not third-party proxies or wrappers. 🟡 **Observed**

**Multi-agent cache tax is structural.** Each subagent is a separate API session with a cold cache. A pipeline running 14 subagents with 50K tokens each generates 700K tokens of `cache_create` on every run. There's no clean solution — only mitigation: minimize subagent count, share context through scout reports rather than re-reading, and use Claude Code's fork pattern when possible (forked sessions share identical parameters and can hit the same cache). 🟡 **Observed**

---

## Token Optimization

Beyond caching, there are three levers for reducing token spend:

**RepoMap.** Aider's RepoMap uses tree-sitter to build a ranked AST map of the repository — files, symbols, call relationships — in a fraction of the tokens a full read would cost. 🟠 **External** This is now spreading across tools via `repomap-mcp`. The tradeoff: a RepoMap gives the agent a fast path to navigation but can hurt deep exploration. The agent follows the map instead of reading code it doesn't know it needs.

**.claudeignore.** Analogous to `.gitignore`, it excludes paths from context assembly. Build artifacts, `node_modules`, generated files, test fixtures with large datasets — none of these belong in the context window. A well-maintained `.claudeignore` cuts your effective token cost per task substantially. 🟡 **Observed**

**Finite Brain Model.** Every knowledge resource declares its token cost upfront. Before a context rebuild, a budget dashboard shows what's being loaded and what it costs. Graceful degradation: if a resource would push the budget over threshold, it loads at reduced fidelity — full body → outline → catalog listing. 🟡 **Observed** This keeps context assembly predictable rather than letting it balloon silently.

[See Specification](./specification.md) — specs themselves are a form of context compression. A good spec loads a task's full intent in ~2K tokens rather than spreading it across a conversation.

---

## How Claude Code Assembles Context

Claude Code's context assembly is structured in two dimensions: **5 tiers of priority** and **4 layers of compression**.

The 5-tier system controls what enters the context at all — project memory, session memory, inline instructions, tool results, and inferred context each fill different roles. Higher tiers shadow lower ones when content conflicts.

When the session runs long, Claude Code applies **4-layer compression** in sequence: 🟡 **Observed**

1. **Snip** — drops low-priority tool outputs and intermediate scratchpad
2. **Microcompact** — condenses repetitive message patterns
3. **Collapse** — merges consecutive messages of the same role
4. **Auto-compact** — full context rebuild from a structured compact prompt

The compact prompt that emerges from auto-compact has 9 defined sections, and critically: **all user messages are preserved verbatim**. The model's responses get compressed; the user's instructions don't. This asymmetry is intentional — user intent is the signal, model output is the work product.

Understanding this pipeline matters for prompt design. Instructions that appear only in assistant turns risk being compressed away. Instructions that need to survive long sessions should appear in user turns or in the stable system prefix — not buried in the middle of a long assistant response.

---

## Open Questions

- **Optimal Tier 2 classification latency.** Contextual loading requires classifying the incoming task before assembling context. How fast can this classification be without sacrificing accuracy? Is a dedicated lightweight classifier worth the added complexity?
- **RepoMap staleness.** A cached RepoMap is cheap but diverges from the actual codebase after commits. What's the right invalidation strategy — same git-hash approach as scout, or incremental diff?
- **Compression fidelity.** Claude Code's auto-compact preserves user messages verbatim. For pipelines where agent-to-agent messages are the primary signal (not user input), does this asymmetry work against us?
