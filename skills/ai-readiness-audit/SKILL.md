---
name: "AI Readiness Audit"
description: "Audit a project against the AI Readiness Framework and produce a prioritised gap report by tier. Use when assessing a codebase for agent readiness, onboarding a project for AI agent work, checking which tier a project currently satisfies, or determining what must be fixed before running agents safely."
---

# AI Readiness Audit

## What This Skill Does

Systematically evaluates a project against the AI Readiness Framework checklist and produces a structured gap report showing:

- Which tier the project currently satisfies (Tier 1 / Tier 2 / Tier 3 / None)
- Which specific requirements are missing or incomplete
- A prioritised remediation plan ordered by impact and tier

The four capabilities being assessed are: **Orient** (can an agent understand the project), **Act** (can an agent produce consistent output), **Verify** (can an agent close its own feedback loop), **Escalate** (does an agent know when to stop).

---

## Quick Start

Run this skill at the root of any project:

1. Read the project root directory listing
2. Work through the checklist below section by section
3. Output a gap report using the format at the end of this skill

---

## Audit Checklist

Work through every item. Mark each as **PASS**, **FAIL**, or **PARTIAL**.

### Tier 1 — Guided Baseline

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

**2.2 Act**
- [ ] `AGENTS.md` or `CLAUDE.md` exists at the root, covers project purpose, stack, conventions, directory ownership, and what the project does not do
- [ ] `RULES.md` exists (or rules are inline in `AGENTS.md`) and is specific enough that two agents produce consistent output independently
- [ ] **2.2f** Each of `AGENTS.md`, `RULES.md`, `ARCHITECTURE.md` is under 200 lines, or the file is an index referencing modular sub-files; run `wc -l` on each and verify

**2.4 Escalate**
- [ ] Escape hatch conditions are defined in `AGENTS.md` with specific triggering conditions
- [ ] Change isolation conventions are documented (branch strategy, commit conventions, PR size limits)

---

### Tier 2 — Safe Baseline

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
- [ ] Documentation follows an established schema with a top-level index
- [ ] A machine-readable project manifest exists at the root

**2.3 Verify**
- [ ] A reproducible environment exists (`Dockerfile`, `.devcontainer/`, or `Makefile` with standard targets)
- [ ] Test surface covers critical contracts and a green result is genuine signal

---

### Tier 3 — Higher Autonomy

- [ ] `adr/` directory with at least one ADR per major structural decision
- [ ] Sub-agent library (`agents/` or `.claude/agents/`) with single-responsibility files
- [ ] Skill set covering at minimum the release workflow
- [ ] Agent output review standard documented in `CONTRIBUTING.md`

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

**Current Tier:** [None / Tier 1 / Tier 2 / Tier 3]
**Assessed:** [date]
**Monorepo:** [Yes / No]

---

### Summary

[2–3 sentences describing overall state and the single most important thing to fix]

---

### Failing Items

#### Tier 1 Gaps (must fix first)

| # | Area | Requirement | Finding |
|---|------|-------------|---------|
| 1 | [area] | [requirement] | [what is missing or wrong] |

#### Tier 2 Gaps

| # | Area | Requirement | Finding |
|---|------|-------------|---------|

#### Tier 3 Gaps

| # | Area | Requirement | Finding |
|---|------|-------------|---------|

---

### Remediation Plan

Ordered by priority (Tier 1 first, then by effort within tier):

1. **[Requirement]** — [Specific action to take] — [Estimated effort: low/medium/high]
2. ...

---

### Passing Items

[Brief list of what is already in good shape]
```

---

## Escalation Conditions

Stop and ask the user before proceeding if:

- The project has no `AGENTS.md`, `CLAUDE.md`, or equivalent — ask whether to create one as part of the audit
- The project appears to be a monorepo but no monorepo map exists — confirm before applying monorepo checklist items
- The codebase is very large — agree on which services or packages to sample before beginning
