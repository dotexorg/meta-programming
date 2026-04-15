# Context engineering: fitting the world into 200K tokens

The context window isn't a buffer you fill. It's an attention budget, and you spend it faster than you think. Every token competes for attention with every other token. You can have 200K tokens available and quietly lose the critical instruction at position 50K.

## The effective window collapses before you reach it

Context is not storage. The model computes attention relationships between every token pair (complexity scales as n² over sequence length), so adding tokens costs the model focus, not just capacity 🟡. Beginning and end of context get the most attention; the middle gets lost. This effect is consistent enough across models that it has a name: lost-in-the-middle 🟡.

The Chroma (2025) Maximum Effective Context Window (MECW) paper reportedly tested 18 frontier models and found universal degradation, with the effective performance window falling as low as 1-2% of the advertised maximum in some conditions 🔴. We haven't verified those specific numbers. What we have verified: Opus 4.6 showed clear instruction-following failures at roughly 40% context fill 🟢. A task that passed cleanly in a fresh session would silently fail mid-context. Not with an error, just with the model ignoring rules set an hour earlier. That silence is the dangerous part.

Four patterns drive degradation 🟡: poisoning, where early errors compound into later reasoning; distraction, where irrelevant content competes for attention with the actual task; confusion, where the model loses track of which sources relate to which question; and clash, where contradictory information from different context positions produces inconsistent outputs. Long-running sessions typically run into at least two. The practical implication: a 200K-token window doesn't give you 200K tokens of reliable context. Plan around a much smaller effective budget. Architect so hitting the limit fails loudly, not quietly.

## Progressive disclosure: load what's needed, when needed

The wrong instinct is to put everything in the system prompt and let the model sort it out. The right instinct is to deliver information as close as possible to when it's actually needed 🟡.

Our knowledge base has 429 bullets, roughly 80-100K tokens. Loading the full KB into every session would consume half the window before any work started, and most of it would land in the lost-in-middle zone anyway. Instead, we load a synthesis file (~8KB) as the always-on layer and expose the full bullet store as a searchable tool 🟢. The model requests what it needs. Irrelevant material never enters context.

The four-bucket strategy 🟡 formalizes this pattern across any system:

- **Write**: externalize persistent state to files, not conversation history
- **Select**: pull relevant content via search or grep rather than preloading everything
- **Compress**: summarize old context before it rots in the middle
- **Isolate**: split work across subagents, each with clean context

We use all four in production. The hardest discipline is Write. Letting the model hold state in conversation instead of writing checkpoint files feels faster, until sessions grow long and that state starts evaporating.

One counterintuitive finding: aggressive compression backfires 🟡. Compressing history eliminates the artifact trail, the chain of intermediate outputs the model uses to avoid re-discovering conclusions it already reached. Tokens-per-task (total across all requests in a workflow) is what matters, not tokens-per-request. Compress too eagerly and the agent redoes work it already completed. You spend fewer tokens per call and more tokens overall.

## Scout caching: read once, use many times

In a multi-agent pipeline, every subagent starts empty. Ten workers reading the codebase independently? You pay for the same files ten times. One of our pipelines burned 1.2M input tokens on scout calls alone. One feature. 🟢.

Scout caching kills this. One agent reads the codebase, writes a structured report to `.pi/scout-cache/`. Everyone else reads the report. Cache key is the git hash. Code changes, cache dies.

The token math is concrete. A worker that needs system context (5K) + relevant scout sections (8K) + task spec (2K) runs on 15K tokens. Passing the full scout report instead costs 67K. Across 14 workers, that's 210K vs 938K tokens of input, roughly $0.60 vs $2.65 at current Sonnet pricing, before output or retries 🟢.

One tradeoff: the cache is a static snapshot. Mid-run code changes mean stale data. Usually fine for feature work. For pipelines that modify shared files, coordinate explicitly.

## Prompt caching: how it works, and what breaks it

Anthropic's prompt cache operates on prefix matching: if the start of a request matches a cached prompt byte-for-byte, the cached tokens reuse stored activations instead of reprocessing them. Cache reads cost roughly 10× less than cache creation; writes cost slightly above standard token price 🟡.

The TTL is 5 minutes. For short workflows that's usually enough. For sequential pipelines where workers start 10+ minutes after the first cache write, it's a full miss, paid at full price.

The harder problem is multi-agent isolation. Each subagent runs as a separate API session. Even when every worker carries an identical system prompt, there's no shared cache across sessions. Each agent creates its own cache, reads zero from prior agents, and the TTL expires before the next worker starts 🟢. In one measured pipeline: 14 subagents, 700K tokens of cache creation, 0 tokens of cache reads. The cache existed individually per agent and never transferred. This is the multi-agent cache tax — a known tradeoff of isolation, not a bug, but it has real cost.

For concurrent agents, spawn them close together so the 5-minute window overlaps across the batch. For sequential pipelines, pass shared context as a file (which lives in the scout cache, persisting across session boundaries) rather than repeating it as a system prompt in each new session.

Model switches are full misses. The cache key includes the model identifier. Switch from Sonnet to Opus mid-pipeline and every cached prompt is invalid 🟡. Build pipelines around one model per tier, or accept that cross-tier handoffs reset the caching budget to zero.

## Token optimization: maps, ignores, compression

**RepoMap** (from Aider) builds a ranked AST map of the codebase using tree-sitter: compressed signatures, class hierarchies, and call relationships without full file content 🟡. It gives an agent fast orientation in large repos. The tradeoff: the map highlights files that appear central by graph structure. When the right file for a specific task isn't the most connected one in the call graph, the agent may follow the map instead of the codebase and miss the actual location. Use maps for orientation; don't let them drive implementation decisions.

**.claudeignore** works like `.gitignore`: patterns listed there are excluded from context loading entirely. Build artifacts, test fixtures, generated files, `node_modules`, `/dist` — a well-maintained ignore file can halve the tokens an agent spends just finding relevant code. No exotic tooling required 🟡.

Community compression tools have promising reported numbers but we haven't benchmarked them ourselves: Tilth CLI claims 40% input reduction, RTK (12K GitHub stars) claims 60-90% output compression, Cozempic claims 23% history reduction 🟠. They're reportedly stackable. One caution: output compression that drops intermediate reasoning can force the agent to regenerate work it already did — compounding the artifact trail problem rather than solving it.

A complementary observation from Willison 🟡: agents are remarkably consistent at following existing code patterns. A clean, idiomatic codebase is itself compressed context. One good example does the work of a paragraph of style instructions. If the first file in a pattern is wrong, expect the pattern to propagate. Fix the example, not just the rule.

## How Claude Code assembles context

Claude Code loads context in a defined order that determines what's always in working memory and what arrives on-demand 🟡. The order matters for placement decisions. Putting the wrong content in the wrong tier wastes budget or loses attention.

**Tier 1, Always loaded**: root `CLAUDE.md` + `@import`s + path-less rules. Every request. Sizing: root under 100 lines, rules 20-50 each, total under 300 lines. Past 200 lines per file, the middle starts degrading. Lost-in-middle at document scale. 🟡

**Tier 2, On-demand**: path-scoped rules. Tagged to `/frontend/**`? Only loads when the agent works there. Framework guidance that would distract the API layer stays out.

**Tier 3, Lazy**: nested `CLAUDE.md` files in subdirectories. Loaded on first access to that subtree, then stays warm for the session.

**Tier 4, Auto Memory**: persistent state, managed automatically. Less control.

**Tier 5, Skills**: zero cost until invoked. Free until needed.

For per-agent roles, we scope context deliberately ⚪: scouts get architecture context plus root rules. Enough to map the codebase, not enough to bias implementation decisions. Workers get root rules plus path-scoped rules for the directories they're modifying. Reviewers get the review checklist and security rules, explicitly not the implementation context — an agent that knows how something was built tends to rationalize rather than evaluate. More context isn't always better; sometimes it's precisely what makes the reviewer useless.

This tier structure, and the pattern of isolated agents with scoped context, is covered in more depth in [Pipeline](./pipeline.md).

## Open questions

**When should compaction trigger?** Claude Code's default compaction fires at around 70-80% fill. We've observed instruction failures starting at 40%. The gap represents 30-40% of capacity where rules set early in the session are already unreliable — but the session continues, and nothing signals the degradation. We haven't tested manually triggering compaction at 50% to see whether it recovers compliance, or whether the compaction overhead makes it net negative for short tasks. The experiment is straightforward; we haven't run it.

**Can sequential pipelines warm the cache across session boundaries?** We've measured the 0-cache-read outcome (14 subagents, 700K cache_create, 0 cache_read). There may be a construction — passing exactly the same system prompt bytes plus shared context in a consistent format — that achieves cache hits across back-to-back sessions started within the 5-minute TTL. We've observed the problem systematically; we haven't tested the workaround systematically.
