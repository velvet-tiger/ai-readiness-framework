---
name: "AI Readiness Audit"
description: "Audit a project against the AI Readiness Framework and produce a prioritised gap report by tier. Use when assessing a codebase for agent readiness, onboarding a project for AI agent work, checking which tier a project currently satisfies, or determining what must be fixed before running agents safely."
---

# AI Readiness Audit

## What This Skill Does

Systematically evaluates a project against the AI Readiness Framework checklist and produces a structured gap report showing:

- Whether the project displays any **fundamentally hostile patterns** that block tier work entirely
- Which tier the project currently satisfies (None / Tier 0 / Tier 1 / Tier 2 / Tier 3)
- Which specific requirements are missing or incomplete
- A prioritised remediation plan ordered by impact and tier

The four capabilities being assessed are: **Orient** (can an agent understand the project), **Act** (can an agent produce consistent output), **Verify** (can an agent close its own feedback loop), **Escalate** (does an agent know when to stop).

---

## Quick Start

Run this skill at the root of any project:

1. Read the project root directory listing
2. Run the **Fundamentally Hostile Patterns** pre-check; if any are present, stop and report — tier assessment is meaningless until they are addressed
3. Work through the tier checklists below section by section, starting at Tier 0
4. Output a gap report using the format at the end of this skill

---

## Pre-check: Fundamentally Hostile Patterns

Some patterns make a codebase agent-hostile in a way that tier work cannot fix. If any of the following are present, stop the audit and report them — no amount of `AGENTS.md` writing or rule tuning will compensate.

- [ ] **Millions of files without clear organisation** — if `find . -type f | wc -l` (excluding generated content, lockfiles, and vendored dependencies) returns more than the team can navigate, the project needs structural triage first
- [ ] **Non-standard version control** — agents assume git semantics; projects on Perforce, ClearCase, or other legacy systems will see agent behaviour degrade in ways configuration cannot fix
- [ ] **Deeply interconnected cross-directory dependencies with no module boundary** — when any file can import from any other file, there is no boundary for an agent to respect or for a rule to enforce
- [ ] **Generated or binary assets mixed with source code** — when generated code, vendored libraries, and source live in the same directory tree without separation, agents will read generated code as canonical and reproduce its conventions
- [ ] **Hundreds of thousands of top-level folders** — a flat, high-cardinality root defeats every orientation strategy an agent can apply

If any item above is present, record it under a **Blocking Conditions** heading in the gap report and recommend structural triage (module extraction, generated-content separation, migration to standard version control) before proceeding to tier assessment.

---

## Audit Checklist

Work through every item. Mark each as **PASS**, **FAIL**, or **PARTIAL**. Tiers are cumulative — Tier 1 assumes Tier 0 is satisfied, and so on.

### Tier 0 — Supervised

The minimum needed for an agent to be pointed at the project without immediately producing harmful or useless output. A human reviews every change.

- [ ] **2.1a** `AGENTS.md` or `CLAUDE.md` exists and states what the project does, what stack it uses, and which areas are off-limits
- [ ] **2.3a** At least one runnable command exists that the agent can run to verify it has not broken the build or test suite, producing a clear pass or fail result
- [ ] **2.4a** Restricted directories or files are explicitly declared in `AGENTS.md`

---

### Tier 1 — Guided

#### Code Quality

**1.1 Consistency**
- [ ] A single dominant pattern exists for every common concern (model definition, error handling, API response shape, etc.)
- [ ] File and class organisation is consistent and predictable — things live where their name predicts
- [ ] No dual implementations — no two coexisting approaches to the same concern

**1.2 Explicitness**
- [ ] No metaprogramming, dynamic dispatch, or macro-heavy patterns without doc comments at the call site
- [ ] All dependencies declared explicitly — constructor injection or equivalent; no ambient resolution

**1.3 Naming**
- [ ] All classes, methods, and variables communicate intent without requiring call-chain tracing
- [ ] No overloaded names — a name means one thing across the entire codebase
- [ ] **1.3c** No duplicate file basenames or class/function simple names across unrelated paths (or each collision is documented as load-bearing — e.g. parallel adapters)
- [ ] **1.3d** No empty generic names (`Service`, `Helper`, `Manager`, `Handler`, `Utils`, `Data`) used unqualified in new code; `RULES.md` lists banned generics

**1.5 Dead Code**
- [ ] No unused classes, methods, or variables
- [ ] No commented-out code blocks

**1.7 Error Handling**
- [ ] A single consistent error handling approach applied throughout
- [ ] No silent failure paths — every error produces a typed error, log event, or visible exception

**1.12 File Length**
- [ ] **1.12a** `RULES.md` declares per-category source-file size budgets (e.g. 500 lines general, 300 components, 800 generated/fixtures)
- [ ] Source files comply with the declared budget — sample 5–10 of the largest files; if any exceed the budget without an inline justification, FAIL

#### Project Structure

**2.1 Orient**
- [ ] `ARCHITECTURE.md` exists at the root, covers all services and communication patterns, includes a diagram, and is current
- [ ] **2.1g** `ARCHITECTURE.md` names an architectural pattern from the shortlist (hexagonal/ports-and-adapters, clean, layered, vertical slice, MVC, or "project-specific" with rationale) and states the dependency-direction rule
- [ ] **2.1i** A `.claudeignore`, `.agentignore`, or equivalent exists at the root and excludes generated content, binary assets, lockfiles, and vendored dependencies; check by file existence and by sampling listed patterns

**2.2 Act**
- [ ] `AGENTS.md` or `CLAUDE.md` exists at the root, covers project purpose, stack, conventions, directory ownership, and what the project does not do
- [ ] `RULES.md` exists (or rules are inline in `AGENTS.md`) and is specific enough that two agents produce consistent output independently
- [ ] **2.2f** Each of `AGENTS.md`, `RULES.md`, `ARCHITECTURE.md` is under 200 lines, or the file is an index referencing modular sub-files; run `wc -l` on each and verify

**2.4 Escalate**
- [ ] Escape hatch conditions are defined in `AGENTS.md` with specific triggering conditions
- [ ] Change isolation conventions are documented (branch strategy, commit conventions, PR size limits)

---

### Tier 2 — Safe

#### Code Quality

**1.2 Explicitness**
- [ ] All functions and methods carry full type signatures including return types

**1.4 Tests**
- [ ] Tests cover public contracts and behaviour, not implementation details
- [ ] Default verification run completes in under two minutes
- [ ] Test fixtures reflect realistic edge-case data

**1.5 Dead Code**
- [ ] No permanently resolved feature flags

**1.6 Boundaries**
- [ ] Business logic does not leak across layers
- [ ] Cross-service communication goes through declared interfaces only
- [ ] **1.6c** The declared architectural pattern's dependency-direction rule is enforced by a tool or test (deptrac, dependency-cruiser, ArchUnit, import-linter, or equivalent) and runs in CI or pre-commit

**1.8 State**
- [ ] No mutable global or static state
- [ ] All side effects are localised and signalled by name

**1.9 Logging**
- [ ] Log levels have defined semantics documented in `RULES.md`
- [ ] All log statements emit consistent structured fields
- [ ] Log transport and destination documented in `AGENTS.md`

**1.10 Dependency Currency**
- [ ] All major dependencies are reasonably current
- [ ] Known version constraints documented with reasons

**1.11 Operational Context**
- [ ] All environment variables documented in `.env.example` or `docs/environment.md`
- [ ] All external service dependencies documented with auth approach, rate limits, quirks
- [ ] Observability infrastructure documented with guidance for new code

**1.12 File Length (T2)**
- [ ] **1.12b** A pre-commit or CI check fails any source file above the declared hard ceiling (default 1500 lines); generated content and lockfiles allowlisted via `.file-length-ignore` or CI config

#### Project Structure

**2.1 Orient**
- [ ] **2.1c** Documentation follows an established schema with a top-level index
- [ ] **2.1d** A machine-readable project manifest exists at the root
- [ ] **2.1e** Canonical data and API contracts are committed (OpenAPI spec, schema file, or shared type layer) and referenced from both the manifest and `AGENTS.md`; absence means agents must infer contracts from code, which produces drift
- [ ] **2.1h** Deep or specialised subdirectories carry their own `AGENTS.md` / `CLAUDE.md` files; sample 2-3 of the largest subtrees and check for a local context file where conventions differ from the root
- [ ] **2.1j** A `docs/codebase-map.md` (or equivalent one-page concept-to-directory guide) exists where the directory structure does not self-describe; not required where directories are self-evidently named

**2.2 Act**
- [ ] **2.2g** `AGENTS.md` lists every MCP server expected to be connected for normal agent work on this project, with the purpose of each and installation instructions

**2.3 Verify**
- [ ] A reproducible environment exists (`Dockerfile`, `.devcontainer/`, or `Makefile` with standard targets)
- [ ] Test surface covers critical contracts and a green result is genuine signal
- [ ] **2.3e** An LSP server for each primary language is documented and installable; check `AGENTS.md` or setup docs for an LSP install instruction or `make setup-lsp` style target

---

### Tier 3 — Autonomous

- [ ] **2.1f** `docs/` is treated as the system of record; documentation freshness is mechanically enforced via CI and a recurring cleanup task (e.g. a scheduled job or background agent that scans for stale content and opens correction PRs)
- [ ] **2.1k** A documented cadence (every 3-6 months) exists for reviewing and pruning agent configuration; check for a review checklist, a calendar entry referenced in `AGENTS.md`, or a `docs/agent-config-review.md`
- [ ] **2.2c** `AGENTS.md` is short (around 100 lines), acts as an index, and points into a structured `docs/` knowledge base rather than inlining all conventions; run `wc -l AGENTS.md` and inspect contents for evidence of indexing rather than encyclopaedic content
- [ ] **2.2d** Sub-agent library (`agents/` or `.claude/agents/`) with single-responsibility files, including at least one explorer / editor split
- [ ] **2.2e** Skill set covering at minimum the release workflow
- [ ] **2.2h** A committed `.claude/` or `.agents/` directory declares hooks, sub-agents, skills, and MCP configuration as code; not relying on per-developer local setup
- [ ] **2.2i** A Stop hook (or equivalent) proposes `AGENTS.md` / `RULES.md` updates from session reflections; check `.claude/hooks.json` or equivalent for a Stop-event hook that opens proposed updates rather than applying them silently
- [ ] **2.2j** (optional) Where multiple projects share configuration, the configuration is packaged as a plugin or installable bundle
- [ ] **2.3d** Architectural invariants from `RULES.md` are encoded as custom linters or structural tests in CI; lint error messages include remediation instructions (a pointer to the relevant rule or doc), not just a failure code
- [ ] **2.3f** Lint, format, and structural checks run as pre-edit, pre-commit, or pre-write hooks that block non-conforming writes; check `.git/hooks`, `.pre-commit-config.yaml`, or `.claude/hooks.json`
- [ ] **2.4d** Agent output review standard documented in `CONTRIBUTING.md`, distinct from the standard for human-authored changes
- [ ] **2.4e** A named person or team is documented in `AGENTS.md` as the DRI for agent configuration; check the AGENTS.md for an "Agent configuration owner" or similar entry

---

### Monorepo Additions (if applicable)

**Tier 1**
- [ ] Root manifest includes a monorepo map of all packages and services
- [ ] Ownership declared per package (`CODEOWNERS` or equivalent)
- [ ] Each package has its own `AGENTS.md`
- [ ] Each package has its own scoped coding rules
- [ ] Cross-boundary and shared library escalation rules defined

**Tier 2**
- [ ] Inter-service communication fully documented in `ARCHITECTURE.md`
- [ ] Build order and dependency graph declared
- [ ] Test and lint commands scoped per package

**Tier 3**
- [ ] Package-specific sub-agents inside the package directory
- [ ] Shared libraries declared with governance documented

---

## How to Assess Each Item

For each checklist item, do the following:

1. **Check for the file or artefact** — does the required file exist?
2. **Read it** — does it actually contain the required content, not just exist?
3. **Spot-check the code** — for code quality items, sample 3–5 representative files and look for violations

Do not mark an item PASS based on file existence alone. An empty `AGENTS.md` is a FAIL.

---

## Gap Report Format

Produce the gap report in this structure:

```
## AI Readiness Gap Report

**Current Tier:** [None / Tier 0 / Tier 1 / Tier 2 / Tier 3]
**Assessed:** [date]
**Monorepo:** [Yes / No]

---

### Blocking Conditions

[Only include this section if any Fundamentally Hostile Patterns were found. List each, with the recommended structural-triage action. If this section is present, tier assessment below is provisional — the blocking conditions must be addressed before tier work is meaningful.]

---

### Summary

[2–3 sentences describing overall state and the single most important thing to fix]

---

### Failing Items

#### Tier 0 Gaps (must fix first — without these the agent has no foothold)

| # | ID | Requirement | Finding |
|---|----|-------------|---------|
| 1 | [ID] | [requirement] | [what is missing or wrong] |

#### Tier 1 Gaps

| # | ID | Requirement | Finding |
|---|----|-------------|---------|

#### Tier 2 Gaps

| # | ID | Requirement | Finding |
|---|----|-------------|---------|

#### Tier 3 Gaps

| # | ID | Requirement | Finding |
|---|----|-------------|---------|

---

### Remediation Plan

Ordered by priority (lowest tier first, then by effort within tier):

1. **[ID — Requirement]** — [Specific action to take] — [Estimated effort: low/medium/high]
2. ...

---

### Passing Items

[Brief list of what is already in good shape, grouped by tier]
```

---

## Escalation Conditions

Stop and ask the user before proceeding if:

- The project has no `AGENTS.md`, `CLAUDE.md`, or equivalent — ask whether to create one as part of the audit
- The project appears to be a monorepo but no monorepo map exists — confirm before applying monorepo checklist items
- The codebase is very large — agree on which services or packages to sample before beginning
