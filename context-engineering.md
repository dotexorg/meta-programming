# Context engineering: fitting the world into 200K tokens

The context window isn't a buffer you fill. It's an attention budget, and you spend it faster than you think. Every token competes for attention with every other token. You can have 200K tokens available and quietly lose the critical instruction at position 50K.

## The effective window collapses before you reach it

Context is not storage. The model computes attention relationships between every token pair (complexity scales as n² over sequence length), so adding tokens costs the model focus, not just capacity. Beginning and end of context get the most attention; the middle gets lost. This effect is consistent enough across models that it has a name: lost-in-the-middle.

The Chroma MECW paper tested 18 frontier models. Every one degraded. In some conditions, the effective window fell to 1-2% of advertised maximum. We haven't verified their specific numbers, but we've seen the pattern ourselves: Opus 4.6 started ignoring instructions at roughly 40% context fill. Not with an error. Just silently skipping rules set an hour earlier. A task that passed in a fresh session would fail mid-context with no signal. That silence is the dangerous part.

Four patterns drive degradation. **Poisoning**: early errors compound into later reasoning. **Distraction**: irrelevant content competes for attention with the actual task. **Confusion**: the model loses track of which sources relate to which question. **Clash**: contradictory information from different positions produces inconsistent output. Long sessions typically hit at least two.

The practical implication: a 200K-token window doesn't give you 200K tokens of reliable context. Plan around a much smaller effective budget. Architect so hitting the limit fails loudly, not quietly.

## Progressive disclosure: load what's needed, when needed

The wrong instinct is to put everything in the system prompt and let the model sort it out. Information lands better when it arrives close to the moment the agent actually needs it.

Our knowledge base holds ~1,030 bullets, roughly 180K tokens of raw archive. Loading it all into every session would consume the entire window before any work started, and most would land in the lost-in-middle zone anyway. The architecture splits the material across four roles. The **raw archive** (`meta.json`) is searchable via a tool call, never loaded whole. A **curated core** (`tier1-core.md`, ~12KB) — thesis, principles, our own experiments — is the actual always-load layer, deliberately kept under the 200–300-line AGENTS.md sweet spot. **Topic tiers** (`tier2-*.md`, 4–18KB each) load on-demand when the task touches that area. A **synthesis doc** sits between the raw KB and the tiers as a staging artifact — it collects narrative during research sessions, gets drained into the tier docs during sync, and is not part of the agent's runtime context.

The model requests what it needs from the tiers or searches the raw store directly. Irrelevant material never enters context. The same pattern generalises: one small always-on layer with the core thesis, domain tiers by topic, everything else searchable.

The four-bucket strategy formalizes this pattern across any system:

- **Write**: save persistent state to files, not conversation history
- **Select**: pull relevant content via search or grep rather than preloading everything
- **Compress**: summarize old context before it rots in the middle
- **Isolate**: split work across subagents, each with clean context

We use all four in production. The hardest discipline is Write. Letting the model hold state in conversation instead of writing checkpoint files feels faster, until sessions grow long and that state starts evaporating.

One counterintuitive finding: aggressive compression backfires. Compressing history eliminates the artifact trail, the chain of intermediate outputs the model uses to avoid re-discovering conclusions it already reached. Tokens-per-task (total across all requests in a workflow) is what matters, not tokens-per-request. Compress too eagerly and the agent redoes work it already completed. You spend fewer tokens per call and more tokens overall.

## Scout caching: read once, use many times

In a multi-agent pipeline, every subagent starts empty. Ten workers reading the codebase independently? You pay for the same files ten times. One of our pipelines burned 1.2M input tokens on scout calls alone. One feature..

Scout caching solves this by reading the codebase once. A single agent explores, writes a structured report to `.pi/scout-cache/`, and every subsequent worker reads that report instead of the source. The cache key is the git hash, so it invalidates automatically when code changes.

The token math is concrete. A worker that needs system context, relevant scout sections, and a task spec runs on about 15K tokens. Passing the full scout report instead costs 67K. Across 14 workers, that's a 4.5× difference in input tokens. The gap compounds with retries.

The tradeoff is staleness. The cache is a static snapshot, so mid-run code changes won't be reflected. That's usually fine for feature work, but pipelines that modify shared files need explicit coordination.

## Prompt caching: how it works, and what breaks it

Anthropic's prompt cache operates on prefix matching: if the start of a request matches a cached prompt byte-for-byte, the cached tokens reuse stored activations instead of reprocessing them. Cache reads cost roughly 10× less than cache creation; writes cost slightly above standard token price.

The TTL is 5 minutes. For short workflows that's usually enough. For sequential pipelines where workers start 10+ minutes after the first cache write, it's a full miss, paid at full price.

The harder problem is multi-agent isolation. Each subagent runs as a separate API session. Even when every worker carries an identical system prompt, there's no shared cache across sessions. Each agent creates its own cache, reads zero from prior agents, and the TTL expires before the next worker starts. In one measured pipeline: 14 subagents, 700K tokens of cache creation, 0 tokens of cache reads. The cache existed individually per agent and never transferred. This is the multi-agent cache tax — a known tradeoff of isolation, not a bug, but it has real cost.

For concurrent agents, spawn them close together so the 5-minute window overlaps across the batch. For sequential pipelines, pass shared context as a file (which lives in the scout cache, persisting across session boundaries) rather than repeating it as a system prompt in each new session.

Model switches are full misses because the cache key includes the model identifier. Switch from Sonnet to Opus mid-pipeline and every cached prompt becomes invalid. Build pipelines around one model per tier, or accept that cross-tier handoffs reset the caching budget to zero.

## Token optimization: maps, ignores, compression

**RepoMap** (from Aider) builds a ranked AST map of the codebase using tree-sitter: compressed signatures, class hierarchies, and call relationships without full file content. It gives an agent fast orientation in large repos. The tradeoff: the map highlights files that appear central by graph structure. When the right file for a specific task isn't the most connected one in the call graph, the agent may follow the map instead of the codebase and miss the actual location. Use maps for orientation; don't let them drive implementation decisions.

**.claudeignore** works like `.gitignore`: patterns listed there are excluded from context loading entirely. Build artifacts, test fixtures, generated files, `node_modules`, `/dist` — a well-maintained ignore file can halve the tokens an agent spends just finding relevant code. No exotic tooling required.

Community compression tools have promising reported numbers but we haven't benchmarked them ourselves: Tilth CLI claims 40% input reduction, RTK (12K GitHub stars) claims 60-90% output compression, Cozempic claims 23% history reduction. They're reportedly stackable. One caution: output compression that drops intermediate reasoning can force the agent to regenerate work it already did — compounding the artifact trail problem rather than solving it.

A complementary observation from Willison: agents are remarkably consistent at following existing code patterns. A clean, idiomatic codebase is itself compressed context. One good example does the work of a paragraph of style instructions. If the first file in a pattern is wrong, expect the pattern to propagate. Fix the example, not just the rule.

## The AGENTS.md cliff

The intuition that more rules produce better behavior is wrong past a specific threshold, and the threshold is documented. A 2026 paper on AGENTS.md (arxiv 2602.11988) measured SWE-bench success rates against context file size: success declines past **500 lines**, and the drop is a cliff rather than a gradual slope. The sweet spot the authors identify sits at 200–300 lines of actionable rules. The failure mode they name: "not giving the model a mental model, giving it a compliance checklist." At checklist length the model stops reading and starts skipping, visibly enough to move benchmark scores by measurable amounts.

Adherence to context file rules sits around **70% even under the budget**. Practitioners converged on this number independently through session analysis; the paper's result lines up. The implication for anything safety-critical: if a rule must hold every time ("never push to main," "always run tests before commit"), a context file is the wrong place for it. That's a hook. The context file is advisory; hooks are deterministic.

What this means for our own structure: the 200-line target in CLAUDE.md isn't an aesthetic preference but a measured ceiling. Past it, every added rule increases the risk that existing rules stop firing. Rule retirement matters as much as rule addition — the `adelaidasofia/claude-performance` loop (measure, add rule when metric drops, retire rule when metric stabilises without it) is the pattern that keeps the file under the cliff.

## How Claude Code assembles context

Claude Code loads context in a defined order that determines what's always in working memory and what arrives on-demand. The order matters for placement decisions. Putting the wrong content in the wrong tier wastes budget or loses attention.

**Tier 1, Always loaded**: root `CLAUDE.md` + `@import`s + path-less rules. Every request. Keep the root under 100 lines, individual rules short, total under 300 lines. Go past that and the middle starts degrading — lost-in-middle at document scale.

**Tier 2, On-demand**: path-scoped rules. Tagged to `/frontend/**`? Only loads when the agent works there. Framework guidance that would distract the API layer stays out.

**Tier 3, Lazy**: nested `CLAUDE.md` files in subdirectories. Loaded on first access to that subtree, then stays warm for the session.

**Tier 4, Auto Memory**: persistent state, managed automatically. Less control.

**Tier 5, Skills**: zero cost until invoked. Free until needed.

For per-agent roles, we scope context deliberately: scouts get architecture context plus root rules. Enough to map the codebase, not enough to bias implementation decisions. Workers get root rules plus path-scoped rules for the directories they're modifying. Reviewers get the review checklist and security rules, explicitly not the implementation context — an agent that knows how something was built tends to rationalize rather than evaluate. More context isn't always better; sometimes it's precisely what makes the reviewer useless.

This tier structure, and the pattern of isolated agents with scoped context, is covered in more depth in [Pipeline](./pipeline.md).

## Open questions

**When should compaction trigger?** Claude Code's default compaction fires at 70-80% fill. We've seen instruction failures starting at 40%. That's a wide band where the session continues, rules are already unreliable, and nothing signals the problem. Triggering compaction earlier (say 50%) might recover compliance — or the overhead might make it net negative for short tasks. The experiment is straightforward; we haven't run it.

**Can sequential pipelines warm the cache across session boundaries?** We've measured the 0-cache-read outcome (14 subagents, 700K cache_create, 0 cache_read). There may be a construction — passing exactly the same system prompt bytes plus shared context in a consistent format — that achieves cache hits across back-to-back sessions started within the 5-minute TTL. We've observed the problem systematically; we haven't tested the workaround systematically.
