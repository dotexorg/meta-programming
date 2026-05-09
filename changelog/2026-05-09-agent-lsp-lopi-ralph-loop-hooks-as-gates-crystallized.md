# What Changed: May 9, 2026

Six pages touched: index, landscape, principles, pipeline, verification, self-improvement (plus references). Two May 2026 shipped products move existing claims from "open question" to "settled" and add one new structural category to the epic. Recovery layer becomes named alongside self-improvement / pipeline / verification. Layer 2 form selection and Layer 3 closed loop both get their first publicly shipped reference implementations on the central thesis page. Hooks-as-gates closes. The 92-99% grep noise rate puts cross-codebase measurement under P#3 for the first time.

---

## Per-page deltas

### [index.md](../index.md) — central thesis page

- **Layer 2 (Intent Form Selection)** picks up its first concrete reference implementation at the small end of the form spectrum: lopi's single-imperative form (`must` / `do not` / `always` / `never`, ≤200 chars, behavior-specific) for cheap per-session capture. The form-selection space now has reference implementations at both ends — single-imperative at the small end, multi-section AGENTS.md at the large end. The decision-tree question stops being abstract.

- **Layer 3 (Closed Loop)** picks up its first publicly shipped score-threshold prevention: lopi's `LESSON_QUALITY_GATE = 0.6`. The reasoning, surfaced in source: a sub-threshold run isn't informative enough to generalise from and persisting it would degrade future retrieval. Closes a known failure mode in the academic frameworks (ExpeL, Reflexion) which assume extraction is value-positive by default.

### [principles.md](../principles.md)

- **P#3 (Exploration vs exploitation)** gains a cross-codebase empirical anchor. The May 2026 agent-lsp benchmark across five real codebases (Go, Python, TypeScript, 15K to 319K LOC) finds 92–99% of grep matches on symbol references are false positives: HashiCorp Consul `Close` 1156 grep → 12 real, FastAPI `validate` 64 → 2, Hono `Close` 15 → 1. The principle had Experiment 1 measurement before; now it has external cross-codebase measurement of the specific mechanism. Telling the agent "start with grep X" doesn't just narrow the search space; it routes 90%+ of the resulting attention onto false positives.

- **Open question "Form selection for Layer 2"** mutated: the form taxonomy gained a named seventh entry (single-imperative). The size axis now has both ends populated. The decision-tree question narrows from "what forms exist" to "which size threshold favors which form."

### [pipeline.md](../pipeline.md)

- **New section: "Recovery layer: Plan → Implement → Test → Score → Fix → Retry."** Recovery becomes a named structural category alongside self-improvement / pipeline / verification. Four independent implementations by May 2026 (Huntley's original Ralph-loop, LoopTroop, ralph-claude-code, lopi) reproduce the same four constraints: fresh context per iteration, external memory in files plus Git, one item per loop, machine-verifiable completion criterion. lopi documented in detail as the architecturally most complete reference: `git reset --hard` per failed attempt, hard turn limits, diff scope check, last-error injection into next attempt's plan, cheap→Opus model routing only after failure, post-mortem distillation into a single imperative constraint on terminal failure. Recovery (in-task, within-session, fast iteration) is a different category from our Lessons stage (post-merge, cross-session, slow promotion).

- **Open question "Pipeline crash recovery"** mutated: pattern question is settled at n=4. lopi is a usable reference architecture. What's open for our pipeline is implementation choice (in-task retry vs respec-from-clean-slate at our task shape), not pattern discovery.

### [verification.md](../verification.md)

- **"Deterministic checks first"** extended with two production-grade primitives shipped May 2026 in agent-lsp. Phase enforcement: declared phase (`scout` / `plan` / `implement` / `review`) with structural rejection of out-of-phase tool calls — hooks-as-gates pattern that Claude Code documented as concept-only in Chapter 12, now shipped as exit-code-enforced runtime gate. Speculative execution: apply hypothetical edit in memory, ask language server for diagnostic delta, commit only if delta is empty or expected. Pulls preflight one step earlier than `tsc --noEmit` — the type-check happens before the file is written, not after, and the failed attempt costs zero retry tokens.

### [landscape.md](../landscape.md) — already updated in this round

- "Semantic edit: the unsolved problem" extended with agent-lsp as a fourth angle (LSP bridge, not AST merge): 92–99% grep noise rate, 5–34× token efficiency, honest tradeoff that bridge quality inherits the underlying language server's coverage.
- New section "Recovery loops crystallize" — Ralph-loop n=4 + LESSON_QUALITY_GATE quality-gate primitive.
- **Open question "How far does AST edit generalize?"** mutated: the edit-reliability problem doesn't reduce to a single solution shape — AST-merge handles failed text replacements after the agent decided what to write; LSP-bridge handles the discovery step before the edit is written; speculative execution handles the verification step before the edit is committed. All three layers needed; open question reframes from "which approach wins" to "which combination dominates at which task profile."

### [self-improvement.md](../self-improvement.md) — already updated in this round

- "Extraction ≠ promotion" extended with form-factor paragraph: lopi's single-imperative form (one line, `must` / `do not` / `always` / `never`, ≤200 chars) plus `LESSON_QUALITY_GATE = 0.6` score gate. Form-bounding cuts model-drift at write time; quality-gating cuts noise-accumulation at the same step.

### [references.md](../references.md)

- **35e. agent-lsp** (blackwell-systems, May 2026) — full entry with bench numbers and source links.
- **35f. lopi** (konjoai, May 2026) — full entry with architecture detail, Ralph-loop n=4 sighting, TOON format reference.

---

## What didn't land this round

- **Hooks-as-gates row in the CC architecture table** moved in tier1 from `gap` to `closed by agent-lsp internal/phase/ (n=1 external, watch for 2nd)`. Surfaces in verification.md as the named primitive but the architecture table itself is tier1-internal, no reader-facing page exposes it directly.
- **Industry convergence consolidated one-liner** stays in tier1 plus tier1's three-layer LMP thesis page. The four new entries (Mitsuhiko 1/1730, Agentic World Modeling 40-author survey, agent-lsp speculative-execution, lopi recovery layer) surface scattered across landscape.md sections; a unified industry-convergence narrative belongs in a public LMP-thesis page that doesn't currently exist.
- **TOON format** (toonformat.dev v3.0, claimed ~40% fewer tokens than JSON, used by lopi for tabular pattern injection) stays in synthesis as a spawn thread, not promoted.
- **Simon Willison agent-incident anti-patterns** (Apr 27) and **AgenticQwen-30B** (Apr) remain in synthesis pending a second independent cite.
- **landscape.md and self-improvement.md word-count overflow** (3099 and 3071 words against the 1500–2000 style-guide target) is sync-debt-2 — content is correct, length needs a separate trim pass that wasn't in this round's scope.
