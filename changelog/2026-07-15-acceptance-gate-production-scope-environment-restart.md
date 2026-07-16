# What Changed: July 15, 2026

A green task is not a green changeset. When every task in a pipeline passes its own tests, the bugs that ship live in the gaps between tasks and in the fresh environment that local dev never sees. On an 18-task financial-auth build, all four CRITICAL defects slipped past the per-task gates and were caught only by review over the whole changeset: a flag the login flow fetched but never enforced, two individually-correct features colliding on one re-login path, and a migration that ran locally and died on fresh CI. A characterization oracle written first held bit-for-bit through every stage (89→182 tests, zero legacy regressions), but it covered only what it was built to cover — the four criticals lived outside it. The takeaway is a shape, not a tuning: guard with whole-changeset review plus a fresh-environment run, not a fancier per-task gate.

This extends the June result (a gate is blind where no test covers the defect) with production evidence and a sharper model: "blind" has three axes, not one.

One page moved: verification.

---

## Per-page changes

### [verification.md](../verification.md)

- **Production build validates the acceptance ladder from scratch.** New subsection under "When deterministic gates go blind." A tech-lead pipeline ran 18 tasks over three stages on a real financial-auth integration; first-try-pass was NO at every stage, four CRITICALs were all caught by independent review and none by the per-task gate, and a characterization oracle written before task one stayed green 89→182 with zero legacy regressions. This is the fullest test of the ladder yet — a from-scratch build, not a recovery or audit pass.
- **"Blind" has three axes, not one.** Beyond the oracle-cost gap the crypto experiment found, a per-task gate is also blind on **scope** (a revoked-device flag the login flow fetched but never enforced) and on **environment** (a journal-orphaned migration, green locally because it already ran, dead on fresh CI). The operational rule: each task passing its own tests is not correctness — the guard is whole-changeset review plus a fresh-environment run.
- **Restart-amortization: a second value mode.** When the expensive operation genuinely can't be oracled away, the gate still pays by forcing a pre-flight block — instrument every live hypothesis into one build so a single restart is decisive. A single instrumented restart named a root cause that had survived two prior debugging sessions. Gate value = max(restarts prevented, hypotheses per restart); the seeded experiment measured the first term, this one the second.
