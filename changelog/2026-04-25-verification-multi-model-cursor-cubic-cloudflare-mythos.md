# What Changed: April 25, 2026

One page touched: verification. Three production-scale multi-model review case studies landed in the previous week, plus a quantified partition threshold and two named termination mechanisms.

---

## Per-page deltas

### [verification.md](../verification.md)

- **"Multi-model review" expanded** with three production case studies: Cursor BugBot (8-pass parallel ensemble + majority vote, 52% → 70% resolution across 2M+ PRs/month), cubic.dev (4 narrow specialists — Planner / Security / Duplication / Editorial — 51% drop in false positives without recall loss), and Cloudflare's CI reviewer (coordinator plus 7 specialist sub-reviewers, mixing Opus 4.7 and GPT-5.4 across roles, deployed at tens of thousands of MRs). Source pattern catalog: Zylos research, March 2026.
- **Mozilla × Anthropic Mythos data point.** Mythos found 271 Firefox vulnerabilities (fixed in Firefox 150) against 22 found by Opus 4.6 (fixed in Firefox 148). Mozilla's "defects are finite" framing reframes systematic multi-model review as enumeration of a finite bug space, not optional rigor.
- **New section: "Partition the review."** ~500 LOC threshold where AI review effectiveness flips — production data shows 30-40% cycle-time reduction below, sharp degradation above. Mechanism: effective context utilization fails at 10-40% of nominal window. Cross-zone coupling problem and the cross-cut zone solution made explicit.
- **"Bounded review iterations" open question expanded** with two named production mechanisms: explicit `complete_review` termination tool and round-limit caps (1-2 loops for CI, 3-5 interactive). Calibration question — productive second cycle vs oscillation — remains open.
