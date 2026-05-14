---
name: "Convention Check"
description: "Verify that new or modified code follows the conventions established in the project's RULES.md and AGENTS.md. Use before committing a change, when reviewing agent-generated code, when adding new code to an existing codebase, or as a consistency gate before raising a pull request."
---

# Convention Check

## What This Skill Does

Reads the project's established conventions and checks whether new or modified code follows them. Produces a deviation report with specific findings and suggested corrections.

This skill addresses a core problem: agents produce consistent output only when conventions are clear. This check closes the loop by verifying the output before it lands.

---

## Quick Start

1. Read `RULES.md` and the relevant sections of `AGENTS.md`
2. Read the changed files (diff or full files)
3. Check each changed file against the conventions
4. Report deviations with file paths, line references, and suggested corrections

---

## Phase 1: Load Conventions

Read `RULES.md` in full. Extract the rules into categories:

- Consistency rules (dominant pattern, file placement)
- Explicitness rules (type hints, dependency injection)
- Naming rules (banned names, required patterns, collision policy)
- Error handling rules (canonical approach, banned patterns)
- State and side effects rules
- Boundary rules (layer constraints)
- Test rules
- Dead code rules
- Logging rules
- Dependency rules
- Environment and configuration rules
- File length budgets (per-category soft budgets and hard ceiling)

Also read `AGENTS.md` for:
- Repository layout (where things should live)
- Key conventions section
- Known framework magic (to avoid false positives)

If neither file exists, stop:

> **Cannot run convention check**: No `RULES.md` or `AGENTS.md` found. Conventions are not defined. Consider running the `ai-readiness-scaffold` skill first.

---

## Phase 2: Read the Changes

Read the set of files to check. This may be:

- Files modified in the current branch (`git diff [base]...HEAD --name-only`)
- Specific files the user has pointed to
- All files in a new feature directory

For each file, read the full content. Do not check only the diff lines — a new function added to a file that already has violations should be checked for the new function's conformance, not the existing violations.

---

## Phase 3: Check Each Convention

Work through each convention category. For each file and each category, determine whether the new code follows the rule.

### Consistency

**Check:** Does the new code use the established pattern for its concern?

Look for:
- New classes placed in unexpected directories
- New patterns introduced alongside existing ones (a new `Manager` class when the pattern is `Action` classes)
- A second approach to something that already has an established approach

**Flag if:** New code uses a different pattern from what is established for the same concern.

---

### Explicitness

**Check:** Are type hints, return types, and dependency injection done correctly?

**PHP:**
- All method parameters have type hints
- All methods have return type declarations
- No `mixed` return types where a specific type is possible
- Dependencies arrive via constructor, not `app()` or `resolve()` mid-method
- No `__call`, `__get`, or dynamic dispatch without a doc comment

**TypeScript:**
- All function parameters are typed
- All functions have explicit return types
- No `any` unless explicitly justified
- No `as unknown as T` casts

**Rust:**
- All public functions have documented types
- No `unwrap()` or `expect()` outside tests

**Flag if:** Any parameter, return type, or dependency is untyped where a type is available.

---

### Naming

**Check:** Do names communicate intent without tracing? Are they unique enough across the codebase?

Look for:
- Generic method names: `handle`, `process`, `manage`, `run`, `data`, `result`, `info`
- Unqualified generic class/type names: `Service`, `Helper`, `Manager`, `Handler`, `Utils`, `Data`, `Info`, `Result` — these must carry a domain prefix or suffix (framework rule 1.3d)
- Variable names that require reading the function body to understand (`$d`, `$tmp`, `$item` in a complex context)
- Names that reuse an existing term for a different concept
- Names that do not match the established convention for their type (e.g. an event not named in past tense, an action not named as `[Verb][Noun]`)

**Flag if:** Any name in new code is on the banned list or does not follow the naming convention for its type.

---

### Naming Collisions

**Check:** Does any newly introduced or renamed identifier collide with an existing one elsewhere in the repo? (Framework rule 1.3c.)

Build collision sets across the whole repo, not just the diff:

1. **Basename collisions** — group all source files by basename (`helpers.ts`, `utils.py`, `config.go`); any group with more than one entry is a collision set.
2. **Class/type collisions** — extract all exported class, type, interface, enum, and trait simple names; group by name; any group with more than one entry is a collision set.
3. **Function collisions** — extract all exported function simple names (or top-level functions in languages without classes); group by name; any group with more than one entry is a collision set.

For each collision set, decide whether it is **load-bearing**:

- Parallel adapters implementing the same port (e.g. `StripeGateway` and `PaypalGateway` both implementing `PaymentGateway`) → load-bearing; collision is allowed and should be documented in `ARCHITECTURE.md` or `RULES.md`.
- Test fixtures with structurally identical setup files → typically load-bearing per directory.
- Anything else → not load-bearing; rename.

**Flag if:**

- The current change introduces a new identifier that lands in a non-load-bearing collision set
- The current change introduces a new basename that already exists elsewhere and is not in a recognised adapter directory

When flagging, list every member of the collision set so the user can decide which to rename.

---

### Error Handling

**Check:** Does new code use the canonical error handling approach?

Look for:
- Empty catch blocks
- Catch blocks that return null
- Catching a broad base type (`\Throwable`, `Exception`, `Error`) when a specific type is expected
- Returning null to signal failure instead of throwing or returning a typed error
- A different error handling pattern from the one established in `RULES.md`

**Flag if:** Any error handling in new code deviates from the canonical approach.

---

### State and Side Effects

**Check:** Are state and side effects handled correctly?

Look for:
- New static properties or global variables
- Methods named `get*` or `find*` that also write or have side effects
- Functions that modify state outside their own scope without the name signalling this

**Flag if:** Any state or side effect rule is violated.

---

### Boundaries

**Check:** Is new code placed in the correct layer and staying within its boundaries?

Look for:
- Business logic in a controller or HTTP handler
- Database queries outside the established data access layer
- HTTP concerns (`$request->`, response objects) in a service or action class
- Calls to another service's internal code rather than its declared interface

**Flag if:** Any code crosses a layer boundary it should not cross.

---

### Tests

**Check:** Does new behaviour have tests? Are the tests correctly written?

Look for:
- New public methods with no corresponding test
- Tests that assert on internal state or mock call counts (rather than observable behaviour)
- Tests using trivial placeholder data (`'test'`, `1`, `'email@example.com'`)
- Tests that do not exercise realistic edge cases

**Flag if:**
- New public behaviour has no test
- Tests in new code do not follow the established test style

---

### Dead Code

**Check:** Does new code introduce dead code?

Look for:
- Imported modules or classes that are not used in the file
- Declared variables that are never read
- Parameters that are accepted but ignored
- Feature flags that are evaluated but always true or always false

**Flag if:** Any dead code is present in new additions.

---

### Logging

**Check:** Does new code log correctly?

Look for:
- Use of `echo`, `print`, `var_dump`, `dd()`, `console.log`, `dbg!()` or equivalent
- Log statements missing required structured fields (as defined in `RULES.md`)
- Wrong log level (using `DEBUG` for errors, `ERROR` for informational events)
- Logging of sensitive data (credentials, tokens, PII)

**Flag if:** Any logging in new code violates the established logging rules.

---

### Dependencies

**Check:** Has new code introduced dependencies?

Look for:
- New entries in `package.json`, `composer.json`, `Cargo.toml`, or equivalent
- `import`/`use`/`require` of a package not already in the manifest

**Flag if:** Any new dependency is added (this triggers the escalation check too — both checks may fire).

---

### File Length

**Check:** Do new or modified files respect the budgets declared in `RULES.md` (framework rules 1.12a and 1.12b)?

For each changed file:

1. Count lines (`wc -l <path>` or equivalent).
2. Compare against the category soft budget from `RULES.md` (e.g. 500 lines general, 300 components, 800 fixtures).
3. Compare against the hard ceiling (default 1500 lines).
4. Check whether the file is on the allowlist (`.file-length-ignore` or CI config) — allowlisted files are skipped.

**Flag if:**

- The file exceeds the soft budget for its category, with no inline comment justifying it
- The file exceeds the hard ceiling

When flagging a soft-budget breach, suggest a split direction (by concern, not by line count). When flagging a hard-ceiling breach, treat it as a blocker.

---

### Environment and Configuration

**Check:** Are environment variables used correctly?

Look for:
- Hardcoded URLs, credentials, or environment-specific values
- New environment variable names not present in `.env.example`
- `env()`, `getenv()`, or `process.env` called deep in business logic rather than at the boundary

**Flag if:** Any configuration rule is violated.

---

## Phase 4: Report

### If deviations are found

```
## Convention Check: Deviations Found

[N] deviation(s) found across [M] file(s). Fix before committing.

---

### [path/to/file.php]

**Naming** — `handle()` is a banned generic name
Line 24: `public function handle(Request $request)`
Suggestion: Rename to describe what this method actually does, e.g. `authorisePayment()`

**Error Handling** — empty catch block
Lines 45–47:
    } catch (PaymentException $e) {
    }
Suggestion: Rethrow, log and rethrow, or return a typed error result. Do not swallow.

**Explicitness** — missing return type declaration
Line 24: `public function handle(Request $request)`
Suggestion: Add return type, e.g. `public function handle(Request $request): JsonResponse`

---

### [path/to/service.ts]

**Boundaries** — direct database query in service layer
Line 78: `const rows = await db.query('SELECT * FROM orders WHERE ...')`
Suggestion: Move this query into the OrderRepository and call it from here.

---

### Summary

Fix the above before committing. If any item requires a design decision, use the escalation-check skill or stop and ask.
```

### If no deviations are found

```
## Convention Check: Clear

[N] file(s) checked against [M] conventions. No deviations found.

Files checked:
- path/to/file.php
- path/to/service.ts

Safe to proceed.
```

---

## Scope

This check evaluates **new and modified code only**. It does not report pre-existing violations in unchanged lines. The purpose is to prevent new violations from being introduced, not to enforce a complete cleanup in a single pass.

If pre-existing violations are found and are significant, note them separately as recommended cleanup but do not block the current change for them.

---

## Escalation Conditions

Stop and ask before proceeding if:

- `RULES.md` is absent or contains fewer than 5 rules — the conventions are not sufficient to run a meaningful check; flag this rather than producing a shallow report
- A deviation is found but the correct fix is genuinely ambiguous — two interpretations of the rule are equally valid — flag the ambiguity rather than picking one
- The number of deviations is very large (more than 20 in new code) — this suggests the rules were not understood at all; flag this rather than listing every one
