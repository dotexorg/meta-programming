# Linguistic Meta-Programming

<img alt="image" src="assets/index_hero.avif" />

How coding agents improve through language — specs, reviews, lessons, rules — without touching model weights.

## What's here

10 articles (~25,500 words) generated from a research knowledge base of 1,033 atomic findings across 171 sessions.

| Page | About |
|------|-------|
| [index](index.md) | Thesis, three layers, evolution from vibe coding to meta-programming |
| [specification](specification.md) | DO/DON'T/GLOSSARY, SDD, Intent Formalization, AGENTS.md |
| [context-engineering](context-engineering.md) | Token budgets, progressive disclosure, caching |
| [pipeline](pipeline.md) | Scout → Spec → Plan → Workers → Review → Lessons |
| [verification](verification.md) | Separate reviewer, multi-model, OTel, Amazon case study |
| [self-improvement](self-improvement.md) | Memory hierarchy, lesson extraction, closed loop |
| [principles](principles.md) | 6 principles, 3 maturity levels, anti-patterns |
| [playbook](playbook.md) | 15 rules — ready-to-apply, tied to principles and experiments |
| [landscape](landscape.md) | Tools, trends, papers — April 2026 snapshot |
| [references](references.md) | Full bibliography — papers, repos, people, our experiments |
| [changelog](changelog.md) | What's new since previous sync for return readers |

## Evidence levels

- 🟢 Our experiment, our data
- 🟡 Trusted source (Anthropic, Microsoft Research, peer-reviewed)
- 🟠 Community reports (practitioners, GitHub projects)
- 🔴 Unverified
- ⚪ Our opinion / synthesis

## How it was built

Articles are generated from a private KB (`meta-programming-kb/`, 1,033 bullets, 2,411 lines across 12 tier files) using a doc-editor pipeline. The KB is internal research memory: raw findings, working specs, engineering backlogs, process notes, source mappings. These articles are the reader-facing product extracted from that memory.

Three-layer memory architecture:
- **tier1-core** (12KB) — always-loaded expert memory: thesis, proven findings, principles, active questions
- **tier2-*** (4–18KB each) — topic deep-dives: spec-driven, verification, self-improvement, memory systems, orchestration, security, experiments, context-engineering, evaluation
- **tier3-*** — appendices: landscape snapshot, tools catalog

Research tools: [Pi coding agent](https://github.com/nicobailon/pi-subagents), Exa API, content pipeline (Twitter/GitHub), Gemini search.

## Related

- [Claude Code from Source](https://claude-code-from-source.com) — 18-chapter technical book on CC v2.1.88 internals
- [AGENTS.md](https://github.com/anthropics/agents-md) — Linux Foundation standard for agent instructions (60K+ repos)
- [spec-kit](https://github.com/github/spec-kit) — GitHub's SDD framework (79K★)
- [DSPy](https://github.com/stanfordnlp/dspy) — Stanford's prompt optimization framework (33K★)
- [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/) — Simon Willison's guide
