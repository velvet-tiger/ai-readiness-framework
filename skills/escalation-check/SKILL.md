---
name: "Escalation Check"
description: "Check whether a proposed or completed change triggers any escape hatch conditions defined in the project's AGENTS.md or RULES.md. Use before committing a change, before raising a pull request, when unsure whether a change is within scope, or as a pre-commit gate in agent workflows."
---

# Escalation Check

## What This Skill Does

Reads the project's defined escape hatch conditions and checks whether the current change triggers any of them. If it does, stops and surfaces the decision for human review instead of proceeding.

This skill closes the escalation loop defined in the AI Readiness Framework: agents must know when to stop, and they must actually stop.

---

## Quick Start

Run this check before every commit or PR when working as an agent:

1. Read `AGENTS.md` and `RULES.md` to collect escape hatch conditions
2. Read the diff of the current change
3. Check the diff against every condition
4. If any condition is triggered: stop, report, and wait for human instruction
5. If no condition is triggered: confirm clear and proceed

---

## Phase 1: Collect Escape Hatch Conditions

Read the "Escape Hatches" or "Stop and Ask When" section from `AGENTS.md` and `RULES.md`.

If neither file exists, stop immediately:

> **ESCALATION REQUIRED**: This project has no `AGENTS.md` or `RULES.md`. Escape hatch conditions are not defined. Cannot proceed without human guidance. Consider running the `ai-readiness-scaffold` skill first.

If the files exist but contain no escape hatch section, treat the following as the default minimum set (this is the skill's own fallback, not a framework-mandated list — surface it as such in the report so the user knows the project should declare its own):

```
- T0 (2.4a)  The change touches a restricted directory or file declared in AGENTS.md
- T1 (2.4b)  The change requires adding or removing a dependency
- T1 (2.4b)  The change modifies a database migration destructively (dropping columns,
             changing types, removing indexes)
- T1 (2.4b)  The change touches code in any sensitive functional area named by the
             framework as a default escape hatch example: authentication, authorisation,
             billing, or money movement
- T1 (2.4b)  The change affects more than the stated scope of the task
- T1 (2.4c)  The branch contains more than one logical change, or the diff exceeds the
             project's declared PR-size limit (default: 400 lines)
- T1 (2.4b)  The correct pattern is genuinely ambiguous
```

Collect all conditions into a list. Each condition should be a clear, checkable predicate. Tag each one with the framework rule it derives from so the user can see which tier-level concern it covers.

---

## Phase 2: Read the Change

Determine what has changed. Depending on context:

- Run `git diff HEAD` to see unstaged changes
- Run `git diff --staged` to see staged changes
- Run `git diff [base-branch]...HEAD` to see all changes in the current branch
- Read a described set of planned changes if no diff is available yet

For each changed file, note:
- The file path
- What type of change it is (new file, modified, deleted)
- Which directories it touches
- Whether it touches any dependency manifest, migration file, or configuration file

---

## Phase 3: Evaluate Each Condition

Work through every escape hatch condition. For each one, determine:

- **TRIGGERED** — the change clearly matches this condition
- **POSSIBLY TRIGGERED** — the change might match; requires clarification
- **CLEAR** — the change does not match this condition

### Standard Conditions and How to Evaluate Them

**"The change requires adding or removing a dependency"**

Check: does the diff touch `package.json`, `composer.json`, `Cargo.toml`, `go.mod`, `requirements.txt`, `pyproject.toml`, or any equivalent manifest file?

If yes and dependencies have been added or removed: **TRIGGERED**

---

**"The change touches [named restricted directory or file]"**

Check: is any changed file path under the named directory or matching the named file pattern?

Common examples from project `AGENTS.md` files:
- `app/Auth/` or `app/Billing/` or `auth/` or `billing/`
- Any file containing `migration` in the path
- `config/security.php`, `config/auth.php`, or equivalent
- `packages/shared/` or `packages/types/`

If yes: **TRIGGERED**

---

**"The change modifies a database migration destructively"**

Check: does the diff touch any migration file? If yes, read the migration content and look for:
- Dropping a column (`dropColumn`, `DROP COLUMN`, `remove_column`)
- Changing a column type (`change`, `ALTER COLUMN`, `ALTER TYPE`, `modifyColumn`)
- Removing an index (`dropIndex`, `DROP INDEX`)
- Dropping a table (`dropTable`, `DROP TABLE`)
- Truncating data

If any of these are present: **TRIGGERED**

---

**"The change affects more than the stated scope"**

Check: does the diff include files that are not directly related to the stated task?

To evaluate this, you need the stated task description. If working from an issue, ticket, or user instruction:
- List the files changed
- List the files you would expect to change given the task
- Flag any file that does not have a clear connection to the task

If unexpected files are included: **POSSIBLY TRIGGERED** — explain which files seem out of scope

---

**"The change modifies a shared library or shared package"**

Check: does `AGENTS.md` list any paths as shared? Does the diff touch any file under those paths?

Also check: in a monorepo, does the diff touch a package that is listed as a dependency by more than one other package?

If yes: **TRIGGERED**

---

**"The change affects more than one top-level service directory" (monorepo)**

Check: collect the set of top-level service directories touched by the diff. If more than one: **TRIGGERED**

---

**"The correct pattern is genuinely ambiguous"**

This condition is evaluated at the time of writing code, not after. If you reached the point of committing, this condition was presumably resolved. However:

Flag it if: the change introduces a new pattern that does not clearly match any existing pattern, and no `RULES.md` rule covers it. In this case: **POSSIBLY TRIGGERED**

---

**"The branch contains more than one logical change, or the diff exceeds the PR-size limit" (framework rule 2.4c)**

Check the project's declared change-isolation rules in `AGENTS.md` (the "Change Isolation" section). If none are declared, fall back to the default: one logical change per branch, 400 lines maximum unless the change is purely mechanical (renames, formatting, generated-file updates).

Evaluate:
- Count the total non-mechanical changed lines in the diff (`git diff [base]...HEAD --stat`, then subtract lockfile, snapshot, and generated-file deltas)
- Scan the diff topology: does it span unrelated concerns (e.g. a feature change plus an unrelated refactor)?
- Check for mixed commit types in the branch's history (e.g. `feat:` plus `refactor:` plus `chore:` in the same branch)

If the diff is over the size limit and is not purely mechanical: **TRIGGERED**
If the branch mixes unrelated concerns: **POSSIBLY TRIGGERED** — describe which subsets should be separate branches.

---

## Phase 4: Report

### If any condition is TRIGGERED or POSSIBLY TRIGGERED

Stop. Do not commit. Do not raise a PR.

Output the following report:

```
## Escalation Required

The following escape hatch conditions were triggered by this change.
Human review is required before proceeding.

---

### TRIGGERED

**Condition:** [exact wording from AGENTS.md or RULES.md]
**Reason:** [specific file(s) or change(s) that triggered this condition]
**Decision needed:** [what the human needs to decide or approve]

---

### POSSIBLY TRIGGERED

**Condition:** [exact wording]
**Reason:** [why it might apply]
**Clarification needed:** [what would resolve the ambiguity]

---

### Next Steps

Please review the above and either:
1. Confirm it is safe to proceed (with any required approvals)
2. Instruct me to modify the change to bring it within scope
3. Take over this change yourself
```

### If all conditions are CLEAR

```
## Escalation Check: Clear

All escape hatch conditions checked. No conditions triggered.

**Change summary:** [brief description of what changed]
**Files changed:** [count] files across [directories]
**Conditions checked:** [count]

Safe to proceed.
```

---

## Running This Automatically

This check can be run:

- Manually, before any commit, by invoking this skill
- As part of a pre-commit checklist alongside `convention-check`
- At the start of a PR description generation step

The check should always run against the full branch diff, not just the most recent commit.

---

## Phase 5: Tier 3 Pre-Check (optional)

For projects targeting Tier 3, run the following additional checks before passing the change. These do not block at Tier 0–2 but should be surfaced as warnings.

**Agent configuration ownership (framework rule 2.4e)**

Check: does `AGENTS.md` declare a named person or team as the DRI for agent configuration?

Look for a section headed "Agent configuration owner", "Configuration owner", "Maintainers", or equivalent that names a specific person, team, or Slack channel.

If absent at Tier 3: **WARNING** — escalation rules cannot be governed without an owner. Recommend adding a DRI before relying on the escape hatch process at Tier 3.

**Harness configuration committed (framework rule 2.2h)**

Check: does the project have a committed `.claude/` or `.agents/` directory declaring hooks, sub-agents, skills, and MCP configuration?

If absent at Tier 3: **WARNING** — without committed harness configuration, escalation behaviour depends on each developer's local setup and cannot be relied on.

**Agent output review standard (framework rule 2.4d)**

Check: does `CONTRIBUTING.md` (or the project's contribution guide) declare what reviewers should check specifically on agent-authored PRs, distinct from the standard for human-authored changes?

Look for a section headed "Reviewing agent PRs", "Agent-authored changes", "AI-generated code", or equivalent that names review concerns particular to agent output (e.g. scope creep, fabricated dependencies, silent file moves outside the stated task).

If absent at Tier 3: **WARNING** — without a declared review standard, reviewers handle agent PRs the same as human PRs, missing the failure modes specific to agent-generated code. Recommend adding the section before relying on agent autonomy at Tier 3.

All three warnings are advisory at this stage; they become hard requirements only once the project is operating at Tier 3 autonomy.

---

## Escalation Conditions for This Skill

This skill itself should escalate if:

- `AGENTS.md` and `RULES.md` both exist but contain no escape hatch section — flag this as a gap rather than silently applying defaults
- The escape hatch conditions in the two files contradict each other — flag the contradiction rather than picking one
- The change is so large that it cannot be evaluated as a unit — flag this as a scope issue
- The project declares itself as Tier 3 but no agent configuration owner is named in `AGENTS.md` (framework rule 2.4e) — flag this before the change is accepted
