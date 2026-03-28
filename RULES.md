# Coding Rules

These rules apply to all code written or modified in this project. Follow them without exception unless an explicit override is documented in `AGENTS.md` or `CLAUDE.md`.

---

## Consistency

- Follow the dominant pattern already present in the codebase. If two patterns exist, ask before proceeding.
- Place every class and file where its name and responsibility predict it should be. Do not create new locations without documenting them.
- Do not introduce a second way of doing something that already has an established approach. Complete or remove the old approach first.

---

## Explicitness

- Never use magic, metaprogramming, dynamic dispatch, or runtime code generation without a doc comment at the call site explaining what it does and why.
- Declare all dependencies explicitly. Prefer constructor injection. Do not rely on ambient resolution or resolve dependencies from the container mid-method.
- All functions and methods should have full type signatures including return types where the language and codebase support them. Avoid untyped arrays, `mixed`, and omitted return types where a more precise type is possible.
- If a function does more than its name says, rename it or split it.

---

## Naming

- Names must communicate intent without requiring the reader to trace the call chain. Prefer long and explicit over short and ambiguous.
- A name must mean one thing across the entire codebase. If a name is already used for a different concept, choose a different name.
- Do not use generic names: `handle`, `process`, `manage`, `data`, `result`, `info`, `manager`, `helper`, `util`.

---

## Error Handling

- Use the project's established error handling approach. Do not introduce a second pattern.
- Never swallow exceptions silently. Every catch block must either rethrow, return a typed error, or log with enough context to diagnose the failure.
- Do not return `null` to signal failure. Use a typed error, a result type, or a typed exception.
- Do not use empty catch blocks under any circumstances.

---

## State and Side Effects

- Do not introduce mutable global or static state.
- If a function produces a side effect outside its own scope, its name must signal this. A getter must not write. A query must not mutate.
- Keep side effects at the edges. Pure logic in the centre, I/O and mutation at the boundary.

---

## Boundaries

- Controllers handle HTTP concerns only. No business logic, no database queries.
- Services handle business logic only. No HTTP concerns, no direct database writes outside a repository or query object.
- Do not reach across service boundaries directly. Use the declared interface — HTTP API, queue message, or shared contract.
- Do not write to another service's database.

---

## Tests

- Write tests against public contracts and observable behaviour, not internal implementation.
- Do not assert on private methods, internal state, or mock call counts unless the side effect is the contract.
- Use realistic fixtures that reflect real-world edge cases. Do not use trivial placeholder data.
- New behaviour must have a corresponding test. Do not leave new code untested.

---

## Dead Code

- Do not comment out code. Delete it. Git history preserves it.
- Do not leave unused variables, parameters, imports, or functions. Delete them.
- Do not introduce feature flags that are always on or always off. Resolve the branch and remove the flag.

---

## Logging

- Use the project's established logger. Do not use `print`, `echo`, `console.log`, `dbg!`, or equivalent for diagnostic output.
- Every log statement must include the core structured fields defined in `AGENTS.md` or `CLAUDE.md`.
- Use the correct log level: `ERROR` for failures requiring investigation, `WARN` for unexpected but handled conditions, `INFO` for significant state changes, `DEBUG` for development-only detail.
- Do not log sensitive data: credentials, tokens, PII, or raw request bodies.

---

## Dependencies

- Do not add a new dependency without stopping and flagging it for human approval.
- Do not use API patterns from a version newer than what is declared in the project manifest.
- If a dependency's installed version differs from its documented version in `AGENTS.md` or `CLAUDE.md`, stop and flag it before proceeding.

---

## Observability

- Instrument new service methods and significant code paths using the project's established tracing approach as documented in `AGENTS.md` or `CLAUDE.md`.
- New background jobs, queue consumers, and scheduled tasks must emit a structured log entry on start, completion, and failure.
- Do not add observability that duplicates existing instrumentation.

---

## Environment and Configuration

- Never hardcode environment-specific values. Use environment variables.
- Use only environment variable names that are declared in `.env.example` or `docs/environment.md`. Do not invent new names without documenting them.
- Do not read environment variables deep in business logic. Read them at the boundary and inject the value.

---

## Escalate — Stop and Ask When

- The change requires adding or removing a dependency.
- The change touches a file or directory marked as restricted in `AGENTS.md` or `CLAUDE.md`.
- The change modifies a database migration in a destructive way (dropping columns, changing types, removing indexes).
- The change affects more than the stated scope.
- The correct pattern is genuinely ambiguous and two reasonable interpretations exist.
- The required behaviour is not covered by existing tests and the correct behaviour is unclear.
- The change modifies a file declared as a shared library or shared package.
