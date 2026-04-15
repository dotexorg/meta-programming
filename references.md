# References

This page collects sources, evidence levels, and experiment details for the [Meta-Programming](index.md) documentation.

## Evidence Levels

| Marker | Level | Meaning |
|--------|-------|---------|
| 🟢 | Proven | Our experiment, our data, measured result |
| 🟡 | Trusted source | Anthropic, Microsoft Research, peer-reviewed — we read the primary source |
| 🟠 | Community reports | Widely observed, not independently verified by us |
| 🔴 | Unverified | Heard, not checked |
| ⚪ | Opinion | Our synthesis, reasoned but not proven |

---

## Our Experiments

| # | Description | Key Finding | Pages |
|---|-------------|-------------|-------|
| 1 | EventBus refactor A/B (5 variants, 509 TS files, DDD, Cloudflare Workers) | Process beats information: Scout→Spec→Worker→Review pipeline ($8.45) outperformed raw context injection ($2.84 fail, $14.30 pass). Code maps hurt: $9.99 fail vs $6.63 pass without map. | [pipeline](pipeline.md), [principles](principles.md) |
| 2 | Telegram bot without spec (support reply feature) | Two consecutive deterministic failures without spec. First corrupted existing handler, second missed intent entirely. | [specification](specification.md), [principles](principles.md) |
| 3 | 24-file type refactor | 253 tests, zero regressions, $5.50 API cost. Review required three iterations (both failure types were missing pre-flight checks). 106-turn task didn't contaminate subsequent tasks due to context reset. | [index](index.md), [verification](verification.md), [pipeline](pipeline.md) |
| 4 | Spec review with barrel re-export | Agent proposed barrel re-export during spec review, recognized it was unnecessary when asked to explain. Spec review is cheaper than code review. | [specification](specification.md), [principles](principles.md) |
| 5 | Pipeline variant comparison | Structured pipeline: correct on first attempt, $8.45. Raw prompt: wrong, $2.84. Sequential with context: correct, $14.30. With code map: wrong, $9.99. | [pipeline](pipeline.md) |
| 6 | KB A/B test (generic vs structured) | Generic Sonnet said "give it a code map." KB-loaded agent flagged exploration-vs-exploitation paradox with prior session evidence before writing code. | [index](index.md), [principles](principles.md) |
| 7 | Edit tool investigation | Persistent error pattern traced to our own extension, not the platform. Post-fix benchmark: 7.1% errors, 1.1% data loss. | [index](index.md) |
| 8 | Model evaluation (thinking levels) | Thinking level acts as compliance-to-conviction dial. Soft sycophancy identified: agent says no while providing implementation. | [index](index.md) |
| 9 | Opus degradation incident (April 2026) | Read:Edit ratio dropped from 6.6 to 2.0, thinking depth fell 67%, costs spiked 80×. Three-day silent degradation with no API-side signal. | [verification](verification.md), [principles](principles.md) |

---

## External Sources

### Large-Scale Studies

1. **LinearB (2026).** "The Real Impact of AI on Developer Productivity." 8.1M pull requests, 4,800 teams, 42 countries. AI-generated code: 1.7× more review revisions, 4.6× longer review wait, 32.7% acceptance rate (vs 84.4% human). Developers feel 20% faster; tasks take 19% longer end-to-end. 🟡
   - Referenced in: [index](index.md), [verification](verification.md), [principles](principles.md)

2. **Stanford Meta-Harness Study (2026).** Changing the harness around a fixed LLM produces a 6× performance gap on the same benchmark. Automated harness optimization searched configurations the way gradient descent searches weight space. 🟡
   - Referenced in: [index](index.md), [principles](principles.md)

3. **ETH Zurich (Feb 2026).** 138 real-world tasks across 3 models. Auto-generated context files reduced task success by 3%. Human-written boundaries improved success by 4%. 🟡
   - Referenced in: [specification](specification.md), [pipeline](pipeline.md), [principles](principles.md)

4. **Sonar (2026).** Survey of 1,000+ developers. Only 48% verify AI output before shipping. 🟡
   - Referenced in: [index](index.md)

5. **Microsoft Copilot Study (2026).** 10-month study, 878 pull requests. "The bottleneck moved from typing speed to knowledge, judgment, and ability to articulate tasks." 🟡
   - Referenced in: [index](index.md)

6. **BSWEN (2026).** 133 cycles, 42 development phases, four models in strict isolation. GPT caught Python security issues Claude missed. Claude caught architectural violations GPT normalized. Each model had different blind spots. 🟡
   - Referenced in: [verification](verification.md), [principles](principles.md)

7. **Bamberg/Heidelberg (2026).** Systematic analysis of 2,926 repositories across Claude Code, GitHub Copilot, Cursor, Gemini CLI, Pydantic AI. Converging on identical patterns independently. 🟡
   - Referenced in: [index](index.md)

### Research Papers

8. **Tsinghua NLAH (March 2026).** "Natural-Language Agent Harnesses." Harness behavior externalized as "a portable executable artifact in editable natural language." 🟡
   - Referenced in: [index](index.md)

9. **Microsoft Research RiSE (March 2026).** Lahiri et al. "Intent Formalization" named as a grand challenge for 2026. Intent gap: the semantic distance between what a developer means and what the system does. 🟡
   - Referenced in: [index](index.md), [specification](specification.md)

10. **Allard et al. ERL (March 2026).** Agents with heuristics extracted from prior trajectories outperformed ReAct baselines by +7.8%. "Heuristics provide more transferable abstractions than few-shot prompting." 🟡
    - Referenced in: [index](index.md)

11. **ExpeL (Andrew Zhao et al., 2023).** Experience Learning: three-stage self-improvement (act → reflect → extract). On HotpotQA and ALFWorld, ExpeL agents improve with each batch of trajectories. 🟡
    - Referenced in: [self-improvement](self-improvement.md), [landscape](landscape.md)

12. **Chroma MECW (2025).** Maximum Effective Context Window. 18 frontier models tested. Universal degradation; effective window as low as 1-2% of advertised maximum in some conditions. 🟡🔴 (specific numbers unverified by us)
    - Referenced in: [context-engineering](context-engineering.md), [principles](principles.md)

13. **Reflexion (NeurIPS 2023).** Verbal reflection with episodic memory raised HumanEval from 80% to 91%. 🟡
    - Referenced in: [landscape](landscape.md)

14. **ADAS (ICLR 2025).** Automated Design of Agentic Systems. The agent designs its own pipeline structure. 🟡
    - Referenced in: [landscape](landscape.md)

15. **Gödel Agent (ICLR 2025).** Recursive self-modification via confidence-based logic. 🟡
    - Referenced in: [landscape](landscape.md)

16. **DKB (January 2026).** Deterministic Knowledge Bases. AST graphs beat vector RAG and LLM-generated knowledge graphs for code navigation. 🟡
    - Referenced in: [landscape](landscape.md)

17. **AMBIG-SWE (ICLR 2026).** Benchmark for ambiguity detection in software engineering tasks. 🟡
    - Referenced in: [specification](specification.md)

18. **ACE Framework.** Agentic Context Engineering. Memory scoring: each unit carries a score that updates on use. Quality saturates at ~7 governed memories per entity across 500 adversarial queries. 🟡
    - Referenced in: [self-improvement](self-improvement.md)

19. **Pavlyshyn (Jan 2026).** History of constrained natural language in programming: COBOL, SQL, Simula. 60-year progression. 🟡
    - Referenced in: [specification](specification.md)

### Vendor Documentation & Posts

20. **Anthropic — "Building Agents with Skills."** Skills as zero-cost-until-invoked context units. Self-evaluation bias documented explicitly. Claude Code architecture: tiered context loading, compaction, coordinator mode. 🟡
    - Referenced in: [index](index.md), [context-engineering](context-engineering.md), [verification](verification.md), [principles](principles.md)

21. **DSPy (Stanford, 33K stars).** Prompts as learnable parameters. BootstrapFewShot, MIPROv2 search language space automatically. 🟡
    - Referenced in: [index](index.md), [landscape](landscape.md)

22. **Pydantic (2026).** Analyzed 4,668 pull request comments, extracted 150 AGENTS.md rules. Engineering taste compiled into agent instructions. 🟡
    - Referenced in: [specification](specification.md)

23. **Promptfoo (acquired by OpenAI, March 2026, $86M).** 350K developers. Trajectory assertions: `tool-used`, `tool-args-match`, `tool-sequence`, `step-count`, `goal-success`. 🟡
    - Referenced in: [verification](verification.md), [landscape](landscape.md)

24. **spec-kit (79K stars).** GitHub's 5-command SDD workflow: constitution → specify → plan → tasks → implement. 20+ agents. 🟡
    - Referenced in: [specification](specification.md), [landscape](landscape.md)

25. **Kiro (Amazon).** IDE built around requirements → design → tasks. Specs live in project root, evolve with codebase. 🟡
    - Referenced in: [specification](specification.md), [landscape](landscape.md)

26. **AGENTS.md.** 60,000+ repositories. Linux Foundation / Agentic AI Foundation standard. Cross-tool coordination protocol with Anthropic, Google, Microsoft, Cursor backing. 🟡
    - Referenced in: [index](index.md), [specification](specification.md), [landscape](landscape.md)

27. **Augment Code.** Single-writer rule for hotspot files. Sequential merge strategy. 🟡
    - Referenced in: [pipeline](pipeline.md)

28. **OpenTelemetry GenAI SIG.** Semantic conventions: `gen_ai.chat` for LLM calls, `agent.invoke` for agent steps, `tool.execute` for tool calls. Datadog, Honeycomb, New Relic support natively. 🟡
    - Referenced in: [verification](verification.md)

29. **Simon Willison (@simonw).** Rigorous public reference on agentic engineering. Tests are free and mandatory. Agents follow existing code patterns. 🟡🟠
    - Referenced in: [context-engineering](context-engineering.md), [landscape](landscape.md)

30. **Martin Fowler.** Spec progression: spec-first → spec-anchored → spec-as-source. Maturity curve mapping. 🟡
    - Referenced in: [index](index.md)

### Community Reports

31. **Amazon deployment (2026).** 21,000 agents, 80% weekly usage. 4 Sev-1 incidents in 90 days. 6-hour outage, ~6.3M lost orders. ~30,000 layoffs concurrent with AI scaling. 🟠
    - Referenced in: [index](index.md), [verification](verification.md), [principles](principles.md), [landscape](landscape.md)

32. **Self-improvement tools (March 2026).** Three independent projects (skill-loop, selfwrite, iterate) shipped in the same week without coordination. All focused on instruction improvement, not weight modification. 🟠
    - Referenced in: [self-improvement](self-improvement.md), [landscape](landscape.md)

33. **Edit tool failure rate.** Agents express edits as text replacements, which break on whitespace drift, formatting changes, and multi-cursor ambiguity. Documented across multiple tools. 🟠
    - Referenced in: [landscape](landscape.md)

34. **Spec sizing (900-1600 tokens).** Community-reported sweet spot for structured quick-dev specs. Below 900: ambiguity risk. Above 1600: tail instruction degradation. 🟠
    - Referenced in: [specification](specification.md)

35. **Rory Teehan.** Structured error logging: what happened, why, what should have happened. 🟡
    - Referenced in: [self-improvement](self-improvement.md)

### People to Follow

36. **Andrej Karpathy** (@karpathy). Autoresearch: 700 commits in two days, −11% validation loss. Memory should be tree-structured, not flat. 🟠
37. **Mario Zechner** (@badlogicgames). Built Pi. When agents self-praise, human review becomes the bottleneck. 🟠
38. **Harrison Chase** (@hwchase17). LangChain. What does production agent orchestration actually look like at scale. 🟠

---

*This reference list is maintained alongside the documentation. Sources marked with evidence levels as used in the main text.*
