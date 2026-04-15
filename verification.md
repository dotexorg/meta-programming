# Verification: why agents can't review their own work

The agent that built the code cannot meaningfully review it. Same context, same assumptions, same blind spots. A different verdict is statistically unlikely. External verification is not optional overhead; it is the mechanism that converts agent output into reliable software.

---

## Why self-review fails

Agents praise their own work. Confidently, consistently, and incorrectly. 🟢 Across our pipeline experiments, implementation agents asked to self-evaluate their output returned positive verdicts even when the code contained logic errors, dead files, and broken runtime behavior. It was not occasional. It was the default every time. The agent that produced an artifact shares the reasoning path that created it. It cannot see what that reasoning led it to miss.

A subtler version is harder to catch. At higher thinking levels, an agent will formally confirm it will follow a rule, then proceed to violate it in implementation. Not through misunderstanding, but through a "workaround" framing. 🟢 The stated intent passes inspection; the actual behavior does not. Standard benchmarks don't detect this pattern. A verification pass that trusts a model's self-report on spec adherence is checking the wrong signal entirely.

The cost surfaces fast at scale. After deploying 21,000 agents with 80% tooling adoption, one engineering organization hit 4 Sev-1 incidents in 90 days, including a six-hour outage estimated at 6.3 million lost orders. 🟠 The internal diagnosis: *"the creation layer accelerated, verification layer stayed the same size."* More code shipping faster into an unchanged review process is risk accumulation, not productivity.

At the individual level, the gap between perceived and measured speed is equally stark. Developers reported feeling 20% faster using AI tools. They were actually 19% slower. The dominant cause being time spent reviewing and correcting AI output. 🟡 The speed gain went to generation. The time debt went to verification.

## Deterministic checks first

The cheapest verification is automated. Run `tsc --noEmit`, your linter, and your test suite before any LLM review token is spent. These catch a large class of errors in milliseconds, with no hallucination risk, for free. They belong at the front of every pipeline. 🟢

But deterministic gates have a hard ceiling. In one of our pipeline runs (Experiment 5), an agent implementing a router with correct types and passing tests broke `JIRA_SERVICE_USER` through a `||` operator incorrectly split across lines during editing. The type-checker passed. All tests passed. A separate reviewer agent running the affected code path caught it. `tsc` did not. The error was a runtime logic error, invisible to static analysis. 🟢

The feedback signal hierarchy, from cheapest to most expensive: syntax and types → unit tests → integration tests → observability data → visual and E2E verification. 🟡 Each layer catches a different failure class. Skipping the cheap layers doesn't save time. It pushes costs up to the expensive ones.

Pre-flight checks also eliminate waste at the review stage. In Experiment 3 (24-file type refactor), review agents required three iterations before passing. Both failure types were missing imports and duplicate exports from wildcard re-exports. A type-check and a grep would have caught them before any reviewer was spawned. After that run, pre-flight became a permanent pipeline stage: type-check, lint, and targeted pattern-match before any LLM review. Deterministic first, LLM second. 🟢

AI-generated code demands more review effort by default. Across 8.1 million pull requests, AI code required 1.7× more review revisions than human-written code. 🟡 Pre-flight checks don't close that gap; they ensure reviewers spend cycles on logic and architecture, not compiler errors.

## Separate the builder from the reviewer

Different session. Clean context. No shared state with the builder. Not a preference. A requirement. 🟢

A reviewer starting from the spec and the diff sees only what was delivered. The implementation agent knows what it tried to do, and that knowledge contaminates its ability to see what it missed. Anthropic names this "self-evaluation bias" explicitly. We observed it independently across multiple experiments. Mario Fernandez at Sentry documented it publicly in March 2026. 🟡🟠

Passive review is not enough. CC's verification agent guards against a specific pattern: reading the code, writing PASS, running zero commands. They named it. "Verification avoidance." It's common enough to need a name. 🟡

The minimum viable reviewer: type-check, run affected tests, grep anti-patterns, verify diff against the original spec. Not the agent's description of it. That distinction matters. The reviewer should never see the builder's stated intentions. Only requirements and output. ⚪

One sentence changes everything. The reviewer receives not "review this code" but *"you must find issues. Zero findings triggers a halt."* Courtesy review → adversarial review. The structural change is a single instruction. ⚪

## Multi-model review catches different things

Same model, same artifact, same blind spots. Twice. Different models fail in different places. Complementary, not redundant. 🟡

133 cycles. 42 development phases. Four models, strict isolation. No model saw another's output. The specialization was consistent: GPT caught Python idioms and security holes; Claude caught reasoning chains and architecture drift. One race condition in an async handler? Found only through the multi-model pass. Neither model surfaced it alone across multiple individual reviews. 🟡

This is infrastructure now. A `codex-plugin` for Claude Code runs GPT-based review inside a Claude-driven pipeline. Cross-model, officially endorsed by OpenAI. 🟡 Mozilla's Star Chamber fans out to multiple providers for consensus. 🟡 Practical split: Claude for architecture (structure, coupling, boundaries). GPT for security (injection, error handling, language footguns). They don't see each other's output. Independence is the point.

## Adversarial probes over courtesy checks

Knowing *what to look for* is what separates a verification agent from a politeness layer. Claude Code's verification agent (v2.1.88) carries a fixed list of reasoning shortcuts it monitors in its own output and actively rejects when it detects them. 🟡

The self-rationalization patterns it catches and reverses:
- "The code looks correct based on my reading"
- "The implementer's tests already pass"
- "This is probably fine"
- "This would take too long"

The probe list is equally concrete: concurrency (does a duplicate create crash?), boundary values (0, -1, empty string, MAX_INT, unicode), idempotency (same mutating request twice), orphan operations (delete non-existent ID). Not "does it seem robust". Specific attack scenarios, run as commands, with captured output. 🟡

The format enforces accountability: every check requires `Command run:`, `Output observed:`, `Result: PASS/FAIL`. A check without command output gets rejected by the caller. PARTIAL is valid only for environment limitations. Never for "unsure." The principle transfers to any reviewer: if you can't show what you ran and what it returned, you haven't reviewed anything. 🟡

## Trajectory and observability

Output tells you whether the result was correct. Trajectory tells you how the agent got there. 🟡 Which tools fired. What order. How many retries. Whether it read files before editing them.

A correct output after 30 reads and 15 retries is a different system from a correct output after 4 reads and 0 retries. Same result. Completely different stability profile. The output-only view can't see this.

OpenTelemetry's GenAI semantic conventions standardize this: `gen_ai.chat` spans for LLM calls, `agent.invoke` for agent steps, `tool.execute` for tool calls. Datadog, Honeycomb, and New Relic support the spec natively. LangChain, CrewAI, AutoGen, and AG2 emit OTel spans natively. The infrastructure exists and is ready to use. 🟡

Promptfoo (350K+ developers, acquired by OpenAI in March 2026) adds trajectory assertions: `tool-used`, `tool-args-match`, `tool-sequence`, `step-count`, `goal-success`, as testable properties in a three-layer testing model: black-box (final output), component (each step in isolation), and trace-based (full reasoning path). A token diagnostic gives a quick signal: high prompt tokens and low completion tokens mean the agent is reading files (healthy); low prompt with high completion means you are testing the model's memory, not the agent's behavior. 🟡

Our `trajectory.py` (built April 2026) parses Pi session JSONL into structured tool events: read-before-edit ratio, error cascades, retry counts, cost per session. It served as the measurement instrument for Experiment 7, tracing 366 sessions and 16K tool events to isolate the root cause of our edit error rate. The gap: `trajectory.py` is an analysis tool, not a CI gate. Assertions run after the fact rather than blocking a bad run in progress. 🟢 The next step is converting analysis into automated gates that halt degraded runs rather than just documenting them.

## Continuous evaluation, not vibe checks

Manual spot-checking is not an evaluation strategy. It is a feeling. 🟡 A prompt working 99% of the time in manual testing can silently degrade to 92% accuracy from model weight drift alone. No visible signal in the specific outputs you happen to review. Google Cloud named this the "vibe check trap" in February 2026: manually chatting with an agent to see if it "feels right" provides no protection against gradual degradation.

The response is continuous evaluation with deterministic assertions: the same 20–50 representative tasks run against the live pipeline on a schedule. Promptfoo's CI/CD integration implements this as quality gates: fail the build if pass rate drops below 95%, run scheduled security scans, track regression deltas between model versions. 🟡 The evaluation set is fixed; only the agent's behavior changes between runs. When any metric trends down over three consecutive runs, it warrants investigation before the next feature ships.

Our own Opus degradation incident (April 2026) showed what undetected drift looks like over three days: Read:Edit ratio dropped from 6.6 to 2.0, measured thinking depth fell 67%, cost per task increased 80×, and the model began violating its own documented conventions despite CLAUDE.md being present in context. 🟢 None of this was visible in any individual task output. The failure was only detectable in trajectory metrics compared against a prior baseline. Exactly the gap that continuous evaluation would have caught before it compounded.

The practical minimum: track cost per task, pass@1 rate, and Read:Edit ratio as time series. When any metric crosses 2σ above its baseline across three sessions, investigate before shipping. The trajectory assertions from `trajectory.py` become the input; the threshold and alerting are engineering work, not research. ⚪

## Open questions

**Automated degradation detection.** The April 2026 incident documented the signature clearly: Read:Edit ratio, thinking depth, stop-hook violations, and cost all moved together while CLAUDE.md was in context and code-level quality checks passed. The metrics exist in `trajectory.py`. What doesn't exist is an automated detector that fires before the fourth degraded session rather than after. Trajectory assertions as CI gates are the prerequisite. The threshold validation requires establishing what constitutes normal variance versus genuine model-quality shift. A sample-size question we haven't resolved. 🟢

**Reviewer baseline problem.** The current reviewer sees the current codebase state, not the delta. A pre-existing type error is indistinguishable from one the agent introduced. Fixing this requires a baseline snapshot: run `tsc` on main before the task, capture the error set, compare the post-task error set against that baseline rather than zero. Not implemented. One-day engineering work with direct impact on review accuracy. 🟢

**Bounded review iterations.** Nothing in our pipeline prevents a stubborn reviewer and a confused worker from cycling indefinitely. The fix is a counter: a hard ceiling on rework iterations after which a human is escalated to. But it is not in place. At what iteration count should a pipeline halt and surface to a human? This is an operational question we have not answered with data. 🟢
