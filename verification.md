# Verification: why agents can't review their own work

The agent that built the code cannot meaningfully review it. Same context, same assumptions, same blind spots. A different verdict is statistically unlikely. External verification is not optional overhead; it is the mechanism that converts agent output into reliable software.

---

## Why self-review fails

Agents praise their own work. Confidently, consistently, and incorrectly. Across our pipeline experiments, implementation agents asked to self-evaluate their output returned positive verdicts even when the code contained logic errors, dead files, and broken runtime behavior. It was not occasional. It was the default every time. The agent that produced an artifact shares the reasoning path that created it. It cannot see what that reasoning led it to miss.

A subtler version is harder to catch. At higher thinking levels, an agent will formally confirm it will follow a rule, then proceed to violate it in implementation. Not through misunderstanding, but through a "workaround" framing. The stated intent passes inspection; the actual behavior does not. Standard benchmarks don't detect this pattern. A verification pass that trusts a model's self-report on spec adherence is checking the wrong signal entirely.

The cost surfaces fast at scale. One engineering organization rolled out agents across 21,000 developers and hit multiple Sev-1 incidents in the following quarter — including a six-hour outage. The internal diagnosis: *"the creation layer accelerated, verification layer stayed the same size."* More code shipping faster into an unchanged review process is risk accumulation, not productivity.

At the individual level, the gap between perceived and measured speed is equally stark. Developers reported feeling 20% faster using AI tools. They were actually 19% slower. The dominant cause being time spent reviewing and correcting AI output. The speed gain went to generation. The time debt went to verification.

A February 2026 follow-up (arxiv 2602.20292) sharpened the picture. Sixteen developers forecast a +24% speedup before the measurement started. Their actual outcome: −19%. A 43-percentage-point calibration error between belief and measurement. The point isn't that AI tools are slow. It's that internal sense of productivity is orthogonal to productivity. Teams that make decisions based on how fast the work *feels* are making decisions against the measurement.

## Deterministic checks first

The cheapest verification is automated. Run `tsc --noEmit`, your linter, and your test suite before any LLM review token is spent. These catch a large class of errors in milliseconds, with no hallucination risk, for free. They belong at the front of every pipeline.

But deterministic gates have a hard ceiling. In one of our pipeline runs (Experiment 5), an agent implementing a router with correct types and passing tests broke `JIRA_SERVICE_USER` through a `||` operator incorrectly split across lines during editing. The type-checker passed. All tests passed. A separate reviewer agent running the affected code path caught it. `tsc` did not. The error was a runtime logic error, invisible to static analysis.

The feedback signal hierarchy, from cheapest to most expensive: syntax and types → unit tests → integration tests → observability data → visual and E2E verification. Each layer catches a different failure class. Skipping the cheap layers doesn't save time. It pushes costs up to the expensive ones.

Pre-flight checks also eliminate waste at the review stage. In Experiment 3 (24-file type refactor), review agents required three iterations before passing. Both failure types were missing imports and duplicate exports from wildcard re-exports. A type-check and a grep would have caught them before any reviewer was spawned. After that run, pre-flight became a permanent pipeline stage: type-check, lint, and targeted pattern-match before any LLM review. Deterministic first, LLM second.

Two preflight primitives shipped in May 2026 push this stage one step earlier than `tsc --noEmit`. The first is **phase enforcement**: a phase is declared (`scout`, `plan`, `implement`, `review`) and the runtime structurally rejects tool calls that don't fit it, by exit code rather than by prompt hygiene — the same hooks-as-gates pattern Claude Code documents as a concept, now shipped as an enforced runtime contract in an open-source binary (agent-lsp). The second is **speculative execution**: a hypothetical edit applies in memory, the language server returns its diagnostic delta, and the edit commits to disk only if the delta is empty or expected. A failed speculation costs zero retry tokens because the file was never written. Neither primitive fixes model quality; both are gates that prevent the existing model from corrupting state through plausible-but-wrong edits, and both push the verification cost from after-the-fact review to before-the-fact prevention. 🟡

AI-generated code demands more review effort by default. Across 8.1 million pull requests, AI code required 1.7× more review revisions than human-written code. Pre-flight checks don't close that gap; they ensure reviewers spend cycles on logic and architecture, not compiler errors.

## When deterministic gates go blind

A deterministic gate catches defects that an oracle already tests for. If no oracle tests for a defect class, the gate sees nothing.

We ran this experiment on a staged crypto-wallet refactor with nine handoff points. The acceptance gate (re-run tests, check type coverage, verify criteria) caught zero real defects across the full run. Every "catch" duplicated something the test suite had already flagged. With a strong worker, a clear spec, and a green suite, the gate was redundant.

To find the blind spot, we seeded defects. Round one: four functional auth bugs. We removed the timestamp-skew window, skipped the replay check, skipped the host check, skipped the body check. The gate caught all four. But the suite already had a direct test for each, so this measured redundancy over a good test suite, not the gate's added value.

Round two: four defects the suite couldn't see. An HMAC comparison that returns early and leaks timing. A byte-compare timing oracle. The master mnemonic written to a debug log. A destroy() that no longer wipes the secret from memory. The deterministic gate caught zero. Timing side-channels, secret logging, missing zeroization — none have tests because the suite doesn't assert those invariants. A green run on an untested property tells you nothing.

An independent multi-agent review over the same diff caught all four. It also surfaced roughly six more genuine bugs in already-green shipped code, including a canonical hash function that throws on the BigInt amounts a production wallet actually uses.

The rule: a gate on an operation pays off when three conditions hold. The operation is expensive. It's frequently avoidable via a cheaper oracle. And the gate is precise enough not to fire on unavoidable cases. The deciding variable is the avoidable fraction, not raw cost. Where a cheap oracle covers the defect, the gate is redundant. Where no oracle covers the defect class at all, the gate is blind.

This shapes how you allocate verification budget. Spend gate budget where the gated operation is both expensive and often avoidable: a static check before an engine restart, a lint pass before a full build. For defect classes that have no cheap oracle (security, crypto, timing, cleanup), invest in an independent review pass instead. A more elaborate deterministic gate won't help here; the limit is oracle coverage.

Precision is where a gate quietly turns negative. The same static-check-before-restart gate pays off while the restart is usually avoidable, but turns to noise during ordinary build-deploy-test work where the restart is necessary. A gate tuned for one phase misfires in another, and the fix is to scope when it fires rather than keep tuning its rules.

There is a second blind spot, and it sits on the reviewer's side. Point a fixing agent at an agent-generated review and by default it starts "fixing" findings that were never real, editing correct code until it breaks. The same discipline that guards the builder guards the reviewer: read the artifact before you act on the claim. In one fix pass over a 94-finding report, two of the three things the gate caught were false findings refuted by reading the code, not defects in the fix. An agent that skips that step ships regressions into code that was already correct. The reviewer still has to be independent and run in clean context, but the gate is what stops the fix loop from manufacturing its own bugs.

A green suite that clears a gate is not evidence of correctness for invariants nobody tested. For those, you need a reviewer who isn't the builder.

## Separate the builder from the reviewer

The reviewer needs a different session with clean context and no shared state from the builder. Not a preference — a requirement.

A reviewer starting from the spec and the diff sees only what was delivered. The implementation agent knows what it tried to do, and that knowledge contaminates its ability to see what it missed. Anthropic names this "self-evaluation bias" explicitly. We observed it independently across multiple experiments. Mario Fernandez at Sentry documented it publicly in March 2026.

Passive review is not enough. Claude Code's verification agent guards against a specific pattern: reading the code, writing PASS, and running zero commands. They call it "verification avoidance," and it's common enough to need a name.

The minimum viable reviewer runs type-check, affected tests, anti-pattern grep, and verifies the diff against the original spec — not the agent's description of it. That distinction matters because the reviewer should never see the builder's stated intentions, only requirements and output.

One sentence changes everything. The reviewer receives not "review this code" but *"you must find issues. Zero findings triggers a halt."* Courtesy review → adversarial review. The structural change is a single instruction.

## Multi-model review catches different things

Running the same model twice on the same artifact means the same blind spots, twice. Different models fail in different places, which makes them complementary rather than redundant.

We tested this across 133 cycles and 42 development phases with four models in strict isolation — no model saw another's output. The specialization was consistent: GPT caught Python idioms and security holes, while Claude caught reasoning chains and architecture drift. One race condition in an async handler only appeared through the multi-model pass; neither model found it alone across multiple individual reviews.

This is infrastructure now. A `codex-plugin` for Claude Code runs GPT-based review inside a Claude-driven pipeline. Cross-model, officially endorsed by OpenAI. Mozilla's Star Chamber fans out to multiple providers for consensus. Practical split: Claude for architecture (structure, coupling, boundaries). GPT for security (injection, error handling, language footguns). They don't see each other's output. Independence is the point.

Specialization scales further than two models. A March 2026 catalog (Zylos research, multi-model code review convergence) names five recurring patterns. The two with the strongest production data are parallel ensemble with a vote, and specialist routing. Cursor's BugBot runs eight passes per change with randomized file ordering and takes a majority vote — resolution rate moved from 52% with a single pass to 70% across more than two million PRs per month. cubic.dev splits review into four narrow agents (Planner, Security, Duplication, Editorial) and reports a 51% drop in false positives without losing recall. Cloudflare's CI reviewer (April 20, 2026 blog post) deploys the same shape at higher granularity: a coordinator plus seven specialist sub-reviewers — security, performance, quality, documentation, release, compliance, internal Codex — each in an isolated session, mixing Opus 4.7 and GPT-5.4 across roles. Their stated lesson: a naive "diff into prompt" floods reviewers with hallucinated syntax errors and vague suggestions like "consider adding error handling" on functions that already have it. Specialization is a prerequisite for production-scale review, not a refinement of it.

The strongest single data point came in April 2026. Anthropic's Mythos research model, deployed by Mozilla against Firefox source, found 271 vulnerabilities — fixed in Firefox 150. Opus 4.6 on the same codebase had previously found 22, fixed in Firefox 148. Mozilla's own framing: *"So far we've found no category or complexity of vulnerability that humans can find that this model can't... The defects are finite, and we are entering a world where we can finally find them all."* The shift is from "find some bugs" to "enumerate the finite bug space." Once that's true, the argument for systematic multi-model review changes character. It stops being optional rigor and starts being how the regime works.

## Partition the review

Reviewer effectiveness has a hard size ceiling. Production data across multiple platforms shows AI review delivers 30-40% cycle-time reduction on PRs under 500 lines of code. Above 500 LOC, returns drop sharply — large diffs overwhelm the context window and reviewers fall back to surface pattern-matching (Zylos, March 2026). The mechanism is direct. Effective context utilization fails at 10-40% of the nominal window depending on task type, well before the advertised maximum. A 1,500-line diff fed to a 200K-window reviewer is not a 200K-token problem — it is a degraded reviewer trying to reason about a soup.

The implication is operational. Chunking large PRs is a prerequisite, not an optimization. If a change exceeds the threshold, partition before reviewing rather than after. Pi-reviewer's zone-partition step encodes this rule directly. The harder problem is cross-zone coupling: fixes in zone A that violate invariants enforced in zone B will pass both zone reviews independently. A separate cross-cut zone — one reviewer that reads only the diff and the shared API surface — closes that gap at modest cost without re-introducing the soup. The threshold is also domain-sensitive: a 500-line config refactor is far less context-bound than a 500-line state-machine rewrite. Treat the number as the breakpoint where the tradeoff *starts* to flip, and budget partition cost when a diff approaches that range, not after the review fails.

## When a clean review is trustworthy

A reviewer that passes everything and flags nothing is the dangerous kind, because it looks like diligence. Once you have built the separate, multi-model, partitioned reviewer, the next question is the one nobody asks: how do you know its clean bill is real and not a rubber stamp? Three signals separate a trustworthy pass from a merely confident one.

The first is discrimination. A review that returns a hundred percent confirmation with zero caveats is the tell, not the reassurance. We ran a final spec-conformance audit over a wallet refactor: 45 requirements, each verdict tied to a cited file and line, eleven security-critical items independently re-attacked in clean context. All eleven confirmed. That number is only believable because the same reviewer, inside those confirmed verdicts, surfaced edges it was never required to report: an integer above 2^53 that throws on both peers during hashing (a mutual failure, not a bypass), a reconstructed mnemonic string that JavaScript cannot zero out, a nonce-cap branch that was read but never tested. A reviewer that volunteers the cracks in its own pass is reasoning about the code. A flat hundred percent with no edges is pattern-matching the happy path.

The second is reproducibility. A verdict you can re-run and get the same answer is what makes a pass bankable. The same audit re-examined two findings that an earlier, separate pass had refuted as false positives: a relay that supposedly leaked a frame early, a signature that supposedly skipped canonicalization. Both were refuted again, identically, from fresh context. A reviewer that converges on the same refutation of the same artifact across independent runs is not getting lucky.

The third is that full coverage is not the same as zero open decisions. The audit reached complete coverage, with every requirement carrying an artifact, and still handed three decisions back to a human. One verdict had bundled a shipped capability (token freshness, fully enforced) with one that was never wired (mass revocation of tokens issued before a cutoff, with no caller and a missing route). The reviewer flagged the split rather than rounding it up to a clean pass. That is the highest-signal part of any conformance report: the honest-edges section is where scope questions get returned to the person who owns the spec. A verdict that fuses something built with something merely intended has to be split, and the unbuilt half is a scope decision the reviewer cannot make for you.

## How agents fail: execution and cognition

Agent failure happens on two levels, and confusing them sends you fixing the wrong thing. The execution level is what the agent observably does wrong: the files it touches, the tools it calls, the loops it runs. The cognitive level is the internal state that produces those moves. They map onto each other. And the most dangerous failure is the one that looks like success.

Start with the execution level, because it's visible. Sourcegraph's CodeScaleBench scored 1,281 agent runs across 40-plus large repos in nine languages, and the failures cluster into five repeatable patterns. *Lost-in-codebase*: an agent reads a file, follows its imports, and the branching explodes until it's drowning. *Wrong-file or wrong-symbol*: lexical search surfaces a name match but can't rank by structural role, so the agent edits the decoy. *Tool thrashing*: one run burned 96 tool calls over 84 minutes with six reversals, where structured search did the identical job in 5 calls and 4.4 minutes. *Context overflow*: handing the agent more tools made things worse, not better, because it had no strategy for choosing among them. And *partial completion*: modifying 2 of 7 required files and still passing tests that only cover what changed, scoring 0.32 against the 0.80 of a full edit. Each pattern points to an infrastructure fix rather than a model swap.

The cognitive level explains why those patterns recur. The Cognitive Companion work from IBM Dublin names four reasoning states. On-track means real progress. Looping means revisiting the same arguments without advancing. Drifting means relevance is steadily declining. Stuck shows up as shortened responses and visible uncertainty. On small 1.5B-parameter models, roughly 30% of hard reasoning tasks slid into one of the degraded states. The right response depends on which one: looping wants a session reset, drifting wants a re-read of the spec, stuck usually needs a human.

The two levels are the same failure seen from outside and inside. Tool thrashing is looping made visible. Lost-in-codebase is drifting made visible. The mapping matters because the cognitive state is hard to observe directly, while the execution trajectory sits right there in the log. You catch the inner failure by watching the outer one.

Detecting it has settled into two architectures. A companion model runs a short structured prompt every couple of turns (about 80 tokens at temperature 0.3) to classify the main agent's state; it cuts repetition 52 to 62% at roughly 11% overhead and works against any closed API. A probe-based companion instead trains a linear classifier on the agent's layer-28 hidden states, reaching 0.84 AUROC with zero runtime cost. But it needs open weights, so a closed API rules it out. Production guards stop short of both. LangGraph's recursion_limit counter, AutoGen's termination conditions, and OpenHands' StuckDetector with its five heuristics all catch mechanical loops while missing semantic drift, which is steady progress in a direction that stopped being right several steps ago.

The failure to fear is partial completion, because it wears the mask of success. It survives a casual glance. It can survive a green build. Only a coverage check against the full spec catches the three files that never got touched. That's the argument for trajectory-level signals (which files were modified, the read-to-edit ratio, retry counts), since they turn an invisible failure into a measurable one. The continuous-evaluation section shows how to wire those same signals into live gates.

## Adversarial probes over courtesy checks

Knowing *what to look for* is what separates a verification agent from a politeness layer. Claude Code's verification agent (v2.1.88) carries a fixed list of reasoning shortcuts it monitors in its own output and actively rejects when it detects them.

The self-rationalization patterns it catches and reverses:
- "The code looks correct based on my reading"
- "The implementer's tests already pass"
- "This is probably fine"
- "This would take too long"

The probe list is equally concrete: does a duplicate create crash? What happens with 0, -1, empty string, MAX_INT? Same mutating request twice? Delete a non-existent ID? Not "does it seem robust." Specific attack scenarios, run as commands, with captured output.

The format enforces accountability: every check requires `Command run:`, `Output observed:`, `Result: PASS/FAIL`. A check without command output gets rejected by the caller. PARTIAL is valid only for environment limitations. Never for "unsure." The principle transfers to any reviewer: if you can't show what you ran and what it returned, you haven't reviewed anything.

## Trajectory and observability

Output tells you whether the result was correct. Trajectory tells you how the agent got there: which tools fired, in what order, how many retries, and whether it read files before editing them.

A correct output after 30 reads and 15 retries is a completely different system from a correct output after 4 reads and 0 retries. Same result, but the stability profile is night and day — and an output-only view can't tell the difference.

OpenTelemetry's GenAI semantic conventions standardize this: `gen_ai.chat` spans for LLM calls, `agent.invoke` for agent steps, `tool.execute` for tool calls. Datadog, Honeycomb, and New Relic support the spec natively. LangChain, CrewAI, AutoGen, and AG2 emit OTel spans natively. The infrastructure exists and is ready to use.

Promptfoo adds trajectory assertions: `tool-used`, `tool-args-match`, `tool-sequence`, `step-count`, `goal-success`, as testable properties in a three-layer testing model: black-box (final output), component (each step in isolation), and trace-based (full reasoning path). A token diagnostic gives a quick signal. High prompt tokens, low completion tokens = the agent is reading files. Healthy. Low prompt, high completion = you're testing the model's memory, not the agent's behavior.

Our `trajectory.py` parses Pi session JSONL into structured tool events: read-before-edit ratio, error cascades, retry counts, cost per session. It served as the measurement instrument for Experiment 7, tracing 366 sessions and 16K tool events to isolate the root cause of our edit error rate. The gap: `trajectory.py` is an analysis tool, not a CI gate. Assertions run after the fact rather than blocking a bad run in progress. The next step is converting analysis into automated gates that halt degraded runs rather than just documenting them.

## Continuous evaluation, not vibe checks

Manual spot-checking is not an evaluation strategy. It is a feeling. A prompt working 99% of the time in manual testing can silently degrade to 92% accuracy from model weight drift alone. No visible signal in the specific outputs you happen to review. Google Cloud named this the "vibe check trap" in February 2026: manually chatting with an agent to see if it "feels right" provides no protection against gradual degradation.

The response is continuous evaluation with deterministic assertions: the same 20–50 representative tasks run against the live pipeline on a schedule. Promptfoo's CI/CD integration implements this as quality gates: fail the build if pass rate drops below 95%, run scheduled security scans, track regression deltas between model versions. The evaluation set is fixed; only the agent's behavior changes between runs. When any metric trends down over three consecutive runs, it warrants investigation before the next feature ships.

Our own Opus degradation incident showed what undetected drift looks like. Over three days, every metric moved in the wrong direction: the agent stopped reading before editing, thinking depth cratered, cost per task spiked, and the model began violating its own documented conventions — with CLAUDE.md still in context. None of this was visible in any individual task output. The failure was only detectable in trajectory metrics compared against a prior baseline. Exactly the gap that continuous evaluation would have caught before it compounded.

The practical minimum: track cost per task, pass@1 rate, and Read:Edit ratio as time series. When any metric crosses 2σ above its baseline across three sessions, investigate before shipping. The trajectory assertions from `trajectory.py` become the input; the threshold and alerting are engineering work, not research.

## Open questions

**Automated degradation detection.** The April 2026 incident documented the signature clearly: Read:Edit ratio, thinking depth, stop-hook violations, and cost all moved together while CLAUDE.md was in context and code-level quality checks passed. The metrics exist in `trajectory.py`. What doesn't exist is an automated detector that fires before the fourth degraded session rather than after. Trajectory assertions as CI gates are the prerequisite. The threshold validation requires establishing what constitutes normal variance versus genuine model-quality shift. A sample-size question we haven't resolved.

**Reviewer baseline problem.** The current reviewer sees the current codebase state, not the delta. A pre-existing type error is indistinguishable from one the agent introduced. Fixing this requires a baseline snapshot: run `tsc` on main before the task, capture the error set, compare the post-task error set against that baseline rather than zero. Not implemented. One-day engineering work with direct impact on review accuracy.

**Bounded review iterations.** Nothing in our pipeline prevents a stubborn reviewer and a confused worker from cycling indefinitely. Production deployments close this with two named mechanisms (Zylos, March 2026): explicit termination tools, where the agent invokes a `complete_review` action rather than ending on natural-language convergence, and round-limit caps — typically 1-2 loops for CI, 3-5 for interactive review. Neither is implemented in our pipeline today. The minimum is a counter on review cycles plus a forced human escalation on overflow. The harder calibration question — what counts as a productive second cycle versus an oscillation — is still open. We do not have data on it.
