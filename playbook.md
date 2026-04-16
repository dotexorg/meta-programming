# 15 Rules That Actually Work

You don't need to read the full site to get results. These 15 rules cover 80% of what makes agent-assisted development succeed or fail. Each one is self-contained. Pick any, apply today, skip the rest.

The rules are backed by A/B tests, production incidents, and a year of daily agent use. The theory lives in the other pages. This page is the cheat sheet.

Examples use Claude Code, but most rules apply to any coding agent — Cursor, Codex, Windsurf, Copilot. The underlying constraint is the same: an LLM with a context window, a set of tools, and your instructions. Where a rule is Claude Code-specific (like `/clear`), the equivalent in your tool does the same thing.

## Setup

### 1. Keep CLAUDE.md under 200 lines

The model can follow roughly 200 rules at a time. The system prompt already consumes about 50. That leaves you ~150. A 500-line CLAUDE.md doesn't make the agent smarter — it makes it selectively deaf. Research across multiple agents and LLMs confirms: context files over 500 lines actively reduce success rates.

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

### 2. Sonnet by default, Opus for judgment calls

Sonnet handles 80% of tasks at 40% of the cost. Bug fixes, feature implementation, tests, code review — all Sonnet territory. Switch to Opus for: complex multi-file refactors, architecture decisions, subtle logic debugging where the agent needs to hold competing constraints.

In controlled testing, Sonnet with high thinking outperformed Opus with medium thinking on rule-following tasks — and cost 5× less. The thinking level matters more than the model for most work.

### 3. New session for every task

Long sessions don't get more done. They get noisier. Every message re-sends the full conversation history. By message 30, the agent has forgotten your database schema. By message 50, it's guessing at your auth flow.

One task → verify → commit → `/clear` or new session → next task. The reset isn't optional. A 106-turn task in our pipeline testing didn't contaminate subsequent tasks — because we reset between them.

If you're doing a feature with multiple parts, write a PLAN.md first, then execute one phase per session.

## Before you code

### 4. Three lines before every task: DO / DO NOT / GLOSSARY

The single highest-leverage habit. Three lines that eliminate most failures before they happen.

- **DO**: What to build. One feature, scoped tight. Include acceptance criteria.
- **DO NOT**: Which files, APIs, and patterns to leave alone.
- **GLOSSARY**: Any term that could mean two things. "Fast" = under 200ms p95. "Reply" = the Telegram reply-to-message API, not a text response.

In a direct test, two consecutive failures without a spec. Both failures were deterministic — they'd have happened every time. Adding `DO NOT modify classifyTicket` prevented both. Three words.

See [Specification](./specification.md) for the full framework.

### 5. Plan first, then code. Every time

"Obvious on big features, overkill on small ones" — except it's never overkill. Skipping the plan on small features is where bugs hide, because nobody reviews what seemed trivial.

Write the plan. Pressure-test it: "what breaks if we do this? what are we assuming?" Review the plan before writing a line of implementation. If the agent proposes something you don't understand during planning (barrel re-exports, unnecessary abstractions), question it. In our testing, an anti-pattern was caught and removed at spec review that would have shipped to production without it.

If you use Cursor: Plan Mode (`Shift+Tab`). If you use Claude Code: write a PLAN.md and have the agent work from it.

### 6. Commands, not prose

`run: npm test -- --coverage && assert coverage > 80%` changes behavior. `We value thorough testing and aim for high coverage` changes nothing. Across controlled runs, prose instructions produced zero measurable effect on agent behavior. Executable commands did.

Every behavioral requirement should be a verifiable command or binary condition. If you can't check whether the agent followed it, it's not a rule — it's a wish.

## During implementation

### 7. Agent loops? Stop, clear, restart

The fix→break→fix→revert cycle happens when contradictory constraints pile up in context. The model tries to satisfy your current error AND every failed fix attempt AND the original bug simultaneously. It can't, so it alternates.

Don't add more context "to help." More context is more room to spiral.

**Do this:**
1. Stop. Don't send another message.
2. `/clear` or start a new session.
3. Write one focused prompt: current state of the code + what you want + DO NOT constraints.

If the same loop happens twice on a clean start, the problem is your spec, not the model.

### 8. Context degrades at 30–40%, not at the limit

Don't wait for auto-compact to fire at 90%. Effective reasoning degrades well before the nominal limit. Complex tasks can collapse at 10–40% utilization depending on task complexity.

**Practical approach:**
1. Turn off auto-compact in settings.
2. Monitor context usage (use `/status` or a status-line extension).
3. At 50–60%, tell the agent: "Summarize current state, decisions, and remaining TODOs to HANDOFF.md."
4. `/clear`, then: "Read HANDOFF.md and continue."

You control what survives. Auto-compact lets the model decide, and the model has primacy and recency bias — it keeps the beginning and end, drops the middle.

### 9. One task, one commit, one reset

Atomic task execution: implement one thing → run tests → verify the diff → commit → reset context → start the next task fresh. This is the single pattern that prevents most compounding failures.

Why it works: errors that slip through on task 3 don't poison tasks 4 through 7. Context is clean. The agent can't reference a wrong approach from three tasks ago because that context doesn't exist.

If a task requires touching 30+ files, it's not one task. Decompose it before starting.

## After implementation

### 10. Review in a separate session

Agents praise their own work. Not sometimes — structurally. The same session that built the code cannot objectively review it.

**The cheap version:** `/clear`, then "Review all changes on this branch against PLAN.md. Identify gaps, deviations, and anything that doesn't match the spec." Surprisingly effective. The model is genuinely better at reviewing than generating — it spots its own mistakes when forced to re-read the actual files instead of going from memory.

**The better version:** Different model reviews. Claude writes, GPT reviews (or vice versa). In 133-cycle cross-model testing, each model had different blind spots. A race condition surfaced only under multi-model review — invisible to any single model reviewing its own work.

See [Verification](./verification.md) for production setups.

### 11. "I verified it" means nothing

When the agent says "I've verified there are no violations" — it hasn't. It's predicting the most likely next token after "I have completed the task," which is a statement of completion. Multiple documented cases: 4 violations found after "verified clean." Tests passing but features missing. Assertions that never actually ran because a base class silently blocked them.

**After the agent says "done":**
1. Run `tsc`, `npm test`, linter — deterministic checks catch free errors.
2. Read the diff yourself. Every file the agent touched.
3. For important features: ask the agent (in a fresh session) to re-read every changed file and verify against the spec.

`tsc` catches types. It does NOT catch logic errors. In one of our experiments, `tsc` passed on code where a critical assignment was corrupted during editing. The reviewer caught it. The compiler didn't.

### 12. Performance is invisible to agents

AI-generated code optimizes for correctness, not performance. In a 76K-line codebase built with Claude Code, 118 functions were running up to 446× slower than necessary. All correct. All tests passing. None caught in code review. The patterns: naive algorithms where efficient ones exist, redundant computation, zero caching, wrong data structures.

Benchmarks show the best LLM achieves less than 0.23× the speedup of human experts on optimization tasks. This isn't closing with better models — it's a fundamental mismatch between single-pass generation and iterative optimization.

**If performance matters:** Run your own profiler/benchmarks on agent-generated code, especially hot paths. Don't trust "it works" as proxy for "it's fast."

## Scaling up

### 13. Documentation beats code maps

When the codebase grows, the intuition is to give the agent more context — code maps, dependency graphs, RAG over the repo. Counter-intuitively, this often hurts. In our A/B testing, the code-map run cost 51% more and produced a wrong architecture. Without the map, the agent explored broadly and found the right approach.

Documentation works better because it's compressed knowledge. An architecture doc conveys the same information as 50 source files in 1% of the tokens. The agent doesn't need to "discover" your architecture if it's written down.

**What to document:** Architecture decisions. Module boundaries. Data flow. How services communicate. Things a new developer would need to know in week one. The same things that make a codebase maintainable for humans make it navigable for agents.

### 14. Scope sessions to one module

Reliable agent work tops out at roughly 1,500–3,000 lines of active, in-scope code per session. Not a product limitation — a consequence of how transformer attention works over long contexts. Cross-file dependency chains beyond that length become unreliable.

Scope each session to one module, one feature, one domain. If the agent needs to understand how auth works to build a payment flow, point it at the auth documentation, not the auth source code. The doc has the contract; the source code has the implementation details that distract.

### 15. Fix the environment, not the prompt

When the agent keeps making the same mistake, don't write a better prompt. Add a lint rule. Add a test. Add a pre-commit hook. Update the documentation. Environment fixes persist across every future session. Prompt fixes persist until the context compacts.

One CI gate catches more bugs than a thousand lines of prompt instructions. If you find yourself adding the same rule to CLAUDE.md for the third time and the agent still ignores it — the rule doesn't belong in CLAUDE.md. It belongs in your linter, your test suite, or your CI pipeline.

This is the principle GitLab's AI playbook calls "constraints as multipliers." And it's the path from using agents to engineering with them.

---

Each rule has a deeper story. The rest of this site covers the theory, the experiments, and the architecture. Start with [Principles](./principles.md) for the evidence, or [Pipeline](./pipeline.md) for the full workflow.
