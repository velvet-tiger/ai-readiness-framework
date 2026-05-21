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

If the files exist but contain no escape hatch section, treat the following as the default minimum set:

```
- The change requires adding or removing a dependency
- The change modifies a database migration destructively (dropping columns, changing types, removing indexes)
- The change affects more than the stated scope of the task
- The correct pattern is genuinely ambiguous
```

Collect all conditions into a list. Each condition should be a clear, checkable predicate.

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

## Escalation Conditions for This Skill

This skill itself should escalate if:

- `AGENTS.md` and `RULES.md` both exist but contain no escape hatch section — flag this as a gap rather than silently applying defaults
- The escape hatch conditions in the two files contradict each other — flag the contradiction rather than picking one
- The change is so large that it cannot be evaluated as a unit — flag this as a scope issue
