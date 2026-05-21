---
name: "Dead Code Cleanup"
description: "Identify and remove dead code from a project following the AI Readiness Framework. Use when preparing a codebase for agent work, when static analysis has flagged unused code, when commented-out code blocks exist, or when permanently resolved feature flags are cluttering the codebase."
---

# Dead Code Cleanup

## What This Skill Does

Finds and removes three categories of dead code that harm agent effectiveness:

1. **Unused code** — classes, methods, functions, variables, parameters, and imports that are never called
2. **Commented-out code** — code that has been disabled by commenting rather than deleted
3. **Resolved feature flags** — conditional branches whose condition is always true or always false

Dead code is actively harmful to agents. They read it as valid signal, attempt to reconcile it with the live code, and produce output consistent with the dead path rather than the live one.

---

## Quick Start

Work through the three phases in order. Do not mix them — complete one before starting the next.

1. **Phase 1**: Find and remove commented-out code (safest, clearest wins)
2. **Phase 2**: Find and remove resolved feature flags
3. **Phase 3**: Find and remove unused code (requires more care)

Commit after each phase. Small, focused commits make review easier and reduce risk.

---

## Phase 1: Commented-Out Code

### What to Look For

Blocks of code that have been disabled by adding comment markers. These are distinct from explanatory comments — look for:

- Multi-line blocks surrounded by `//`, `#`, `--`, or `/* */`
- Code syntax inside comments (function calls, control flow, assignments)
- Comments like `// old implementation`, `// TODO: remove`, `// was:`, `// disabled`, `// deprecated`

### How to Find Them

Search for patterns that indicate commented-out code rather than explanatory comments:

- Long comment blocks (more than 2–3 lines) containing code syntax
- Commented-out import statements
- Commented-out function or class definitions
- Commented-out closing braces or brackets

### What to Do

**Delete the commented-out block entirely.** Do not:
- Move it somewhere else
- Add a note explaining why it was removed
- Keep it "just in case"

The commit history preserves the deleted code. It can always be recovered if needed.

### Exceptions

Legitimate comment patterns to leave in place:
- Explanatory comments describing why something is done a certain way
- Licence headers
- Generated code markers
- Temporarily disabled test cases where there is an open issue — leave the comment but note it

---

## Phase 2: Resolved Feature Flags

### What to Look For

Conditional branches controlled by a configuration value, environment variable, or constant that is effectively always true or always false.

Common patterns:

```php
// PHP
if (config('features.new_checkout')) { ... }
if (Feature::active('dark_mode')) { ... }
if (env('ENABLE_LEGACY_API')) { ... }
```

```typescript
// TypeScript
if (process.env.FEATURE_FLAG_NEW_UI === 'true') { ... }
if (flags.isEnabled('beta_feature')) { ... }
```

```rust
// Rust
if cfg!(feature = "legacy_mode") { ... }
```

### How to Identify Resolved Flags

A flag is resolved (always true or always false) if:
- The controlling config/env value is hardcoded in all environments
- The feature was part of a rollout that is now complete and the flag is always on
- The feature was abandoned and the flag is always off
- The constant or config key no longer exists (making the branch always false)

Check:
- `.env.example` and environment config files
- Feature flag management code or database
- CI/CD environment variable definitions

### What to Do

**Always-true flag**: Inline the true branch. Delete the condition and the false branch.

Before:
```php
if (config('features.new_checkout')) {
    return $this->newCheckout($cart);
} else {
    return $this->legacyCheckout($cart);
}
```

After:
```php
return $this->newCheckout($cart);
```

**Always-false flag**: Delete the entire conditional and the true branch. Keep only whatever comes after.

**Also delete:**
- The flag constant or config key declaration
- The flag registration in any feature flag manager
- Tests that only test the dead branch
- Documentation of the flag

### When to Stop

Stop and ask if:
- You cannot determine from the codebase or config whether a flag is currently on or off
- The flag appears to be environment-specific (on in production, off in staging) — this is not a resolved flag
- The flag is used for A/B testing that is still active

---

## Phase 3: Unused Code

### What to Look For

Code that is defined but never called, referenced, or imported:

- Functions and methods with no callers
- Classes with no instantiations or references
- Variables declared but never read
- Parameters accepted but never used
- Imports of modules that are not used in the file
- Constants defined but never referenced

### How to Find Them

**Use static analysis tools appropriate to the language.** Do not rely solely on text search.

| Language | Tools |
|----------|-------|
| PHP | PHPStan (`--level=6` or higher), Psalm |
| TypeScript / JavaScript | TypeScript compiler (`noUnusedLocals`, `noUnusedParameters`), ESLint (`no-unused-vars`) |
| Rust | `cargo check` — the compiler reports unused items |
| Python | `pylint`, `vulture`, `pyflakes` |
| Go | `go vet`, `staticcheck` |
| Ruby | `rubocop`, `debride` |

Run the tool and collect the full output before making any changes.

### Priority Order

Work from lowest risk to highest:

1. **Unused imports** — safe to remove; no behaviour change
2. **Unused local variables** — safe to remove; no behaviour change
3. **Unused private methods** — safe to remove; not accessible outside the class
4. **Unused private classes** — safe to remove; not accessible outside the module
5. **Unused public methods** — requires checking for dynamic dispatch, reflection, or external callers (see caution below)
6. **Unused public classes** — same caution as public methods

### Caution: Public API and Dynamic Dispatch

Do not remove a public method or class without first checking:

- Is it called via reflection, `call_user_func`, `method_exists`, or equivalent dynamic dispatch?
- Is it part of an interface or abstract class contract?
- Is it in a library that is consumed by other packages outside this repository?
- Is it a framework hook that the framework calls by convention (e.g. `boot()`, `register()`, `handle()`)?
- Is it used in a template, view, or configuration file (not always caught by code-level search)?

If any of these are true, do not delete without confirming.

### What to Do

Delete unused code entirely. Do not comment it out — that creates the same problem in a different form.

For each deleted item:
- Delete the definition
- Delete any associated test that only tests the deleted code
- Delete any documentation that only documents the deleted code
- Check whether the deletion exposes further unused code (cascade)

---

## Commit Strategy

Make one commit per category of change:

```
chore: remove commented-out code blocks

Deleted 12 commented-out code blocks across the codebase.
All deleted code is preserved in git history.
```

```
chore: resolve permanently-enabled feature flags

Inlined the live branch for flags: new_checkout, v2_api_enabled, stripe_v3.
Deleted the dead legacy branches and their tests.
```

```
chore: remove unused code identified by PHPStan

Removed 8 unused private methods, 14 unused variables, and 3 unused imports.
No behaviour change.
```

---

## Quality Check

After completing all three phases:

- [ ] No commented-out code blocks remain (spot-check 5–10 files)
- [ ] Static analysis no longer reports unused code items that were present before
- [ ] All identified resolved feature flags have been inlined or deleted
- [ ] Tests still pass
- [ ] No new lint warnings introduced

---

## Escalation Conditions

Stop and ask before proceeding if:

- A public method or class flagged as unused is in a directory that suggests it is a public API or SDK entry point
- A feature flag cannot be confirmed as resolved from the codebase alone — it may require checking a deployment configuration or feature flag service
- Deleting a class would require more than 5 call sites to be updated — confirm scope before proceeding
- Static analysis produces hundreds of findings — agree on a prioritised scope before beginning
