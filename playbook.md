# 15 Rules That Actually Work

You don't need to read the full site to get results. These 15 rules cover 80% of what makes agent-assisted development succeed or fail. Each one is self-contained. Pick any, apply today, skip the rest.

The rules are backed by A/B tests, production incidents, and a year of daily agent use. The theory lives in the other pages. This page is the cheat sheet.

Examples use Claude Code, but most rules apply to any coding agent — Cursor, Codex, Windsurf, Copilot. The underlying constraint is the same: an LLM with a context window, a set of tools, and your instructions. Where a rule is Claude Code-specific (like `/clear`), the equivalent in your tool does the same thing.

## Setup

### 1. Keep CLAUDE.md under 200 lines

The model can follow roughly 200 rules at a time. The system prompt already consumes about 50, which leaves you around 150. A 500-line CLAUDE.md doesn't make the agent smarter — it makes it selectively deaf. Research across multiple agents and LLMs confirms that context files over 500 lines actively reduce success rates.

What belongs in there: build commands, test commands, naming conventions, architectural constraints, forbidden patterns. What doesn't: aspirational guidelines, style philosophy, anything you'd put in a README for humans. `use typescript strict mode` is a rule. `write clean, maintainable code` is noise.

**Template:**

```markdown
## Build & Run
npm run dev | npm test | npm run lint

## Stack
TypeScript 5, React 19, Postgres, Prisma

## Rules
- Functional style, no classes
- All new code gets tests in __tests__/ next to source
- Never use console.log — use the logger utility
- Never modify src/auth/ without explicit approval

## Architecture
src/modules/{domain}/  — one folder per domain
src/shared/            — cross-cutting utilities only
```

### 2. Cheap model by default, expensive model for judgment

Most tasks don't need the strongest model. Bug fixes, feature implementation, tests, refactors, code review — a mid-tier model handles all of these. As of mid-2026, that's Sonnet for Claude users or GPT-4.1 for OpenAI users. Switch to the top-tier model (Opus, o3) for complex multi-file refactors, architecture decisions, or subtle logic debugging where the agent needs to hold competing constraints in mind.

The thinking level matters more than the model itself. In controlled testing, Sonnet with high thinking outperformed Opus with medium thinking on rule-following tasks — at a fraction of the cost. If you're going to spend tokens, spend them on thinking depth, not model size.

### 3. Manage your context like a budget

The context window is an attention budget, not storage. Effective reasoning degrades well before the nominal limit — complex tasks can start failing at 30–40% utilization, not 90%.

**The discipline:** one task → verify → commit → `/clear` or new session → next task. Errors that slip through on task 3 don't poison tasks 4 through 7, because that context no longer exists. A 106-turn task in our pipeline testing didn't contaminate subsequent tasks — because we reset between them. If a task requires touching 30+ files, it's not one task. Decompose before starting.

**The cache:** don't throw away cached context for no reason. Most providers have cache tiers — Claude has a 5-minute and a 1-hour window, OpenAI has similar mechanics. The longer cache means you can work comfortably in one session without losing the cost benefit of cached prompts. The short cache is ideal for fast tasks and pipeline workflows where agents spawn in quick succession.

**When context gets heavy:** at 50–60% fill, tell the agent to summarize current state, decisions made, and remaining TODOs to a HANDOFF.md file. Then `/clear` and start fresh with "Read HANDOFF.md and continue." You control what survives instead of letting auto-compact decide — and auto-compact has primacy and recency bias, keeping the beginning and end while dropping the middle.

If you're doing a feature with multiple parts, write a PLAN.md first, then execute one phase per session.

## Before you code

### 4. Three lines before every task: DO / DO NOT / GLOSSARY

The single highest-leverage habit. Three lines that eliminate most failures before they happen.

- **DO**: What to build. One feature, scoped tight. Include acceptance criteria.
- **DO NOT**: Which files, APIs, and patterns to leave alone.
- **GLOSSARY**: Any term that could mean two things. "Fast" = under 200ms p95. "Reply" = the Telegram reply-to-message API, not a text response.

In a direct test, two consecutive failures without a spec — both deterministic, meaning they'd have happened every time. Adding `DO NOT modify classifyTicket` prevented both. Three words.

See [Specification](./specification.md) for the full framework.

### 5. Plan first, restate, then code

"Obvious on big features, overkill on small ones" — except skipping the plan on small features is exactly where bugs hide, because nobody reviews what seemed trivial.

Write the plan. Pressure-test it: "what breaks if we do this? what are we assuming?" Then before any implementation, ask the agent to restate what it understood the task to be. Often the model misinterprets something, or you left out a detail that forced it to assume. This one step exposes blindspots before a single line of code is written.

If the agent proposes something you don't understand during planning (barrel re-exports, unnecessary abstractions), question it. In our testing, an anti-pattern was caught and removed at spec review that would have shipped to production without it.

### 6. Commands, not prose

`run: npm test -- --coverage && assert coverage > 80%` changes behavior. `We value thorough testing and aim for high coverage` changes nothing. Across controlled runs, prose instructions produced zero measurable effect on agent behavior. Executable commands did.

Every behavioral requirement should be a verifiable command or binary condition. If you can't check whether the agent followed it, it's not a rule — it's a wish.

## During implementation

### 7. Agent loops? Stop, clear, restart

The fix→break→fix→revert cycle happens when contradictory constraints pile up in context. The model tries to satisfy your current error AND every failed fix attempt AND the original bug simultaneously. It can't, so it alternates between partial fixes that each break the other.

Don't add more context hoping it helps — more context means more room to spiral.

**Do this:**
1. Stop. Don't send another message.
2. `/clear` or start a new session.
3. Write one focused prompt: current state of the code + what you want + DO NOT constraints.

If the same loop happens twice on a clean start, the problem is your spec, not the model.

### 8. Use extended thinking when quality drops

If the agent's output suddenly feels shallow — missing edge cases, ignoring rules, producing generic code — the first thing to try is raising the thinking level. Adding `THINK` or `ULTRATHINK` to your prompt makes a measurable difference, often described by practitioners as "night and day."

The thinking budget is what the model spends on reasoning before it starts writing. Low thinking = the agent jumps to the most likely answer. High thinking = it actually works through the problem. In controlled testing, thinking level was a stronger predictor of rule-following than model choice. A cheaper model that thinks deeply outperformed an expensive model that thinks briefly.

When you notice quality fluctuating across sessions (same prompt, noticeably worse output), it may be provider-side: GPU load, checkpoint rotation, or A/B testing can all reduce the thinking budget silently. Extended thinking keywords override that floor and restore the baseline.

### 9. Verify before you move on

Don't treat "it looks done" as done. After the agent finishes a task, run your deterministic checks (`tsc`, `npm test`, linter) before anything else. These catch free errors in milliseconds with no hallucination risk. Then read the diff — every file the agent touched, not just the ones you expected.

Only after the checks pass and the diff looks right: commit, reset context, move to the next task. The commit is the checkpoint. The reset is what keeps subsequent tasks clean. Skipping either one means errors from this task can compound into the next.

## After implementation

### 10. Review in a separate session

Agents praise their own work — not sometimes, but structurally. The same session that built the code shares the reasoning that created it, which means it can't see what that reasoning missed.

**The cheap version:** `/clear`, then "Review all changes on this branch against PLAN.md. Identify gaps, deviations, and anything that doesn't match the spec." This is surprisingly effective because the model is genuinely better at reviewing than generating. It spots its own mistakes when forced to re-read the actual files instead of going from memory.

**The better version:** A different model reviews. Claude writes, GPT reviews (or vice versa). In 133-cycle cross-model testing, each model had consistent but different blind spots. One race condition only appeared through the multi-model pass — invisible to either model reviewing its own work.

See [Verification](./verification.md) for production setups.

### 11. "I verified it" means nothing

When the agent says "I've verified there are no violations," it hasn't actually verified anything. It's predicting the most likely next token after "I have completed the task," and that token is a statement of completion. Multiple documented cases: 4 violations found after "verified clean," tests passing but features missing, assertions that never actually ran because a base class silently blocked them.

`tsc` catches types, but it does NOT catch logic errors. In one of our experiments, `tsc` passed on code where a critical assignment was corrupted during editing. The reviewer caught it. The compiler didn't. For important features, ask the agent in a fresh session to re-read every changed file and verify against the spec.

### 12. Performance is invisible to agents

AI-generated code optimizes for correctness, not performance. In a 76K-line codebase, 118 functions were running up to 446× slower than necessary — all correct, all tests passing, none caught in code review. The patterns were consistent: naive algorithms where efficient ones exist, redundant computation, zero caching, wrong data structures.

This isn't closing with better models. Benchmarks show the best LLM achieves less than 0.23× the speedup of human experts on optimization tasks. It's a fundamental mismatch between single-pass generation and iterative optimization.

**If performance matters:** Add explicit performance requirements to your spec ("this endpoint must respond within 200ms p95"). Run a profiler on hot paths after the agent generates code. Write benchmark tests for critical functions and include them in CI. The agent won't optimize what you don't measure.

## Scaling up

### 13. Documentation beats code maps for implementation

When the codebase grows, the intuition is to give the agent more context — code maps, dependency graphs, RAG over the repo. This helps for orientation (finding the right file, understanding project structure), but it can hurt for implementation. In our A/B testing, the code-map run cost 51% more and produced a wrong architecture because the agent followed the map's structure instead of exploring the actual problem space.

Documentation works better for implementation because it's compressed knowledge. An architecture doc conveys the same information as 50 source files in a fraction of the tokens. The agent doesn't need to "discover" your architecture if it's written down.

**What to document:** Architecture decisions. Module boundaries. Data flow. How services communicate. Things a new developer would need to know in week one. The same things that make a codebase maintainable for humans make it navigable for agents.

### 14. Scope sessions to one module

Reliable agent work tops out at roughly 1,500–3,000 lines of active, in-scope code per session. Not a product limitation — a consequence of how transformer attention works over long contexts. Cross-file dependency chains beyond that length become unreliable.

Scope each session to one module, one feature, one domain. If the agent needs to understand how auth works to build a payment flow, point it at the auth documentation, not the auth source code. The doc has the contract; the source code has the implementation details that distract.

### 15. Fix the environment, not the prompt

When the agent keeps making the same mistake, don't write a better prompt. Add a lint rule. Add a test. Add a pre-commit hook. Update the documentation. Environment fixes persist across every future session. Prompt fixes persist until the context compacts.

If your tool supports hooks (Claude Code has `PreToolUse`, `PreCompact`, `Stop`), use them as automated quality gates. A hook that runs `tsc --noEmit` before every edit catches type errors before they enter the codebase. A `PreCompact` hook that dumps session state to a file preserves critical context. These are one-time setup costs that pay off every session.

One CI gate catches more bugs than a thousand lines of prompt instructions. If you find yourself adding the same rule to CLAUDE.md for the third time and the agent still ignores it — the rule doesn't belong in CLAUDE.md. It belongs in your tooling.

This is the principle GitLab's AI playbook calls "constraints as multipliers." And it's the path from using agents to engineering with them.

---

Each rule has a deeper story. The rest of this site covers the theory, the experiments, and the architecture. Start with [Principles](./principles.md) for the evidence, or [Pipeline](./pipeline.md) for the full workflow.
