# What Changed: June 22, 2026

One reader-facing page touched: verification. Yesterday's acceptance-gate result was an experiment; today it got validated live on a production wallet refactor, twice, and surfaced two ideas the chapter didn't have. First, the gate guards the reviewer's claims, not just the builder's — when you point a fixing agent at an agent-generated review, the same "read the artifact before you act" discipline stops it from "fixing" findings that were never real. Second, once you have built the reviewer, the unasked question is how you know its clean bill is trustworthy and not a rubber stamp. Both came out of running the ladder on enclave: a 94-finding fix pass and a 45-requirement final conformance audit.

---

## Per-page deltas

### [verification.md](../verification.md)

- **"When deterministic gates go blind" extended with a reviewer-side blind spot.** Pointing a fixing agent at an agent-generated review, it will by default "fix" findings that were never real, editing correct code until it breaks. The same discipline that guards the builder guards the reviewer: read the artifact before acting on the claim. In one fix pass over a 94-finding report, two of the three things the gate caught were false findings refuted by reading the code, not defects in the fix. The reviewer still has to be independent and run in clean context; the gate is what stops the fix loop from manufacturing its own bugs.

- **New section: "When a clean review is trustworthy."** The meta-question once the separate, multi-model, partitioned reviewer exists: how do you know its PASS is real? Drawn from a final spec-conformance audit (45 requirements, coverage 1.00, eleven security items independently re-attacked in clean context, 11/11 confirmed). Three signals separate a trustworthy pass from a merely confident one. Discrimination: a hundred-percent confirmation with zero flagged edges is the tell, not the reassurance — the trustworthy reviewer surfaces cracks inside its own confirmed verdicts (a bigint that throws on both peers during hashing, an un-zeroable JS string, an untested nonce-cap branch held at the lower evidence tier). Reproducibility: the same two false positives were re-refuted identically across two independent passes from fresh context, which is what makes a pass bankable. And full coverage is not zero open decisions: the honest-edges section is where in/out-of-scope gets handed back to the spec owner, and a verdict that bundles a shipped capability with an unwired one (token freshness enforced, mass-revocation never wired) has to be split rather than rounded up to a clean pass.

---

## Why this is its own entry

The June 21 round was curation and promotion — moving a stabilized result into the published pages. This is different in kind: the acceptance-ladder instrument was run live on a real codebase across two task shapes it wasn't originally built for (recovery fix-pass, conformance audit), and both runs produced new reader-facing material rather than confirming old material. The instrument generalized; the chapter grew to match.
