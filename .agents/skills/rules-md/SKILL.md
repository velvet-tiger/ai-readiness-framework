---
name: "RULES.md Author"
description: "Create or update RULES.md for a project following the AI Readiness Framework. Use when a project is missing its coding rules file, when RULES.md exists but is too generic to produce consistent agent output, when the codebase has adopted new patterns that should be codified, or when an audit has flagged insufficient or contradictory rules."
---

# RULES.md Author

## What This Skill Does

Creates or updates `RULES.md` — the file that gives AI agents specific, enforceable coding rules so that two agents working independently produce consistent output.

Generic rules ("write clean code", "follow best practices") are useless. The output of this skill must be specific enough that an agent reading it has no ambiguity about what correct code looks like in this project.

---

## Quick Start

1. Read any existing `RULES.md`, `AGENTS.md`, or `CLAUDE.md`
2. Read 3–5 representative source files to understand actual conventions in use
3. Read the package manifest to confirm language and framework versions
4. Draft the rules file using the structure below
5. Every rule must be specific to what is actually observed — not generic boilerplate

If the file already exists, compare each section against the spec. Add or sharpen rules that are missing or vague. Do not remove rules that are correct.

---

## Sampling Strategy

Before writing, read enough code to understand actual conventions:

- 2–3 business logic files (services, use cases, action classes)
- 2–3 HTTP handler files (controllers, route handlers, resolvers)
- 2–3 test files
- The main entry point
- The error handling infrastructure (exception handler, result types, error enums)

Look for: how errors are returned, how dependencies arrive, how methods are named, what the layer structure is, what the test style is.

---

## Required Sections

Each section must contain specific, actionable rules. Use imperative statements. Every rule must be falsifiable — an agent must be able to look at a piece of code and determine whether it passes or fails the rule.

---

### Consistency

Rules about following the dominant pattern and not introducing alternatives.

Write the actual pattern name. Do not write "follow the dominant pattern" — name it.

**Example (Laravel / Action classes):**
```markdown
## Consistency

- All business logic lives in Action classes under `app/Actions/`. Do not create service classes,
  manager classes, or helper classes as alternatives.
- Controllers exist only to translate HTTP requests into Action calls. If you find yourself writing
  logic in a controller, move it to an Action.
- If two patterns exist for the same concern, stop and ask before proceeding. Do not average between
  them or introduce a third.
- Place every file where its name and the directory structure predict it should be. Do not create
  new directories without documenting them in `AGENTS.md`.
```

---

### Explicitness

Rules about type hints, dependency injection, and avoiding hidden behaviour.

Tailor to the language. PHP rules look different from Rust rules.

**Example (PHP 8.3):**
```markdown
## Explicitness

- All method and function signatures must carry full type hints including return types. No `mixed`,
  no untyped arrays, no omitted return types.
- Use named constructor injection for all dependencies. Never resolve from the container
  mid-method. Never use `app()` or `resolve()` inside a class body.
- Document any facade or magic method with a doc comment at the call site explaining what it
  resolves to.
- If a function does more than its name says, rename it or split it into two functions.
```

**Example (TypeScript):**
```markdown
## Explicitness

- All functions must have explicit parameter types and return types. No implicit `any`.
- Use `interface` for object shapes passed across module boundaries. Use `type` for unions,
  intersections, and utility types.
- No `as unknown as T` casts. If you need a cast, it is a signal that the types are wrong.
- Inject dependencies as constructor parameters or function arguments. No module-level singletons
  with mutable state.
```

---

### Naming

Specific naming rules for this project. Include banned names and required patterns.

**Example:**
```markdown
## Naming

- Names must communicate intent without requiring the reader to trace the call chain. Prefer long
  and explicit over short and ambiguous.
- A name must mean one thing across the entire codebase. If a name is already used for a different
  concept, choose a different name.
- Banned generic names: `handle`, `process`, `manage`, `run`, `execute` (unless the class name
  provides the missing context, e.g. `ProcessPaymentRefund::execute()`), `data`, `result`, `info`,
  `manager`, `helper`, `util`, `service` (as a suffix without a specific domain noun).
- Action classes: named as `[Verb][Noun]` — `CreateSubscription`, `CancelOrder`, `RefundPayment`
- Events: named in past tense — `PaymentSucceeded`, `OrderCancelled`, `SubscriptionCreated`
- Exceptions: named as `[Noun][Problem]Exception` — `PaymentDeclinedException`,
  `OrderNotFoundException`
```

---

### Error Handling

The single most important section. Name the specific approach used in this project.

**Example (PHP / typed exceptions):**
```markdown
## Error Handling

- Use typed domain exceptions. Every failure condition has its own exception class under
  `app/Exceptions/Domain/`.
- Never return `null` to signal failure. Throw an exception or return a typed result object.
- Never use an empty catch block. Every catch block must either rethrow, return a typed error,
  or log with enough context to diagnose the failure.
- Never catch `\Throwable` or `\Exception` in business logic. Catch only the specific exception
  type you are expecting.
- The global exception handler in `app/Exceptions/Handler.php` converts domain exceptions to
  HTTP responses. Do not duplicate this mapping in controllers.
```

**Example (Rust / Result types):**
```markdown
## Error Handling

- All fallible functions return `Result<T, E>` where `E` is a typed error enum defined in
  `src/errors/`.
- Use `?` for propagation. Do not `.unwrap()` or `.expect()` in library code or production paths.
  `.unwrap()` is permitted in tests only.
- Error enums must implement `std::error::Error` and `Display`.
- Do not use `anyhow` in library crates. `anyhow` is permitted in binary entry points only.
- Every error variant must carry enough context to diagnose the failure without reading the source.
```

---

### State and Side Effects

**Example:**
```markdown
## State and Side Effects

- Do not introduce mutable global or static state. All state flows through the dependency graph.
- If a function produces a side effect outside its own scope, its name must signal this.
  A getter must not write. A query must not mutate. A method named `get*` or `find*` must be
  pure reads.
- Keep side effects at the edges. Pure logic in the centre, I/O and mutation at the boundary.
- Static methods are permitted only for stateless utilities (pure functions). No static methods
  that read or write shared state.
```

---

### Boundaries

Name the actual layers in this project. Do not write generic layer rules.

**Example (Laravel / layered architecture):**
```markdown
## Boundaries

- Controllers (`app/Http/Controllers/`) handle HTTP only: validate the request, call one Action,
  return the response. No business logic. No database calls.
- Actions (`app/Actions/`) handle business logic only. No HTTP concerns. No direct database
  queries — call a Repository method.
- Repositories (`app/Repositories/`) handle all database queries. No business logic.
  Return domain objects or collections, not raw query builder results.
- Do not reach across service boundaries directly. Cross-service calls go through the HTTP client
  wrapper at `app/Http/Client/` or through a queued event.
- Do not write to another service's database under any circumstances.
```

---

### Tests

**Example:**
```markdown
## Tests

- Write tests against public contracts and observable behaviour. Do not assert on private methods,
  internal state, or mock call counts unless the side effect is the contract.
- Feature tests (`tests/Feature/`) cover HTTP endpoints: send a request, assert on the response
  shape and status code.
- Unit tests (`tests/Unit/`) cover Action classes: provide inputs, assert on outputs and
  exceptions thrown.
- Use factories for all test data. Do not use hand-crafted arrays or trivial placeholder strings.
  Factories must produce data that reflects real-world edge cases.
- New behaviour must have a corresponding test. Do not leave new code untested.
- Tests must pass in under two minutes. If your change makes the suite meaningfully slower,
  flag it.
```

---

### Dead Code

**Example:**
```markdown
## Dead Code

- Do not comment out code. Delete it. Git history preserves it.
- Do not leave unused variables, parameters, imports, or functions in place. Remove them.
- Do not introduce feature flags that evaluate to a constant. If a flag is always on or always
  off, inline the live branch and delete the flag.
- Do not leave `TODO` comments in code without a linked issue. Either fix it now or delete the
  comment.
```

---

### Logging

Name the logger and the required fields.

**Example:**
```markdown
## Logging

- Use the established logger only. Do not use `echo`, `print_r`, `var_dump`, `dd()`, `dump()`,
  or `console.log` for diagnostic output.
- Every log statement must include these structured fields:
  - `service`: always the service name (e.g. `'payments-api'`)
  - `trace_id`: from the current request context
  - `user_id`: when available
- Log levels:
  - `ERROR`: the operation failed and requires investigation
  - `WARN`: the operation succeeded but something unexpected occurred
  - `INFO`: a significant state change occurred (payment processed, order created)
  - `DEBUG`: development-only detail; stripped in production
- Do not log sensitive data: credentials, tokens, card numbers, PII, raw request bodies.
```

---

### Dependencies

**Example:**
```markdown
## Dependencies

- Do not add a new dependency without stopping and flagging it for human approval.
- Do not use API patterns from a version newer than what is declared in `composer.json` /
  `package.json` / `Cargo.toml`.
- If a library's installed version differs from its documented version in `AGENTS.md`, stop
  and flag it before using any of its APIs.
```

---

### Observability

**Example:**
```markdown
## Observability

- Instrument new service entry points using the project's established tracing approach (documented
  in `AGENTS.md`).
- New background jobs, queue consumers, and scheduled tasks must emit a structured log entry on
  start, completion, and failure.
- Do not add observability that duplicates existing instrumentation.
```

---

### Environment and Configuration

**Example:**
```markdown
## Environment and Configuration

- Never hardcode environment-specific values (URLs, credentials, feature switches). Use
  environment variables.
- Only use environment variable names declared in `.env.example`. Do not invent new names
  without adding them to `.env.example` with documentation.
- Read environment variables at the application boundary (service providers, config files,
  entry points). Do not call `env()` or `getenv()` deep in business logic.
```

---

### Escalate — Stop and Ask When

Mirror the escape hatch conditions from `AGENTS.md`. These should be identical.

**Example:**
```markdown
## Escalate — Stop and Ask When

- The change requires adding or removing a dependency
- The change touches `app/Auth/`, `app/Billing/`, or any authentication/authorisation code
- The change modifies a database migration in a destructive way (dropping columns, changing types,
  removing indexes)
- The change affects more than the stated scope
- The correct pattern is genuinely ambiguous and two reasonable interpretations exist
- The required behaviour is not covered by existing tests and the correct behaviour is unclear
- The change modifies a shared package or library
```

---

## Quality Check

Before finishing, verify:

- [ ] Every rule is specific — it names actual classes, directories, or patterns from this project
- [ ] No rule contradicts what is actually done in the codebase
- [ ] The error handling section names the specific approach in use
- [ ] The boundaries section names the actual layers, not generic ones
- [ ] The logging section names the actual logger and required fields
- [ ] The escalation conditions match those in `AGENTS.md`
- [ ] No rule is so broad it provides no guidance (e.g. "write clean code", "use good names")

---

## Escalation Conditions

Stop and ask before proceeding if:

- Conflicting patterns exist in the codebase — two different error handling approaches, two different layer structures — ask which is canonical before writing rules based on either
- The existing `RULES.md` has rules that appear to contradict what is actually in the code — confirm before overwriting
- The project has no tests and no established test style — confirm the intended approach before writing test rules
