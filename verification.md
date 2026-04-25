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

AI-generated code demands more review effort by default. Across 8.1 million pull requests, AI code required 1.7× more review revisions than human-written code. Pre-flight checks don't close that gap; they ensure reviewers spend cycles on logic and architecture, not compiler errors.

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

## Four cognitive states in a stuck agent

The "agent is looping" intuition has been formalised. An April 2026 paper (Cognitive Companion, IBM Dublin) names four states a reasoning agent can be in: `ON_TRACK` (making progress), `LOOPING` (revisiting arguments without advancement), `DRIFTING` (relevance declining), `STUCK` (shortened responses, visible uncertainty). The last three are distinct enough that different responses apply to each. Looping needs a session reset. Drifting needs a re-read of the spec. Stuck often needs human help.

Two detector architectures exist. An LLM-based companion runs a short structured prompt every couple of turns (temperature 0.3, ~80 tokens) that classifies the state of the main agent. It cuts repetition by 52–62% at ~11% overhead, and it works against any model you can call over an API. A probe-based companion trains a linear classifier on hidden states at layer 28 of the reasoning model, achieves AUROC 0.84 with zero runtime overhead, and requires open-weight models (Gemma, Llama, Qwen). The probe is cheaper but unavailable through the Anthropic API; the LLM companion is what we can actually deploy today.

The existing production baselines are rule-based and miss the semantic cases. LangGraph's `recursion_limit` is a counter. AutoGen composes termination on message count, token budget, and pattern match. OpenHands' `StuckDetector` implements five heuristics. All catch the mechanical loops. None catch drift, where the agent is making progress in a direction that's no longer the right direction. A model-based detector catches both, which is why the LLM-companion pattern is the one worth pulling into production pipelines next.

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
