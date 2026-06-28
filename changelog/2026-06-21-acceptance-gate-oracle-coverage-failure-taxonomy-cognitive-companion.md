# What Changed: June 21–22, 2026

This round was about verification, and one result leads it: a deterministic acceptance-gate only earns its keep where your tests already cover the defect. Everywhere they don't — security, crypto, timing, cleanup — the gate runs green and sees nothing, and a fresh-context reviewer is the only thing that catches the bug. That result then survived production. It held on a live wallet refactor across two task shapes it was never built for, and threw off two new findings on the way: the gate guards the reviewer's claims too, not just the builder's, and three concrete signals separate a real PASS from a rubber stamp.

Five reader-facing pages moved: index, verification, principles, landscape, references. The "stuck agent" section grew into a two-level failure taxonomy, execution and cognition. Separate builder/reviewer crossed from best practice to shipped default across two vendors. Two changes stayed internal: tier1 went back under its size cap, and the doc pipeline picked up a validated two-model writing step.

---

## Per-page deltas

### [index.md](../index.md) — central thesis page

- **Convergence framing sharpened from "trend" to "emerging standard."** The thesis section already argued the industry is converging on language as the primary artifact without naming it; this round extends that beat past configuration files into the rest of the stack — independent review shipped as default (Cursor, Devin), "evidence-gated" completion gates as a product category, graded reliability ladders rebuilt by uncoordinated teams. Decision: do NOT spin up a separate convergence / LMP-thesis page (it would duplicate index); strengthen the claim in place instead.

### [references.md](../references.md)

- **New "## Tools" section, organised by pipeline layer** (Spec / Edit / Navigate / Verify / Recover / Self-improve / Multi-layer). Tool entries that were scattered through the flat numbered source list (AGENTS.md, spec-kit, Kiro, Tessl, Pydantic, Cursor, Aider, Morph, Relace, Augment, agent-lsp, Promptfoo, OTel GenAI, lopi, ralph-claude-code, DSPy, CORE, claude-performance, Homunculus, Claude Code, Codex CLI, Cubic) pulled up front as runnable artifacts, each tagged by layer and access/license. The page now reads tools-first, then sources, then our experiments. The layer category is the legend: it tells you what assumption a tool made about where agents fail.

### [verification.md](../verification.md)

- **New section: "When deterministic gates go blind."** Our own experiment layered a deterministic acceptance-gate (re-run the oracle, check criteria coverage before handoff) onto a staged crypto-wallet refactor. Over nine stages with a strong worker and a green suite, the gate caught zero real defects — pure redundancy. Seeding test-visible bugs: gate caught 4/4, but only because the suite already tested each. Seeding test-invisible bugs (HMAC timing leak, byte-compare oracle, mnemonic to a debug log, a `destroy()` that no longer wipes the secret): gate caught 0/4, independent multi-agent review caught 4/4 plus ~6 genuine bugs cold in already-green code. The rule: a gate pays off only when the operation is expensive, frequently avoidable via a cheaper oracle, and precise enough not to fire on the unavoidable cases — the deciding variable is the avoidable fraction, not raw cost. For defect classes with no cheap oracle (security, crypto, timing, cleanup), invest in review, not a more elaborate gate. Includes the phase-misscope corollary: a gate tuned for one workflow phase misfires in another; scope when it fires.
- **Reviewer-side blind spot added** (validated live, enclave fix pass on a 94-finding agent review): the same gate guards against "fixing" false findings, not just builder lies — 2 of 3 gate_catches were false-positive refutations caught by reading the artifact before acting. Without it, an agent fixing an agent review ships regressions into already-correct code. rung-4 honesty (batched independent reviewer, no self-grant) and honest exit codes (separating a pre-broken baseline from own changes) held under the same loop.
- **New section: "When a clean review is trustworthy."** How to tell a trustworthy PASS from a rubber-stamp, from a 45-requirement spec-conformance audit (coverage 1.00, 11/11 security items re-attacked in clean context). Three signals: discrimination (a 100% confirm with zero flagged edges is the tell — the trustworthy reviewer surfaces cracks inside its own CONFIRM verdicts), reproducibility (same false positives re-refuted identically across independent passes), and full-coverage ≠ zero-open-decisions (the honest-edges block hands scope calls back to the spec owner; a verdict bundling a shipped capability with an unwired one must be split).

- **"Four cognitive states in a stuck agent" → "How agents fail: execution and cognition."** Section widened from the Cognitive Companion 4-state model alone into a two-level taxonomy. Execution level (Sourcegraph CodeScaleBench, 1,281 runs): five repeatable patterns — lost-in-codebase, wrong-file/wrong-symbol, tool thrashing (96 calls/84 min/6 reversals vs 5 calls/4.4 min), context overflow, and partial completion (2/7 files, 0.32 vs 0.80). Cognitive level: ON_TRACK / LOOPING / DRIFTING / STUCK. The mapping is the point — tool thrashing is looping made visible, lost-in-codebase is drifting made visible; the cognitive state is hard to observe directly while the execution trajectory sits in the log. Partial completion named as the dangerous one because it wears the mask of success — only a coverage check against the full spec catches it.

### [principles.md](../principles.md)

- **P#4 (Separate builder from reviewer)** strengthened from advice to default infrastructure. Cursor ships automatic review for new users; Devin runs a security review on every pull request that finds the vulnerabilities scanners miss and drafts the fix. The claim moved from "best practice teams adopt" to "table stakes the tools ship." The generalising reason is tied to the new verification result: a reviewer with fresh context is the only thing that catches the defect classes no test asserts — the gap a deterministic gate structurally can't close.

### [landscape.md](../landscape.md)

- **New section: "June 2026: what moved since."** Three shifts that sharpen the April snapshot. (1) Separate review shipped as default (Cursor auto-review, Devin per-PR security review). (2) Eval infrastructure consolidating — harbor became the de-facto stateful-eval harness, underpins Terminal-Bench 2, hosts LangSmith/Daytona/E2B/Modal sandboxes behind one interface. (3) "Evidence-gated" hardened into a product category — completion gates that demand proof not prose (fenced report or reject, real working-tree checks, exit-code-gated runtime commands, non-self-attestable review rung); open-multi-agent-kit, agent-completion-gate, claude-prove-done rebuild the acceptance-ladder idea independently. Plus the academic "machine studying" line reframing continual learning as expertise from study compute.

---

## What didn't land this round

- **Failure taxonomy stayed a verification section, not a standalone page.** The Sourcegraph-5 × Cognitive-4 synthesis was owed as a possible new article; folding it into the existing verification section avoided duplicating the cognitive-states and continuous-evaluation content already on that page. No new NAV entry.
- **No standalone convergence / LMP-thesis page.** Considered (the unified industry-convergence narrative flagged as missing in the May round); rejected as an index duplicate. The convergence-as-industry-standard argument was strengthened in index.md in place instead.
- **tier1 trim (internal).** The Active-Questions detailed bodies moved out of tier1 into a new `tier2-active-questions.md`, bringing tier1 back under the ~9.6K cap after the Exp-10 promotion pushed it to ~17K. Reader-facing pages unaffected.
- **Dual-model doc writing (internal tooling).** The doc-editor pipeline gained a validated procedure: run two pinned Opus writers (4.5-Nov and 4.8) on the same page-spec and splice — 4.5-Nov tends to tighter editorial judgment, 4.8 to instruction completeness, and the merge beats either alone. Both sections shipped this round were written that way.
- **Content-pipeline backlog drained.** 220 pending items triaged — six findings to the KB, four distinct tools to the catalog, the rest dismissed as clones and SEO noise.
- **landscape.md / self-improvement.md word-count overflow** remains sync-debt from the prior round; the June addendum adds to landscape's length. A trim pass is still owed, out of scope here.
