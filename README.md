# AI Readiness Framework

by Christopher Skene | [https://velvettiger.com.au](https://velvettiger.com.au)

[![Install in Automatic](https://tryautomatic.app/badges/install.svg)](https://tryautomatic.app/install?repo=velvet-tiger/ai-readiness-framework)

A specification and skill set for making codebases safe for AI agent work.

---

## What this is

The AI Readiness Framework defines the minimum practices, files, and code quality standards a project needs before AI coding agents can operate on it reliably. It gives agents the ability to orient themselves, produce consistent output, verify their own work, and know when to stop and ask.

The framework covers code quality (consistency, explicitness, naming, error handling, boundaries, state, logging, dependencies, observability), project structure (orientation files, machine-readable manifest, verification commands, escalation rules), and the three dimensions that materially affect agent attention: **file length** (source and context files kept within readable size), a **named architectural pattern** (declared in `ARCHITECTURE.md` and enforced by tooling), and **naming hygiene** (no collisions across paths, no unqualified generic names).

It also covers **harness configuration** — the ignore files, hierarchical context files, language servers, MCP servers, committed `.claude/` and `.agents` directories, and hooks that determine how an agent meets the codebase at runtime. The framework names hostile patterns (millions of files, non-standard VCS, mixed generated content) that require structural triage before any tier work is viable.

The framework is structured as four tiers:

| Tier | Name | What it enables |
|---|---|---|
| 0 | Supervised | Agent can be pointed at the project without producing harmful output; human reviews every change |
| 1 | Guided | Agent understands conventions and makes consistent changes within a scoped area |
| 2 | Safe | Agent completes normal engineering tasks, runs its own checks, and escalates correctly — the target for most teams |
| 3 | Autonomous | Agent works across broader areas with less guidance; mechanical enforcement replaces documentation-only rules |

**[Read the full framework →](framework.md)**

---

## What's in this repository

```
framework.md      The specification — requirements, how to satisfy them, and examples
RULES.md          Coding rules for agents working on this repository itself
skills/           Agent skills for implementing the framework in other projects
skill.json        Package manifest for skill distribution
```

---

## Skills

Eleven skills are included for agents to use in other projects. Install them and invoke them by name.

### Assessment

| Skill | What it does |
|---|---|
| `ai-readiness-audit` | Audits a project against the full tier checklist and produces a prioritised gap report |
| `ai-readiness-recommendations` | Turns a gap report into a consultant-style remediation report — executive summary, current-state diagnosis, themed strategic recommendations, phased roadmap, risks, and item-by-item appendix |
| `ai-readiness-scaffold` | Generates `AGENTS.md`, `RULES.md`, and `ARCHITECTURE.md` from codebase discovery |

### Document authoring

| Skill | What it does |
|---|---|
| `agents-md` | Creates or updates `AGENTS.md` with all required sections |
| `rules-md` | Creates or updates `RULES.md` with stack-specific, non-generic rules |
| `architecture-md` | Creates or updates `ARCHITECTURE.md` including a Mermaid diagram |

### Code quality remediation

| Skill | What it does |
|---|---|
| `dead-code-cleanup` | Removes commented-out code, resolved feature flags, and unused symbols |
| `error-handling-audit` | Finds and fixes silent failures, empty catches, and inconsistent error patterns |
| `boundary-audit` | Detects and remediates layer violations and cross-service boundary breaches |

### Ongoing compliance

| Skill | What it does |
|---|---|
| `escalation-check` | Checks a change against escape hatch conditions before commit or PR |
| `convention-check` | Verifies new code follows `RULES.md` conventions; produces a deviation report |

---

## How to use

### Starting a new project

Run `ai-readiness-scaffold` to generate the required files from what can be discovered in the codebase. Review the output, fill in the marked placeholders, and check against the Tier 0 checklist in `framework.md`.

### Assessing an existing project

Run `ai-readiness-audit` to get a gap report showing which tier the project currently satisfies and what is missing. Then run `ai-readiness-recommendations` against the gap report to produce a consultant-style remediation report — executive summary, themed recommendations, phased roadmap, and risks — suitable for sharing with engineering leads or steering groups. Work through the roadmap phase by phase, re-auditing at each milestone.

### Reaching a specific tier

Each tier in `framework.md` lists its requirements with identifiers (e.g. `2.2a`, `1.7b`). The checklist in the appendix groups them by tier. Use the document authoring and remediation skills to address individual gaps.

### Keeping a project compliant

Add `escalation-check` and `convention-check` to your agent workflow as pre-commit gates. They read the project's own `AGENTS.md` and `RULES.md`, so they require no configuration beyond those files existing.

### Installing skills

Skills follow the `skill.json` standard. Install with any compatible skill package manager, or copy the relevant `skills/` directory into your project's `.claude/skills/` directory.

---

## Recommended order of adoption

1. **Tier 0** — Create a minimal `AGENTS.md`, declare restricted areas, confirm a verification command exists. Takes an hour for most projects.
2. **Tier 1** — Address code quality gaps (consistency, naming, dead code, error handling) and write full `AGENTS.md`, `RULES.md`, and `ARCHITECTURE.md`. Expect days to weeks depending on codebase condition.
3. **Tier 2** — Add type coverage, a fast reliable test suite, clean layer boundaries, and operational documentation. This is the target for most teams.
4. **Tier 3** — Encode architectural rules into CI, restructure `AGENTS.md` as an index, build a `docs/` knowledge base with mechanical freshness checks. Only pursue this after validating agent workflows at Tier 2.

Do not skip tiers. The requirements at each level are prerequisites for the ones above.

---

## License

Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You may share, adapt, and use this framework and its skills — including commercially — provided you give appropriate credit. See [LICENSE](LICENSE) for the full terms.


--- 

## Copyright

Copyright 2026 Christopher Skene | [https://velvettiger.com.au](https://velvettiger.com.au)
