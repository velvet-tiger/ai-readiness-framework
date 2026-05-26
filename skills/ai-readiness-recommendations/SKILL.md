---
name: "AI Readiness Recommendations"
description: "Turn an ai-readiness-audit gap report into a consultant-style remediation report with an executive summary, current-state diagnosis, strategic recommendations grouped by theme, a phased implementation roadmap, risks, and an appendix of item-by-item findings. Use after running ai-readiness-audit, when preparing a remediation report for a stakeholder, when planning work to reach a target tier, or when prioritising across a long backlog of gaps."
---

# AI Readiness Recommendations

## What This Skill Does

Reads an existing AI Readiness gap report (produced by the `ai-readiness-audit` skill) and produces a **consultant-style remediation report** suitable for delivery to a project owner, engineering lead, or steering group.

The report is structured the way a consultancy would deliver findings:

- An executive summary that can be read in two minutes
- A current-state diagnosis that names the underlying problems, not just the symptoms
- Strategic recommendations grouped by theme — orientation, code quality, verification, escalation — rather than by rule ID
- A phased implementation roadmap with named milestones and sequencing rationale
- A risks-and-considerations section calling out what could go wrong
- A detailed appendix that maps every recommendation back to the framework rule, the skill that can execute it, and its prerequisite dependencies

This skill does **not** re-audit. It assumes the gap report is current and works from its contents. Run `ai-readiness-audit` first; bring its output here.

The voice of the output is authoritative and recommendation-led. The four capabilities being assessed (Orient, Act, Verify, Escalate) are preserved from the framework so the report remains anchored to a defined model.

---

## Quick Start

1. Confirm a gap report is available — either pasted into the conversation, output by a recent `ai-readiness-audit` run, or stored in the project
2. Confirm the **target tier** the user is working toward (T0 / T1 / T2 / T3); the report stops recommending work above the target
3. Identify the **audience** for the report (engineering lead, project owner, steering group) — the executive summary and risks section should be calibrated to what that audience needs to decide
4. Work through the five phases below
5. Output the consultant-style report in the structure at the end of this skill

---

## Inputs

- **Gap report** — the Markdown output of `ai-readiness-audit`, with Failing Items grouped by tier and each item carrying an ID, area, requirement, and finding
- **Target tier** — T0 / T1 / T2 / T3; if not stated, default to **T2** (the framework's "fit for purpose" tier)
- **Optional context**: team size, time budget, sensitive areas (e.g. an upcoming release that constrains refactor scope)

If no gap report is available, stop and recommend running `ai-readiness-audit` first. Do not produce recommendations from a fresh audit done inside this skill — the two responsibilities are deliberately separate.

---

## Phase 1: Parse the Gap Report

Extract every failing item. For each one, record:

- The framework ID (e.g. `2.1g`, `1.7b`, `3.2a`)
- The tier the item belongs to
- The original finding (what the audit observed)
- Whether the item is marked FAIL or PARTIAL — PARTIAL items often need narrower work than FAILs

Drop items that the audit marked PASS. They are not part of the remediation surface.

Drop items above the target tier. They are out of scope for this plan.

---

## Phase 2: Effort and Sequencing

For each item, assign an effort band based on the rubric below. Effort is a rough budget for an engineer or agent working alone; team work can compress it.

| Band | Typical scope | Effort |
|---|---|---|
| **Low** | Configuration change, single-section doc edit, declaring a budget, adding an ignore file | ≤ 1 day |
| **Medium** | Running a remediation skill across the codebase, writing a full framework doc, encoding one rule in CI | 1–5 days |
| **High** | Structural refactor, removing dual implementations, large test coverage uplift, monorepo restructuring | 1+ weeks |

Default banding for common items:

- **1.1 (consistency, dual implementations)** — High when migrations are incomplete; Medium when only a few deviations exist
- **1.3c, 1.3d (naming collisions, banned generics)** — Low to Medium depending on number of collisions
- **1.5a, 1.5b (dead code, comments)** — Low to Medium (use `dead-code-cleanup`)
- **1.5c (resolved feature flags)** — Medium (`dead-code-cleanup` Phase 3)
- **1.6a–c (boundaries, enforcement)** — Medium to High (use `boundary-audit`)
- **1.7a, 1.7b (error handling)** — Medium (use `error-handling-audit`)
- **1.9, 1.10, 1.11 (logging, deps, ops context)** — Low to Medium documentation work
- **1.12a, 1.12b (file length)** — Low for declaration; Medium for splitting large files
- **2.1b, 2.1g (ARCHITECTURE.md, named pattern)** — Medium (use `architecture-md`)
- **2.2a, 2.2b, 2.2f (AGENTS.md, RULES.md, length)** — Medium (use `agents-md`, `rules-md`)
- **2.1d, 2.1i, 2.1j (manifest, ignore, codebase-map)** — Low (use `ai-readiness-scaffold`)
- **2.2g, 2.3e (MCP, LSP)** — Low to Medium (documentation + install)
- **2.2d, 2.2e, 2.2h, 2.2i (T3 harness)** — Medium each (use `ai-readiness-scaffold` Phases 12–15)
- **2.3d (CI architectural invariants)** — Medium to High depending on language tooling
- **2.4a–e (escalation, owner)** — Low

### Dependencies

Some items must be done before others can be satisfied. Build a dependency list:

- **2.1g (named pattern)** must precede **1.6c (enforce pattern)**
- **1.12a (declared budget)** must precede **1.12b (CI ceiling)** — the ceiling references the budget
- **2.2b (RULES.md)** must precede convention-driven items (**1.3d banned generics**, **1.9a log semantics**, **1.12a budgets**) — these rules live in `RULES.md`
- **2.2a (AGENTS.md)** must precede items that document content in it (**1.10b version constraints**, **1.11c observability tooling**, **2.2g MCP inventory**, **2.4b escape hatches**)
- **2.1c (docs schema)** must precede **2.2c (AGENTS.md as index)** — the index needs somewhere to point
- **All T2 items** must be satisfied before **any T3 item** is meaningful — framework rule

When item A depends on item B, list B as a **prerequisite** in A's recommendation. The plan's ordering must respect these dependencies.

---

## Phase 3: Cross-Reference to Skills

Every recommendation should name the skill that can execute the fix where one exists. Use this mapping:

| Framework area | Skill |
|---|---|
| 1.5a, 1.5b, 1.5c (dead code) | `dead-code-cleanup` |
| 1.6a, 1.6b, 1.6c (boundaries) | `boundary-audit` |
| 1.7a, 1.7b (error handling) | `error-handling-audit` |
| 2.1b, 2.1g (architecture) | `architecture-md` |
| 2.2a, 2.2c, 2.2f (AGENTS.md) | `agents-md` |
| 2.2b, 1.3d, 1.12a, 1.9a (rules content) | `rules-md` |
| 2.1d, 2.1e, 2.1i, 2.1j, 2.1c, 1.11a, 1.12b, 2.2d, 2.2e, 2.2h, 2.2i, 2.4d (scaffolds) | `ai-readiness-scaffold` |
| Verification of any 1.x or 2.2b rule after a fix | `convention-check` |
| Pre-commit gate for an in-progress fix | `escalation-check` |
| Code quality items not covered by a remediation skill (1.1, 1.2, 1.8, 1.10) | manual work — no skill yet |

When a fix has no skill, write the action as concrete imperative steps rather than `<TODO>`.

---

## Phase 4: Group into Themes and Phases

The output is a consultant report, not a checklist. Items must be grouped twice: by **theme** for the Strategic Recommendations section, and by **phase** for the Implementation Roadmap.

### Themes (for Section 3 of the report)

Group failing items by the underlying capability gap they expose. Use themes the executive reader can act on directly, not framework rule IDs. Typical themes include:

- **Establish the orientation foundation** — items related to `AGENTS.md`, `RULES.md`, `ARCHITECTURE.md`, codebase map, manifests; affects whether agents can understand the project at all (Orient)
- **Resolve inconsistency in code quality** — items related to dual implementations, naming, dead code, error handling, file length; affects whether agent output is predictable (Act)
- **Make verification trustworthy** — items related to tests, reproducible environments, CI checks, LSP, pre-commit hooks; affects whether agents can close their own feedback loop (Verify)
- **Define the boundaries of agent autonomy** — items related to escape hatches, restricted areas, change isolation, agent-output review standards; affects whether agents stop when they should (Escalate)
- **Operationalise the agent harness** — items related to committed `.claude/`, sub-agents, skills, MCP inventory, hooks; relevant at T3
- **Address the monorepo blast radius** — items from Part 3 (monorepo additions); only if the project is a monorepo

The themes are not required to be exhaustive; pick the ones the gap report's content justifies. A project may have only two strategic themes if its findings concentrate in two areas. Do not invent themes to fill space.

For each theme, identify:
- The dominant tier of the items in it (most themes will span tiers; that is fine)
- Whether a single skill executes most of the theme (`rules-md`, `boundary-audit`, etc.) — if so, call it out as the primary lever
- Whether the theme has prerequisites in another theme (e.g. "Code quality" depends on "Orientation foundation" because rules must be declared before they are enforceable)

### Phases (for Section 4 of the report)

Translate the themes into a sequence the team can execute. Each phase should produce a re-auditable milestone — a point at which the audit can be re-run and the team can see measurable progress.

Phase construction rules:

1. **Never skip tiers.** Tier 0 items go in Phase 1 regardless of theme. Tier 1 items go in Phase 1 or 2. Tier 2 items go after Tier 1 is clean. Tier 3 items go last.
2. **Respect dependencies.** An item with a prerequisite cannot land in the same phase as its prerequisite — the prerequisite phase must complete first.
3. **Aim for phases that take 1–3 weeks each.** Single-week phases are too granular; phases longer than a month lose momentum and accumulate risk.
4. **Each phase has one outcome statement** — a verifiable condition the team can check. "Tier 1 items pass on re-audit" is verifiable; "Documentation improved" is not.

Typical phase structures (pick the one that matches the project's actual gaps):

- **Two-phase**: Foundation (Phase 1) → Code Quality and Verification (Phase 2). Suits projects already at Tier 1 working toward Tier 2.
- **Three-phase**: Foundation → Code Quality → Verification and Operations. Suits projects entering at Tier 0 working toward Tier 2.
- **Four-phase**: Foundation → Code Quality → Verification → Autonomy. Suits projects targeting Tier 3.

Do not pad with phases the gap report does not justify. A project with fifteen failing items does not need four phases.

### Quick wins call-out

Identify items that are Low effort and have no prerequisites. Note these in the roadmap as "can be done alongside the phase work" rather than as a separate bucket — the consultant framing prefers integrated guidance over fragmented backlogs.

---

## Phase 5: Produce the Report

Output a consultant-style remediation report using the structure below. Write in continuous prose where the template calls for narrative — do not reduce executive sections to bullet lists. The voice is authoritative and recommendation-led: "We recommend...", "The most pressing concern is...", "Before this work can proceed...".

Tables are appropriate for the roadmap and the appendix, but the executive summary, current-state assessment, and strategic recommendations sections should read as written analysis.

```markdown
# AI Readiness Remediation Report

**Prepared for:** [project name]
**Target tier:** [T0 Supervised / T1 Guided / T2 Safe / T3 Autonomous]
**Source assessment:** ai-readiness-audit gap report, [date]
**Date of this report:** [date]

---

## 1. Executive Summary

[One to two paragraphs in prose. Open by stating the project's current tier
position and the gap to the target. Name the single most consequential finding
— the one issue that most determines whether agent work will succeed. Close
with a one-sentence recommendation on the overall direction: "We recommend
prioritising X, then Y, then Z, over an estimated N weeks of focused effort."]

**Headline recommendations**

1. [First top-level recommendation, one sentence — the most important thing
   the team should do]
2. [Second top-level recommendation]
3. [Third top-level recommendation, if material]

**At a glance**

- Current tier: [T0 / T1 / T2 / T3 / None]
- Target tier: [T0 / T1 / T2 / T3]
- Failing items to address: [N]
- Estimated total effort to target: [X person-weeks, range]
- Earliest practical re-audit point: [milestone, e.g. "after the critical path completes"]

---

## 2. Current State Assessment

[Two to four paragraphs of diagnosis. This section names the underlying
problems, not just the symptoms. Group the findings into themes — "agent
orientation is incomplete", "code quality is inconsistent across layers",
"verification is informal" — and explain why each theme matters for the
specific risks the team is trying to manage. Reference the framework's four
capabilities (Orient, Act, Verify, Escalate) where useful, since they are the
shared vocabulary that connects the audit to the framework.]

[Identify any cross-cutting patterns. For example: "Several findings —
inconsistent error handling, missing logging conventions, untyped function
signatures — share a common root cause: the project has no enforced RULES.md.
Addressing the root cause is more effective than treating each symptom in
isolation."]

[If a fundamentally hostile pattern is present, this section must lead with it
and the report must recommend structural triage before any tier work. Do not
bury a hostile-pattern finding behind tier-N recommendations.]

---

## 3. Strategic Recommendations

The findings group into [N] strategic themes. Each is presented with a
recommendation, its rationale, and the framework rules and skills involved.

### 3.1 [Theme name, e.g. "Establish the orientation foundation"]

**Recommendation.** [One or two sentences. State what should be done, in plain
language. Avoid framework jargon at this level — the executive reader should
not need to look up rule IDs.]

**Why this matters.** [One paragraph. Explain what fails if this is not done,
or what the team gains by doing it. Tie to a concrete risk — "agents will
produce inconsistent output", "review burden will remain high", "the team
cannot safely raise the agent's autonomy".]

**What it involves.** [One paragraph or short bulleted list. Describe the
scope of work. Where a framework skill exists, name it: "We recommend running
the `rules-md` skill against the project to draft the initial content, then
having the lead engineer review and tighten it." Reference the framework rule
IDs in parentheses for traceability.]

**Effort.** [Low / Medium / High, with a person-day range. State the dependency
on any other theme.]

### 3.2 [Next theme — e.g. "Tighten code quality where it most affects agents"]

[Repeat the structure.]

[Typical themes to consider, grouped by what the gap report tends to reveal:

- "Establish the orientation foundation" — for projects missing AGENTS.md, RULES.md, ARCHITECTURE.md
- "Resolve inconsistency in code quality" — for projects with dual implementations, mixed error handling, dead code
- "Make verification trustworthy" — for projects without reliable tests, reproducible environments, or LSP
- "Define the boundaries of agent autonomy" — for projects with weak or absent escape hatches
- "Operationalise the agent harness" — for T3-targeting projects missing committed `.claude/` config
- "Address the monorepo blast radius" — for monorepos with weak package boundaries or no shared-library governance]

---

## 4. Implementation Roadmap

The work is structured in [N] phases. Each phase has a defined outcome and
should be completed before the next begins; the framework's tier model forbids
skipping tiers, and the same principle applies to this roadmap.

### Phase 1 — [Phase name, e.g. "Foundation"] (estimated [X weeks])

**Outcome.** [What "done" looks like for this phase. State it as a verifiable
condition: "Tier 0 items pass on re-audit", "AGENTS.md, RULES.md, and
ARCHITECTURE.md are present, under 200 lines each, and reviewed."]

**Work items**

| ID | Action | Skill | Effort | Depends on |
|----|--------|-------|--------|------------|
| [2.2a] | [concrete next step] | `agents-md` | Med | — |
| [2.1g] | [concrete next step] | `architecture-md` | Med | — |
| [...] | [...] | [...] | [...] | [...] |

**Recommended sequencing notes.** [One short paragraph. Call out anything
non-obvious — items that can run in parallel, items that must wait, items
whose effort is sensitive to codebase scale.]

### Phase 2 — [Phase name] (estimated [X weeks])

[Repeat. Group items into phases that respect dependencies and that produce
re-auditable milestones. A typical structure:

- Phase 1: Foundation — Tier 0/1 orientation and documentation
- Phase 2: Code quality — Tier 1 consistency, naming, dead code, error handling
- Phase 3: Verification and operations — Tier 2 tests, boundaries, observability
- Phase 4 (optional): Autonomy — Tier 3 harness, CI enforcement, governance

The phase count and naming should reflect what the gap report actually shows;
do not force four phases when the project only needs two.]

---

## 5. Risks and Considerations

[One short paragraph per risk. Cover at minimum:

- **Scope risk.** If the failing-item count is high, the temptation will be
  to address everything in parallel. Recommend resisting this — the framework
  tiers and the dependency graph mean some work is wasted if done out of order.

- **Codebase volatility.** If active feature development is ongoing, the
  baseline the audit captured may shift. Recommend a re-audit at the end of
  each phase.

- **Skill maturity.** Several of the recommended remediation skills produce
  drafts that require human review (notably `rules-md`, `architecture-md`,
  `agents-md`). Do not assume their output is final on first run.

- **Hostile patterns** if any were detected. These cannot be remediated by
  this plan; they require structural triage first.

- **Monorepo-specific risks** if relevant — cross-team coordination,
  shared-library change governance, build-order constraints.

- **Tier discipline.** Skipping tiers is the most common cause of agent
  workflow failure. Recommend resisting pressure to attempt Tier 3 work
  before Tier 2 is genuinely clean.]

---

## 6. Recommendations Summary

A condensed view of every recommendation, suitable for tracking.

| Phase | ID | Recommendation | Skill | Effort | Status |
|-------|----|----------------|-------|--------|--------|
| 1 | [ID] | [one-line summary] | `<skill>` | Low/Med/High | Open |
| 1 | [ID] | [one-line summary] | `<skill>` | Low/Med/High | Open |
| 2 | [ID] | [one-line summary] | `<skill>` | Low/Med/High | Open |

---

## Appendix A — Item-by-Item Findings

Every failing item from the source gap report, with the framework rule, the
proposed action, the executing skill, the effort estimate, and any
prerequisite gaps it depends on.

| ID | Tier | Finding | Action | Skill | Effort | Prerequisites |
|----|------|---------|--------|-------|--------|---------------|
| [ID] | [N] | [from gap report] | [concrete] | `<skill>` or manual | L/M/H | [IDs] |

---

## Appendix B — Items Excluded from This Plan

[List items that were in the gap report but excluded — usually because they
are above the target tier, or because the user marked them as deferred.
State each item's ID, tier, and the reason it was excluded.]

| ID | Tier | Reason for exclusion |
|----|------|----------------------|
| [ID] | T3 | Above target tier (current target is T2) |

---

## Appendix C — Method

This report was produced by the `ai-readiness-recommendations` skill from the
ai-readiness-framework. The skill reads an existing audit gap report and maps
each failing item to:

- A concrete next-step action
- An effort estimate (Low ≤ 1 day, Medium 1–5 days, High 1+ weeks)
- The framework skill, where one exists, that can execute the fix
- Any prerequisite items from elsewhere in the gap report

The roadmap is constructed by respecting tier order (lower tier before higher,
never skipping), dependency depth (items with no remaining prerequisites
first), and effort (low effort before high within the same depth). Themes in
Section 3 group items by the underlying capability gap they expose, rather
than by rule ID.

Effort estimates are calibrated for a single engineer or agent. Calendar time
will vary with team size, the codebase's existing condition, and competing
priorities; the report estimates effort, not delivery dates.
```

---

## Effort Estimation Notes

The effort bands are calibrated for a single engineer or agent working through the framework's remediation skills. Adjust for context:

- A team of three working in parallel on independent items can compress total time by ~2x — but not for items with serial dependencies
- A codebase that already passes most Tier 1 code quality items will see Tier 2 documentation work move faster than the default estimates suggest
- A codebase with many dual implementations or hundreds of test failures will see consistency and boundary work run **higher** than the default bands — surface this as a risk in the executive summary

Do not promise specific timelines. The plan estimates effort; calendar time depends on the team.

---

## Escalation Conditions

Stop and ask the user before proceeding if:

- No gap report can be found — recommend running `ai-readiness-audit` first rather than producing recommendations from an inferred audit
- The gap report is older than 30 days — the codebase may have moved; recommend re-running the audit before planning
- More than 30 items are failing — the plan will be too large to action as one piece; agree on a smaller target tier first (e.g. plan for T1 now, T2 next quarter) before producing recommendations
- A blocking condition (fundamentally hostile pattern from the audit) is present — recommendations are meaningless until the structural blocker is resolved; surface the blocker and stop
- Tier 3 is the stated target but Tier 2 items are still failing — push back; recommend re-targeting Tier 2 first
