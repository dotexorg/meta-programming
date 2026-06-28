# What Changed: June 21–22, 2026

This round was about verification, and one result leads it: a deterministic acceptance-gate only earns its keep where your tests already cover the defect. Everywhere they don't (security, crypto, timing, cleanup) the gate runs green and sees nothing, and a fresh-context reviewer is the only thing that catches the bug. That result then held up in production. It survived a live wallet refactor across two task shapes it was never built for, and turned up two more findings on the way: the gate also catches a reviewer wrongly "fixing" a false finding (not just a builder shipping a bug), and there are three concrete signals that separate a real review PASS from a rubber stamp.

Five pages moved: index, verification, principles, landscape, references.

---

## Per-page changes

### [index.md](../index.md)

The convergence argument sharpened from "trend" to "emerging standard." The thesis already said the industry is converging on language as the primary artifact. This round extends that past configuration files into the rest of the stack: independent review now ships as a default, and "evidence-gated" completion gates have hardened into a product category.

### [verification.md](../verification.md)

- **New section: "When deterministic gates go blind."** We layered a deterministic acceptance-gate (re-run the tests, check every acceptance criterion before handoff) onto a staged crypto-wallet refactor. Over nine stages with a strong worker and a green test suite, the gate caught zero real bugs. Pure redundancy. Then we seeded bugs the tests could see: the gate caught 4 of 4, but only because a test already covered each one. Then we seeded bugs the tests couldn't see: an HMAC timing leak, a byte-compare oracle, the master mnemonic written to a debug log, a `destroy()` that no longer wiped the secret. The gate caught 0 of 4. A separate multi-agent review caught all four, plus roughly six more real bugs sitting in already-green code. The rule: a gate pays off only when the work it guards is expensive, usually avoidable with a cheaper check, and precise enough not to fire on the cases you can't avoid. For bug classes with no cheap check (security, crypto, timing, cleanup), the budget belongs in review, not a fancier gate.
- **New section: "When a clean review is trustworthy."** How to tell a real PASS from a rubber stamp. Three signals. A reviewer that confirms everything and flags nothing is the tell. A trustworthy one surfaces the cracks inside its own approvals. The same false positives get refuted the same way across independent passes. And full coverage is not the same as zero open questions: an honest review hands scope calls back to whoever owns the spec.
- **Reviewer-side blind spot.** The same gate guards a reviewer, not just a builder. When one agent fixes another agent's review, it can "fix" findings that were never real and ship regressions into correct code. Reading the actual artifact before acting catches that.
- **"Four cognitive states in a stuck agent" became "How agents fail: execution and cognition."** The section grew from one model into two levels. The execution level (from 1,281 benchmark runs) names five repeatable patterns: lost in the codebase, editing the wrong file or symbol, tool thrashing, context overflow, and partial completion. The cognitive level names what's happening underneath: on-track, looping, drifting, stuck. The mapping is the point: tool thrashing is looping you can see in the log, and partial completion is the dangerous one, because it looks like success until a coverage check against the full spec catches it.

### [principles.md](../principles.md)

Principle 4, separate the builder from the reviewer, moved from advice to default infrastructure. Cursor now ships automatic review to new users; Devin runs a security review on every pull request and drafts the fix. A reviewer with fresh context is the only thing that catches the bug classes no test asserts, which is the gap a deterministic gate structurally can't close.

### [landscape.md](../landscape.md)

New section, "June 2026: what moved since," with three shifts that sharpen the April snapshot. Independent review shipped as a default (Cursor, Devin). Eval infrastructure consolidated onto a single stateful harness. And "evidence-gated" completion hardened into a product category: gates that demand proof, not prose.

### [references.md](../references.md)

New Tools section, organised by where in the pipeline each tool bets the work matters: spec, edit, navigate, verify, recover, self-improve. Tools that were scattered through the flat source list are pulled to the front as runnable artifacts, each tagged by layer and license. The page now reads tools first, then sources, then our own experiments.
