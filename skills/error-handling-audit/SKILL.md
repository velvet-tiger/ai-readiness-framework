---
name: "Error Handling Audit"
description: "Audit and remediate error handling inconsistencies in a project following the AI Readiness Framework. Use when preparing a codebase for agent work, when error handling is inconsistent or uses multiple patterns, when silent failures or empty catch blocks exist, or when agents are producing error handling code that does not match the rest of the codebase."
---

# Error Handling Audit

## What This Skill Does

Finds and fixes error handling that makes it impossible for agents to reason about the failure surface:

- **Silent failures** — exceptions caught and discarded, errors swallowed, null returned to signal failure
- **Inconsistent patterns** — multiple error handling approaches coexisting in the same codebase
- **Ambiguous nulls** — functions that return null for both "not found" and "failure"
- **Over-broad catches** — catching base Exception/Throwable/Error when a specific type is expected

Agents replicate whatever error handling pattern they encounter first. A codebase with mixed patterns produces agents that average between them — satisfying neither.

---

## Quick Start

1. Identify the established error handling approach (or establish one if none exists)
2. Search for violations of that approach
3. Remediate in order: silent failures first, then inconsistencies, then ambiguous nulls

---

## Phase 1: Identify the Established Approach

Read 5–10 service or business logic files. Determine which approach is dominant:

### Common Approaches by Language

**PHP:**
- Typed domain exceptions (`throw new PaymentDeclinedException(...)`)
- Result objects (`Result::ok($value)` / `Result::err($error)`)
- Mixed (common problem — treat the typed exception approach as canonical if it exists at all)

**TypeScript / JavaScript:**
- `throw new Error(...)` with typed error classes
- Result/Either types (`{ ok: true, value } | { ok: false, error }`)
- Promise rejection with typed errors
- Mixed (common — establish which is canonical)

**Rust:**
- `Result<T, E>` with typed error enums (always the right answer)
- Confirm: are error types defined per module, per crate, or using `anyhow`/`thiserror`?

**Go:**
- `error` return values with typed sentinel errors or wrapped errors
- Confirm: is `fmt.Errorf("...: %w", err)` wrapping used consistently?

**Python:**
- Typed exceptions in a defined hierarchy
- Explicit `raise` at failure points

### Document the Canonical Approach

Before making changes, write down:

1. The established approach (one sentence)
2. Where typed errors/exceptions are defined (directory or module)
3. Where errors are caught and translated into responses (the boundary — HTTP handler, main function, etc.)
4. Any documented exceptions to the rule (in `AGENTS.md` or `RULES.md`)

If there is no clear canonical approach, stop and decide with the user before proceeding. Do not make one up.

---

## Phase 2: Find Silent Failures

### PHP

Search for:
```
catch.*Exception.*\{[^}]*\}           # empty catch blocks
catch.*\$e\).*\{\s*\}                 # empty catch with variable
catch.*\{\s*return null               # null swallowing
catch.*\{\s*\/\/                      # commented body
```

Patterns to look for manually:
```php
// Silent swallow — DELETE the catch or handle properly
try {
    $result = $this->process($data);
} catch (Exception $e) {
    // nothing here
}

// Null return on failure — REPLACE with typed exception
try {
    return $this->findOrder($id);
} catch (OrderNotFoundException $e) {
    return null;
}

// Over-broad catch — NARROW to the specific exception type
try {
    $this->chargeCard($amount);
} catch (\Throwable $e) {
    Log::error('something went wrong');
}
```

### TypeScript / JavaScript

Search for:
```
catch\s*\([^)]*\)\s*\{\s*\}           # empty catch
catch.*\{\s*return (null|undefined)   # null/undefined swallow
catch.*\{\s*\/\/                      # commented body
\.catch\(\s*\(\)\s*=>                 # swallowed promise rejection
```

### Rust

The compiler catches most issues. Look for:
```
.unwrap()    # only permitted in tests
.expect()    # only permitted in tests or with a message that includes context
let _ =      # discarded Result — may indicate silently ignored failure
```

### Go

Search for:
```
err != nil.*{$    # followed by nothing — i.e. the if body is empty
_, err :=         # error assigned to blank identifier
if err != nil { continue }  # without logging or handling
```

### Python

Search for:
```
except.*:$        # bare except with empty body
except.*pass      # exception swallowed with pass
except.*:.*return None  # None on failure
```

---

## Phase 3: Find Inconsistent Patterns

After identifying the canonical approach, find code using a different approach.

### If canonical is typed exceptions

Look for:
- Functions returning null, bool, or a special value to signal failure instead of throwing
- Functions using a Result/Either pattern where the rest of the code uses exceptions
- Error codes returned as integers or strings

### If canonical is Result types

Look for:
- Functions throwing exceptions where they should return `Result::err(...)`
- Functions returning null where they should return `Result::err(...)`
- `try/catch` blocks in business logic where the caller should be checking `Result`

### If canonical is Rust Result

Look for:
- `.unwrap()` or `.expect()` outside of tests
- Functions that return `Option` where `Result` is more appropriate (when the absence is an error)
- Inconsistent error type usage (some functions use `anyhow`, others use typed enums)

---

## Phase 4: Find Ambiguous Nulls

A function should not return null to mean both "this thing does not exist" and "something went wrong finding this thing". These are different outcomes and should be handled differently.

Look for functions that:
- Return `T | null` where null has more than one meaning
- Are documented as returning null "if not found or if an error occurs"
- Have callers that check `if (result === null)` and do not distinguish between not-found and error

---

## Remediation

> **When introducing log calls** as part of a fix (e.g. log-and-rethrow, or logging a declined card), follow the project's logging rules from `RULES.md` (framework 1.9a, 1.9b, 1.9c): use the established logger only, emit the required structured fields, and pick the log level whose declared semantics match the situation. A typed error path that logs at the wrong level or omits the required fields produces structurally inconsistent output and undoes part of the gain from this audit.

### Fix: Empty Catch Block

Option A — the exception should propagate:
```php
// Before
try {
    $this->processPayment($order);
} catch (PaymentException $e) {
    // nothing
}

// After — let it propagate; the caller or boundary handler deals with it
$this->processPayment($order);
```

Option B — the exception should be logged and rethrown:
```php
// After — log context, then rethrow
try {
    $this->processPayment($order);
} catch (PaymentException $e) {
    Log::error('Payment processing failed', [
        'order_id' => $order->id,
        'error' => $e->getMessage(),
    ]);
    throw $e;
}
```

Option C — the failure is genuinely expected and should produce a typed result:
```php
// After — explicit typed handling
try {
    $this->processPayment($order);
} catch (CardDeclinedException $e) {
    return PaymentResult::declined($e->reason());
}
```

### Fix: Null Returned on Failure

```typescript
// Before
async function findUser(id: string): Promise<User | null> {
    try {
        return await db.users.findById(id);
    } catch (err) {
        return null;  // ambiguous: not found? DB error?
    }
}

// After — separate the two outcomes
async function findUser(id: string): Promise<User> {
    const user = await db.users.findById(id);  // throws NotFoundError if missing
    return user;  // throws DatabaseError if connection fails — propagates naturally
}
```

### Fix: Over-Broad Catch

```php
// Before
try {
    $this->chargeCard($amount, $card);
} catch (\Throwable $e) {
    Log::error('error', ['msg' => $e->getMessage()]);
    return null;
}

// After — catch only what is expected; let the unexpected propagate
try {
    $this->chargeCard($amount, $card);
} catch (CardDeclinedException $e) {
    Log::warning('Card declined', ['amount' => $amount, 'reason' => $e->reason()]);
    return ChargeResult::declined($e->reason());
} catch (PaymentGatewayException $e) {
    Log::error('Payment gateway error', ['amount' => $amount, 'error' => $e->getMessage()]);
    throw $e;  // unexpected — propagate up to the boundary handler
}
```

---

## Commit Strategy

One commit per type of fix:

```
fix: remove silent exception swallowing in payment service

Replaced 4 empty catch blocks with explicit typed handling.
Failures now propagate or return typed error results.
```

```
refactor: normalise error handling to typed exceptions

Replaced Result-returning pattern in OrderRepository with typed exceptions
to match the rest of the codebase. No behaviour change.
```

---

## Quality Check

After remediation:

- [ ] No empty catch blocks remain
- [ ] No functions return null to signal failure (unless null is a valid domain value)
- [ ] No over-broad catches of base Exception/Throwable/Error in business logic
- [ ] All business logic uses the canonical error handling approach
- [ ] All error catch points at the boundary (HTTP handlers, queue consumers, main) translate errors to responses
- [ ] Tests pass

---

## Escalation Conditions

Stop and ask before proceeding if:

- No canonical error handling approach can be identified — two equally prevalent patterns exist — ask the user which to standardise on
- A silent failure appears to be intentional (e.g. a background job that must not crash even if one item fails) — confirm the intended behaviour before changing it
- Removing a null return changes the call site contract — there may be many callers to update
- The changes are large enough that they cannot be reviewed in a single PR — agree on scope before proceeding
