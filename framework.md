# AI Readiness Framework

**Version:** 1.0
**Author:** Christopher Skene
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Purpose

This specification defines the minimum set of practices, files, structures, code quality standards, and harness configuration that a project must have in place before AI coding agents can operate on it with low error rates and minimal human intervention. It covers both single-repository and monorepo layouts.

A project that satisfies this specification gives agents the ability to orient themselves, act within defined boundaries, verify their own output, and escalate correctly when they encounter decisions that require human judgement.

The aim is not to make a project perfect. The aim is to get a project to the point where agents can work on it safely.

This version covers both **codebase readiness** (Parts 1 and 3) and **harness configuration** — the hooks, ignore files, language servers, MCP servers, and committed agent context that determine how an agent meets the codebase at runtime. Harness items are folded into the existing Part 2 capability sections (Orient, Act, Verify, Escalate) rather than collected separately, because each one belongs to one of those four capabilities. A codebase that satisfies Part 1 but ships no harness configuration is still a codebase an agent must reorient itself to on every session.

---

## Fundamentally Hostile Patterns

Some patterns make a codebase agent-hostile in a way that the tiers cannot fix without prior structural work. A project displaying any of these patterns must address them before any tier work begins; no amount of `AGENTS.md` writing or rules tuning will compensate.

- **Millions of files without clear organisation.** An agent that cannot form a mental model of where things live cannot reason about them. If `find . -type f | wc -l` returns more than the team can navigate, the project needs structural triage first.
- **Non-standard version control.** Agents assume git semantics — branches, commits, history, working trees. Projects on Perforce, ClearCase, or other legacy systems will see agent behaviour degrade in ways that are not fixable through configuration.
- **Deeply interconnected cross-directory dependencies with no module boundary.** When any file can import from any other file, there is no boundary for an agent to respect or for a rule to enforce.
- **Generated or binary assets mixed with source code.** When generated code, vendored libraries, and source live in the same directory tree without separation, agents will read generated code as if it were canonical and produce output that follows generator conventions rather than project conventions.
- **Hundreds of thousands of top-level folders.** A flat, high-cardinality root defeats every orientation strategy an agent can apply.

These conditions are signals that a project needs structural triage — module extraction, generated-content separation, migration to standard version control — before agent work is viable at any tier.

---

## Core Model

Every requirement maps to one of four agent capabilities:

- **Orient** - the agent can understand the project without reading source code
- **Act** - the agent can produce consistent, in-convention output
- **Verify** - the agent can close its own feedback loop
- **Escalate** - the agent knows when to stop and surface a decision

---

## Readiness Tiers

This framework is tiered so teams can adopt the controls required for safe agent work first, then add more structure only if they want broader autonomy.

Most projects already at Tier 0 or Tier 1 without knowing it. The investment to reach Tier 2 is real but bounded. Tier 3 is optional and appropriate only for teams that have already validated agent workflows at Tier 2.

Do not skip tiers. The requirements at each level are prerequisites for the ones above. A project that attempts Tier 2 autonomy without satisfying Tier 1 code quality will spend more time fixing agent mistakes than it saves.

---

**Tier 0 — Supervised**

The agent can be pointed at the project without immediately producing harmful or useless output. A human reviews every change before it is accepted.

Requires:
- a minimal `AGENTS.md` stating what the project is and what stack it uses
- restricted directories or files explicitly declared in `AGENTS.md`
- at least one runnable command that produces a clear pass or fail result

Start here. Most projects can reach Tier 0 in an hour.

---

**Tier 1 — Guided**

The agent can orient itself, understand the project's conventions, and make consistent changes within a tightly scoped area. A human still reviews every change, but the agent's output is predictable enough that review is fast.

Requires: everything at Tier 0, plus code quality baseline (consistent patterns, no dead code, no magic, clean error handling), a full `AGENTS.md`, `RULES.md`, and `ARCHITECTURE.md`.

This tier requires the most investment because it demands code quality work that many existing codebases have deferred. Expect days to weeks depending on codebase condition.

---

**Tier 2 — Safe**

The agent can complete normal engineering tasks safely inside defined boundaries, run its own checks, and escalate correctly when it hits a decision requiring human judgement. This is the baseline tier for a project that is fit for purpose for agent work. Human review remains in the loop but is not required on every change.

Requires: everything at Tier 1, plus type coverage, a trustworthy fast test suite, clean layer boundaries, consistent logging and observability, documented environment and external dependencies, and a reproducible environment.

This is the target for most teams. The requirements at this level exist because they directly affect whether agent output can be trusted without line-by-line review.

---

**Tier 3 — Autonomous**

The agent can work across broader areas of the project with less human guidance because the project provides stronger structure, mechanical enforcement, and reusable tooling. Human review shifts from individual changes to system-level outcomes.

Requires: everything at Tier 2, plus CI-enforced architectural invariants, a structured `docs/` knowledge base with mechanical freshness checks, a short `AGENTS.md` that acts as an index, a sub-agent library, and a skill set.

This tier is appropriate for teams that have validated their agent workflows at Tier 2 and want to reduce the human review burden further. The investment is significant and ongoing.

---

Unless otherwise stated, the requirements below are marked with the minimum tier at which they become necessary.

---

## Part 1: Code Quality

Code quality is a prerequisite for everything else in this specification. Documentation, rules, and skills cannot compensate for a codebase that gives agents contradictory or ambiguous signal. These requirements must be satisfied before agents are given autonomy.

The rough test for any given file is: could a competent developer who has never seen the project before read it and understand what it does, why it exists, and what it depends on - without reading five other files first? If yes, agents will handle it. If no, they will not.

### 1.1 Consistency

Agents learn conventions from the code they read. If the codebase has mixed patterns - old and new styles coexisting, different approaches to the same problem in different files - the agent has no reliable signal for what correct looks like and will average across them, producing output that satisfies neither pattern.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.1a | T1 | Single dominant pattern for every common concern | Identify the preferred pattern for each concern and migrate all deviations before running agents; document the chosen pattern in `RULES.md` | One way to define models, one way to handle service errors, one way to shape API responses - not three coexisting approaches |
| 1.1b | T1 | Consistent file and class organisation | Every class lives where an agent would expect to find it based on its name and responsibility; no historical accidents left in place | All payment handlers under `app/Payments/`, not split between `app/Payments/`, `app/Services/`, and `app/Legacy/` |
| 1.1c | T1 | No dual implementations | Incomplete migrations - two HTTP clients, two auth systems, two ways of dispatching jobs - must be completed or the deprecated path deleted; agents will use whichever they encounter first | An old cURL wrapper and a new framework HTTP client coexisting in the same codebase - remove the old one |

### 1.2 Explicitness

Agents handle explicit code well and clever code poorly. Magic - heavy macro use, dynamic method dispatch, metaprogramming, overloaded operators, global state mutation - is hard for agents to reason about because the behaviour is not visible at the call site.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.2a | T1 | No magic that hides behaviour at the call site | Replace or document heavily any metaprogramming, dynamic dispatch, or macro-heavy patterns; where framework magic is unavoidable, add an explanatory doc comment at the call site | Replace `__call` dispatch with explicit method definitions; annotate model casts so agents know what type a field actually is at runtime |
| 1.2b | T1 | No ambient or implicit dependencies | All dependencies declared explicitly - constructor injection preferred; facades and static globals documented in `AGENTS.md` as known exceptions with their scope | A service class that takes `PaymentGatewayClient $client` in its constructor rather than resolving it from the container mid-method |
| 1.2c | T2 | Explicit return types and type hints everywhere | All functions and methods should carry full type signatures including return types where possible; no untyped arrays or `mixed` where a shaped type is possible | `public function store(CreateOrderRequest $request): OrderResource` rather than `public function store($request)` |

### 1.3 Naming

Agents use names to infer intent. Vague names give agents nothing to work with and they will make assumptions that may be wrong. Every class, method, and variable name should communicate what it contains or does without requiring the agent to trace the full call chain.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.3a | T1 | Classes, methods, and variables communicate intent without tracing | Rename anything that requires reading the body to understand what it does; prefer long explicit names over short ambiguous ones | `buildCheckoutPayload()` not `handle()`; `$pendingOrderSync` not `$data` |
| 1.3b | T1 | No overloaded names | A name should mean one thing across the codebase; if the same word is used for different concepts in different contexts, rename one of them | `Order` means the customer order object everywhere - not sometimes a checkout session and sometimes a database row depending on the namespace |
| 1.3c | T1 | No name collisions across paths | Walk the repo grouping files by basename and exported classes or functions by simple name; rename to disambiguate, or document an exception where the collision is load-bearing (e.g. parallel adapters implementing the same port) | Two `helpers.ts` files in unrelated modules renamed to `format-helpers.ts` and `auth-helpers.ts`; two classes named `Service` renamed to `OrderService` and `PaymentService` |
| 1.3d | T1 | No empty generic names in new code | Names like `Service`, `Helper`, `Manager`, `Handler`, `Utils`, `Data` used unqualified carry no intent; require a domain prefix or suffix in `RULES.md` and check it on every change | `OrderService` not `Service`; `CurrencyFormatter` not `Helper`; `WebhookDispatcher` not `Manager` |

### 1.4 Tests

Agents need a feedback loop they can close. A test suite that passes must be genuine signal that behaviour is correct. Tests that are coupled to implementation details, use trivial fixtures, or take too long to run give agents false confidence or no signal at all.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.4a | T2 | Tests cover public contracts and behaviour, not implementation | Delete or rewrite tests that assert on internal state or are coupled to implementation details; focus coverage on inputs and outputs at service and API boundaries | A test that calls a checkout endpoint and asserts the response shape is correct - not a test that asserts a private formatter method was called with specific arguments |
| 1.4b | T2 | Tests are fast enough for an agent to run in a feedback loop | A default verification run should complete in under two minutes; slower suites should be tagged and excluded from the default run | `make test` runs the fast suite in under two minutes; `make test-full` runs everything including integration tests |
| 1.4c | T2 | Tests use realistic data, not trivial fixtures | Fixtures and factories should produce data that reflects real-world edge cases; agents will write code that handles what the tests model | An order factory that produces orders with missing optional fields and mixed tax rules, because that is what real data looks like |

### 1.5 Dead Code

Dead code is actively harmful to agents. They will read unused classes, commented-out blocks, and permanently resolved feature flags as valid signal and either try to use them, try to reconcile them with the live code, or produce output that is consistent with the dead path rather than the live one.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.5a | T1 | No unused classes, methods, or variables | Run static analysis to identify dead code and delete it; do not comment it out | `cargo +nightly udeps` for Rust, PHPStan for PHP; anything flagged and unused is deleted not commented |
| 1.5b | T1 | No commented-out code blocks | All commented-out code is deleted; if it needs to be recoverable it is in git history | A block of old webhook handling code commented out during a refactor - delete it, the commit history has it |
| 1.5c | T2 | No permanently resolved feature flags | Feature flags that are always on or always off must be resolved and removed; the branching code should be collapsed to the live path | A flag introduced for a rollout two years ago that is now always true - inline the true branch and delete the flag and its false branch entirely |

### 1.6 Boundaries

If business logic leaks into controllers, or database queries appear in views, or service classes instantiate their own dependencies, agents will replicate those patterns. They need clean, consistent layers to work within.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.6a | T2 | Business logic does not leak across layers | Controllers handle HTTP concerns only; services handle business logic; repositories or query objects handle data access | A controller that calls `$order->save()` directly should be refactored to delegate to a service; agents will otherwise write new controllers that do the same |
| 1.6b | T2 | Cross-service communication goes through defined interfaces | No direct database sharing between services; no reaching into another service's internals; all cross-boundary calls go through a declared interface or contract | A billing service communicating with a notifications service via an internal HTTP API or queue, not by writing directly to the notifications database |
| 1.6c | T2 | Declared architectural pattern is enforced | The pattern named in `ARCHITECTURE.md` (see 2.1g) has its dependency-direction rules validated by a tool or test; the rule cannot be a comment that the agent ignores | `deptrac` for PHP, `dependency-cruiser` for JS, `archunit` for Java, or an import-graph test in Rust; lint error message names the rule that was broken |

### 1.7 Error Handling

Code that swallows exceptions, returns null ambiguously, or uses error codes inconsistently makes it impossible for an agent to reason about the failure surface. Agents will write new code that reproduces whatever pattern they find.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.7a | T1 | Consistent, explicit error handling throughout | Choose one error handling approach per language and apply it everywhere; no silent swallowing of exceptions; no ambiguous null returns | Rust: propagate with `?` and typed error enums throughout; PHP: throw typed exceptions, catch at the boundary, never `catch (\Throwable $e) {}` with an empty body |
| 1.7b | T1 | No silent failures | Every error path must produce either a typed error, a logged event, or a visible exception; code that suppresses errors gives agents no signal about the failure surface | Replace `try { ... } catch (Exception $e) { return null; }` with a typed exception that the caller handles explicitly |

### 1.8 State

Global variables, static singletons with mutable state, and ambient context make it hard for agents to understand what a piece of code depends on. They will write code that appears correct in isolation but breaks when the ambient state is different from what they assumed.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.8a | T2 | No mutable global or static state | Convert static singletons and global variables to injectable dependencies; where truly unavoidable, document in `AGENTS.md` | A static `Registry::$instances` array replaced with a dependency-injected registry passed through the call chain |
| 1.8b | T2 | Side effects are localised and declared | A function that modifies state outside its own scope should either be renamed to signal it, refactored to return the change, or documented explicitly | A method named `getOrder()` that also updates a last-accessed timestamp should be renamed `touchAndGetOrder()` or split into two methods |

### 1.9 Logging

Agents write log statements as part of new code. Without a consistent logging approach they will produce log output that is missing, duplicated, at the wrong level, or in a different format to the rest of the codebase.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.9a | T2 | Consistent log levels with defined semantics | Document what each log level means in the context of this project and enforce it in `RULES.md` | `ERROR` means the request failed and a human should investigate; `WARN` means the request succeeded but something unexpected occurred; `DEBUG` is stripped in production |
| 1.9b | T2 | Consistent structured fields | All log statements emit the same core fields in the same format; agents add these fields to any new log statement they write | Every log line includes `service`, `trace_id`, `user_id` where available, and `duration_ms` for timed operations |
| 1.9c | T2 | Log transport documented | The logging destination and any log aggregation tooling is documented in `AGENTS.md` so agents know where output goes and how to query it | "Logs are shipped to the project's central log pipeline via the structured JSON logger; do not use `echo` or `print_r` for diagnostic output" |

### 1.10 Dependency Currency

Agents write code against current API documentation, which may not match what is actually installed. Severely outdated dependencies cause agents to use patterns, method signatures, or behaviours that do not exist in the installed version.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.10a | T2 | Dependencies are reasonably current | Run `composer outdated` or `cargo outdated` and update major dependencies before running agents; patch and minor updates should be kept current as a matter of routine | A framework installation three major versions behind means agents will suggest patterns that do not exist |
| 1.10b | T2 | Known version constraints are documented | Where a dependency cannot be updated, document the constraint and the reason in `AGENTS.md` so agents do not attempt to use newer API patterns | "We are pinned to `library-x` 2.x because 3.x has a breaking API change that requires a migration; do not use the 3.x API" |

### 1.11 Operational Context

Agents that lack operational context will write code that is locally correct but operationally incomplete - missing observability hooks, using wrong environment variable names, or making incorrect assumptions about external service behaviour.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.11a | T2 | Environment variables documented | A committed `.env.example` or `docs/environment.md` listing every variable, its purpose, its format, and whether it is required or optional | `PAYMENTS_API_BASE_URL` - base URL for the payments provider; required in production, defaults to `http://localhost:8080` in development |
| 1.11b | T2 | External service dependencies documented | Third-party APIs, managed services, and external integrations documented with their authentication approach, rate limits, known quirks, and sandbox/production distinction | "The shipping API requires an API key header in production, has a sandbox environment, and enforces a rate limit of 10 requests per second" |
| 1.11c | T2 | Observability infrastructure documented | Tracing, metrics, and structured logging tooling documented in `AGENTS.md` with guidance on when to instrument new code; where an observability platform is reachable through MCP, declare it in the MCP inventory at 2.2g | "All new service methods should emit a trace span; use the project's tracing helper in Rust or the `Tracer` service in PHP; Sentry MCP is connected for error lookup, see 2.2g" |

### 1.12 File Length

Agents read files into a working window before they can act on them. A file that exceeds that window forces the agent to summarise or chunk, both of which lose context and break edit precision. The effective working window degrades well before the model's hard context limit. Predictable file sizes are an agent-readiness concern, not a style preference.

This applies to both source files and to the orientation documents (`AGENTS.md`, `RULES.md`, `ARCHITECTURE.md`); the latter are covered by 2.2f.

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 1.12a | T1 | Declared per-language source-file size budget | `RULES.md` declares a soft line budget per file category; files exceeding the budget are split or justified inline with a comment naming the reason | "500 lines general source; 300 lines for UI components; 800 lines for generated migrations or fixtures"; a 1200-line controller is split into per-concern controllers or refactored to delegate |
| 1.12b | T2 | Hard ceiling on source-file size | A pre-commit or CI check fails any source file above the harness-readable ceiling (suggested 1500 lines, configurable in `RULES.md`); generated files, lockfiles, and snapshots may be excluded with an explicit allowlist | A check that runs `find . -name '*.ts' | xargs wc -l | awk '$1 > 1500'` and fails CI if anything is returned; allowlisted paths declared in `.file-length-ignore` or the CI config |

---

## Part 2: Project Structure

### 2.1 Orient

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 2.1a | T0 | Project identity | `AGENTS.md` or `CLAUDE.md` exists and states at minimum what the project does, what stack it uses, and which areas are off-limits; without this an agent has no basis for any decision it makes | A two-paragraph `AGENTS.md` naming the project, its language and framework, and listing three or four directories the agent must not touch |
| 2.1b | T1 | Architecture overview | `ARCHITECTURE.md` at the root covering all services, communication patterns, infrastructure layout, and a diagram of service boundaries and data flows; updated in any PR that changes a service boundary | Mermaid diagram showing `web`, `api`, `worker`, and `database` boundaries with queue and HTTP connections marked |
| 2.1c | T2 | Structured documentation schema | All docs follow an established directory schema; a top-level index lets agents locate documents without reading every file | A `docs/` tree with stable sections and an index page linking architecture, operations, and API docs |
| 2.1d | T2 | Machine-readable project manifest | `project.json` or `manifest.toml` at the root declaring stack, entry points, test commands, lint commands, and inter-service dependencies | `{ "stack": ["typescript", "php"], "test": "pnpm test && php artisan test", "lint": "pnpm lint && ./vendor/bin/pint" }` |
| 2.1e | T2 | Canonical data and API contracts | A committed OpenAPI spec, schema file, or shared type layer that is authoritative; manifest and `AGENTS.md` both reference it explicitly | OpenAPI spec at `docs/api/openapi.yaml`; `schema.sql` committed and kept current with migrations |
| 2.1f | T3 | Documentation as system of record | `docs/` is the authoritative knowledge base for everything that influences agent behaviour; CI validates structure and cross-links; a recurring task scans for drift and opens fix-up PRs; knowledge that exists only in chat, external tools, or people's heads is invisible to agents and must be encoded into the repository | A background agent that checks documented behaviours against actual code weekly and opens correction PRs for any content it finds to be stale or inaccurate |
| 2.1g | T1 | Named architectural pattern | `ARCHITECTURE.md` names the pattern in use from a recommended shortlist (or declares "project-specific" with rationale), lists the layers or ports, and states the dependency-direction rule that 1.6c will enforce; pattern choice should be drawn from the ranked shortlist below | "Ports-and-adapters. Domain core in `src/domain/`, primary adapters in `src/api/`, secondary adapters in `src/infra/`. Domain has no imports from `api/` or `infra/`." |
| 2.1h | T2 | Hierarchical context files | Subdirectory `AGENTS.md` or `CLAUDE.md` files cover local conventions for deep or specialised areas of the codebase; agents load them additively as they enter those subtrees, keeping the root file focused on the big picture; for monorepos see also 3.2a | `app/Payments/AGENTS.md` describes payment-specific conventions, gateway abstractions, and known caveats; the agent reads it when working in that subtree without polluting the root file |
| 2.1i | T1 | Agent-visibility ignore file | A `.claudeignore`, `.agentignore`, or equivalent declares paths the agent should not load - generated code, lockfiles, binary assets, vendored dependencies; without this, agents waste context on noise and may treat generated artefacts as canonical source | `.claudeignore` excludes `dist/`, `build/`, `*.lock`, `vendor/`, `node_modules/`, and any path matching `**/__generated__/**` |
| 2.1j | T2 | Codebase map | Where directory structure does not self-describe, a `docs/codebase-map.md` or equivalent provides a one-page guide from concept to filesystem location; distinct from `ARCHITECTURE.md`, which describes the system - the map describes where things live | A table listing each domain concept (orders, billing, notifications) and the directories that own it; updated whenever a new top-level concern is added |
| 2.1k | T3 | Configuration freshness review | A documented cadence (every three to six months) for reviewing and pruning the agent configuration; constraints that compensated for past model limitations are removed, skills and sub-agents that are no longer used are archived, and obsolete rules are deleted | A quarterly review checklist that lists every file in `.claude/`, `RULES.md`, and `AGENTS.md` and asks whether each entry is still earning its keep against the current model |

**Recommended pattern shortlist**, in approximate order of agent-friendliness (strongest boundaries first). Projects pick by fit, not by ranking:

1. **Hexagonal / ports-and-adapters** — strongest boundary enforcement. Domain core has no infrastructure imports; adapters depend on domain. Preferred for service code with non-trivial business logic.
2. **Clean architecture** — formalised hexagonal with explicit use-case layer. Good for larger codebases where the domain has many entry points.
3. **Layered (n-tier)** — explicit top-to-bottom direction (controller → service → repository). Simplest with strong direction; default for most service code.
4. **Vertical slice** — feature-isolated modules, each containing its own thin layers. Good for monoliths with feature teams; reduces cross-feature blast radius.
5. **MVC** — acceptable for UI-driven web apps where the controller-view boundary is the primary concern.
6. **Project-specific** — allowed but must be diagrammed and have an explicit dependency-direction rule, otherwise it is indistinguishable from "no pattern".

### 2.2 Act

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 2.2a | T1 | `AGENTS.md` / `CLAUDE.md` | Root-level file covering project purpose, stack, conventions, directory ownership, and what the project does not do; treated as a living document updated with every structural change | Covers which directories own which concerns, known framework magic exceptions, links to `RULES.md` and the manifest, escape hatch conditions |
| 2.2b | T1 | Coding rules | `RULES.md` or inline in `AGENTS.md`; specific enough that two agents produce consistent output independently | "Never unwrap in library code", "all models must declare assignable fields explicitly", "all public functions must have doc comments" |
| 2.2c | T3 | `AGENTS.md` as index, not encyclopedia | `AGENTS.md` is kept short (around 100 lines) and acts as a table of contents pointing into `docs/`; all convention detail, operational context, and external dependency information lives in the structured `docs/` tree; a monolithic `AGENTS.md` crowds out task context, makes everything equally important, and rots because it cannot be mechanically validated | `AGENTS.md` contains project purpose, stack summary, directory layout, escape hatch conditions, and links to deeper documents — nothing else |
| 2.2d | T3 | Sub-agent library | `agents/` or `.claude/agents/` directory with one file per sub-agent, each with a single well-scoped responsibility; prefer pairs that split exploration from editing so an explorer sub-agent can return findings to a main agent without consuming its working context | `migration-writer.md`, `test-generator.md`, `openapi-updater.md`; a `repo-explorer.md` that maps an unfamiliar subsystem and returns a summary, leaving the main agent's context free to make the edit |
| 2.2e | T3 | Skill set | `skills/` directory with runnable or instructable skill files covering common tasks; centralised across projects where possible | Release skill that bumps versions across project manifests, tags, and produces a changelog |
| 2.2f | T1 | Context-file compactness | `AGENTS.md`, `RULES.md`, and `ARCHITECTURE.md` each stay below 200 lines; when content exceeds that, the main file becomes an index referencing modular sub-files; an agent that has to scroll past 200 lines to find what it needs will skim or truncate | `RULES.md` splits into `rules/php.md`, `rules/typescript.md`, `rules/naming.md`, with the root `RULES.md` as a one-page index; the T3 rule 2.2c then tightens `AGENTS.md` further to around 100 lines |
| 2.2g | T2 | MCP server inventory | `AGENTS.md` lists every MCP server expected to be connected for normal agent work on this project - issue tracker, internal API gateway, observability platform, documentation system - with the purpose of each and installation instructions; without this inventory, agents that depend on internal context will work blind or fabricate | "Required MCP servers: Linear MCP (ticket lookup), Sentry MCP (error context), internal docs MCP (search engineering wiki); install via `claude mcp install ...`" |
| 2.2h | T3 | Harness configuration as artefact | A `.claude/` or `.agents/` directory is committed to the repository declaring the expected hooks, sub-agents, skills, and MCP server configuration as code; the configuration travels with the project rather than depending on each developer's local setup | `.claude/hooks.json`, `.claude/agents/`, `.claude/skills/`, and `.claude/mcp.json` all committed and version-controlled; a new contributor running `claude` in the repo gets the same configuration as everyone else |
| 2.2i | T3 | Reflective configuration updates | A Stop hook or equivalent reviews each agent session and proposes updates to `AGENTS.md` or `RULES.md` where the session revealed a missing convention; the agent surfaces the proposal for human approval and does not silently rewrite its own rules | A Stop hook that scans the session for repeated user corrections on the same topic and opens a PR adding the relevant rule to `RULES.md`, tagged for the configuration owner to review |
| 2.2j | T3 | Plugin / distribution layer (optional) | Where multiple projects share a common harness configuration, the configuration is packaged as a plugin or installable bundle so it can be distributed across repositories without tribal-knowledge drift; marked optional because not every organisation operates at this scale | An internal plugin published to a managed marketplace, installed by every project in the organisation, providing the shared sub-agents, skills, and hooks; project-specific configuration extends the plugin rather than replacing it |

### 2.3 Verify

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 2.3a | T0 | At least one runnable verification command | A single command exists that the agent can run to confirm it has not broken the build or test suite; without any feedback loop the agent cannot know whether its output is correct | `npm test`, `php artisan test`, or `cargo test` runs and produces a clear pass or fail result |
| 2.3b | T2 | Reproducible environment | `Dockerfile`, `.devcontainer/`, or a `Makefile` with standard targets the agent can run without knowing the host environment | `make test`, `make lint`, `make build` - agent runs these and treats the result as ground truth |
| 2.3c | T2 | Trusted test surface | Tests covering critical system contracts; a green result must be genuine signal that behaviour is correct | Checkout response shape tests rather than unit tests on internal formatting helpers |
| 2.3d | T3 | Architectural invariants enforced by CI | Architectural rules from `RULES.md` are encoded as custom linters or structural tests in CI; a rule that exists only in a document can be ignored by an agent that has not read it or has been given conflicting signal from the code; lint error messages include remediation instructions so an agent that triggers a check is told how to fix it, not just that it failed | A custom lint enforcing layer dependency direction with an error message reading "move this logic into a service — see docs/DESIGN.md#layers"; a structural test asserting no file in `app/Http/` contains a direct database query |
| 2.3e | T2 | Language Server Protocol available | An LSP server for each primary language is documented and installable; setup instructions ensure agents can use symbol-precise navigation rather than string-based grep; critical for languages where symbol names collide across files (C, C++, Java) and useful for any codebase past a few hundred files | `AGENTS.md` declares "this project uses `gopls` / `tsserver` / `phpactor`; install via `make setup-lsp`"; agents searching for a function by name follow the call to its definition rather than matching every occurrence of the string |
| 2.3f | T3 | Deterministic enforcement hooks | Linting, formatting, and structural checks run as hooks (pre-commit, pre-edit, pre-write) so the agent cannot ship code that violates them; complements 2.3d by enforcing at the point of edit rather than at PR time, closing the feedback loop tighter | A pre-write hook runs `eslint --fix` and `phpstan analyse` on every file the agent touches; failures block the write and surface the error message back to the agent for immediate correction |

### 2.4 Escalate

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 2.4a | T0 | Restricted areas declared | `AGENTS.md` lists the directories or files the agent must not modify under any circumstances; without this an agent has no basis for knowing where it should not go | `AGENTS.md` states "do not modify anything under `auth/`, `billing/`, or `database/migrations/`" |
| 2.4b | T1 | Defined escape hatches | A section in `AGENTS.md` listing specific conditions that require human approval before the agent proceeds | "Stop if a migration drops a column", "stop if a new dependency is required", "stop if the change touches `auth/` or `billing/`" |
| 2.4c | T1 | Change isolation strategy | Documented conventions in `AGENTS.md` or `CONTRIBUTING.md` giving the agent a natural unit-of-work boundary | "One logical change per branch", "conventional commits", "PRs must not exceed 400 lines unless mechanical" |
| 2.4d | T3 | Agent output review standard | A section in `CONTRIBUTING.md` defining what reviewers should check specifically when reviewing agent-authored changes, distinct from human-authored changes | "Agent PRs must include a summary of what the agent was asked to do; reviewer must verify no files outside the stated scope were modified" |
| 2.4e | T3 | Organisational ownership | A named person or team owns the project's agent configuration as a DRI; documented in `AGENTS.md` so contributors know whom to contact when configuration breaks or evolves, and so the freshness review at 2.1k has an accountable owner | `AGENTS.md` states "Agent configuration owner: @platform-dx team; questions to #agent-platform on Slack"; the owner is responsible for the quarterly freshness review |

---

## Part 3: Monorepo Additions

Projects using a monorepo layout must satisfy everything in Parts 1 and 2 and additionally satisfy the following.

### 3.1 Additional Orient Requirements

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 3.1a | T1 | Monorepo map | A root-level manifest section listing every package and service with its path, language, and role | `{ "services": { "web": { "path": "apps/web", "lang": "typescript" }, "api": { "path": "services/api", "lang": "php" } } }` |
| 3.1b | T1 | Explicit ownership | Each package or service directory declares what it owns and what it depends on; agents must not reach across undeclared boundaries | `CODEOWNERS` file; or a `package.json` / `Cargo.toml` workspace member with an explicit description field |
| 3.1c | T2 | Inter-service communication documented | `ARCHITECTURE.md` must explicitly cover all cross-service communication - shared queues, internal APIs, shared databases - so agents do not treat services as more isolated than they are | A section listing every queue topic, which service publishes, and which services consume |
| 3.1d | T2 | Build order and dependency graph | If services must be built in a specific order due to inter-package dependencies, this must be declared; agents running builds or CI tasks need to know the correct order and which packages are downstream of a change | A `BUILD_ORDER.md` or a `Makefile` that encodes the dependency graph explicitly; or Turborepo / Cargo workspace dependency declarations |

### 3.2 Additional Act Requirements

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 3.2a | T1 | Package-level `AGENTS.md` | Each service or package has its own `AGENTS.md` that inherits from the root but overrides anything specific to that context; this is the monorepo manifestation of the hierarchical context pattern at 2.1h, made mandatory because monorepo packages have distinct conventions that single-repo subdirectories may not | `services/api/AGENTS.md` covering API conventions and which packages are in scope |
| 3.2b | T1 | Package-level coding rules | Rules that apply only within a given package live close to that package, not in the root | Backend-specific rules in `services/api/RULES.md`; frontend rules in `apps/web/RULES.md` |
| 3.2c | T3 | Scoped sub-agents | Shared sub-agents live at the root; package-specific sub-agents live inside the package | A `migration-writer.md` sub-agent inside `services/api/agents/` that knows the service schema |
| 3.2d | T3 | Shared library governance | Packages that multiple services depend on must be declared as shared; the manifest or a dedicated document should declare who owns them and what the change process is; agents must never modify a shared library without flagging it as a cross-boundary change | A `SHARED.md` listing shared packages, their owners, and the requirement that any change to a shared package must be reviewed by all consuming service owners |

### 3.3 Additional Verify Requirements

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 3.3a | T2 | Scoped test and lint commands | The manifest must declare commands at both root level and package level; agents working on one service should not need to run the full suite | Root: `make test` runs all; `make test service=api` runs only the API service; or use Cargo workspaces / Turborepo |

### 3.4 Additional Escalate Requirements

| ID | Tier | Requirement | How to satisfy | Example |
|---|---|---|---|---|
| 3.4a | T1 | Cross-boundary change detection | Escape hatch rules must explicitly cover changes that touch more than one package - these are higher risk and always require human review | "Stop if your change modifies files in more than one top-level service directory" |
| 3.4b | T1 | Shared library change escalation | Any change to a package declared as shared in the manifest must be escalated regardless of how small it appears | "Stop if your change modifies any file under `packages/shared/` or `packages/types/`" |

---

## Appendix: Readiness Checklist

A project reaches each tier when all items for that tier are true. A project is fit for purpose for safe agent work at Tier 2.

### Tier 0 - Supervised baseline

- [ ] **2.1a** `AGENTS.md` or `CLAUDE.md` exists and states what the project does, what stack it uses, and which areas are off-limits
- [ ] **2.3a** At least one command exists that the agent can run to verify it has not broken the build or test suite
- [ ] **2.4a** Restricted directories or files are explicitly declared in `AGENTS.md`

### Tier 1 - Guided baseline

- [ ] **1.1a** A single dominant pattern exists for every common concern and deviations have been migrated
- [ ] **1.1b** File and class organisation is consistent and predictable
- [ ] **1.1c** No incomplete migrations - no dual implementations of the same concern
- [ ] **1.2a** No metaprogramming or magic that hides behaviour without documentation at the call site
- [ ] **1.2b** All dependencies are explicitly declared; no ambient resolution
- [ ] **1.3a** All names communicate intent without requiring the reader to trace the call chain
- [ ] **1.3b** No overloaded names used for different concepts in different contexts
- [ ] **1.3c** No duplicate file basenames or class/function simple names across unrelated paths, except where documented
- [ ] **1.3d** No empty generic names (`Service`, `Helper`, `Manager`, `Handler`, `Utils`, `Data`) used unqualified in new code
- [ ] **1.5a** No unused classes, methods, or variables
- [ ] **1.5b** No commented-out code blocks
- [ ] **1.7a** A single error handling approach is applied consistently throughout
- [ ] **1.7b** No silent failure paths
- [ ] **1.12a** `RULES.md` declares per-language source-file size budgets and oversized files are split or justified
- [ ] **2.1b** `ARCHITECTURE.md` exists, is current, and includes a diagram
- [ ] **2.1g** `ARCHITECTURE.md` names an architectural pattern from the shortlist (or declares "project-specific" with rationale and dependency-direction rule)
- [ ] **2.1i** A `.claudeignore` / `.agentignore` (or equivalent) is committed and excludes generated, binary, and vendored content
- [ ] **2.2a** `AGENTS.md` or `CLAUDE.md` exists at the root, is current, and covers all required sections
- [ ] **2.2b** `RULES.md` exists and is specific enough to produce consistent output across agents
- [ ] **2.2f** Each of `AGENTS.md`, `RULES.md`, `ARCHITECTURE.md` is under 200 lines or split into modular sub-files
- [ ] **2.4b** Escape hatches are defined in `AGENTS.md` with specific triggering conditions
- [ ] **2.4c** Change isolation conventions are documented

### Tier 2 - Safe baseline

- [ ] **1.2c** All functions and methods carry full type signatures where possible
- [ ] **1.4a** Tests cover public contracts and behaviour, not implementation details
- [ ] **1.4b** Default verification run completes fast enough to be used as an agent feedback loop
- [ ] **1.4c** Test fixtures reflect realistic edge-case data
- [ ] **1.5c** No permanently resolved feature flags
- [ ] **1.6a** Business logic does not leak across layers
- [ ] **1.6b** Cross-service communication goes through declared interfaces only
- [ ] **1.6c** The declared architectural pattern's dependency-direction rule is enforced by a tool or test
- [ ] **1.8a** No mutable global or static state
- [ ] **1.8b** All side effects are localised and signalled by name
- [ ] **1.9a** Log levels have defined semantics documented in `RULES.md`
- [ ] **1.9b** All log statements emit consistent structured fields
- [ ] **1.9c** Log transport and destination documented in `AGENTS.md`
- [ ] **1.10a** All major dependencies are reasonably current
- [ ] **1.10b** Known version constraints are documented in `AGENTS.md` with reasons
- [ ] **1.11a** All environment variables documented in `.env.example` or `docs/environment.md`
- [ ] **1.11b** All external service dependencies documented with auth, rate limits, and quirks
- [ ] **1.11c** Observability infrastructure documented with guidance on instrumenting new code
- [ ] **1.12b** A pre-commit or CI check fails any source file above the declared hard ceiling (default 1500 lines), with explicit allowlist for generated content
- [ ] **2.1c** Documentation follows an established schema with a top-level index
- [ ] **2.1d** A machine-readable project manifest exists at the root
- [ ] **2.1e** Canonical data and API contracts are committed and referenced from the manifest and `AGENTS.md`
- [ ] **2.1h** Deep or specialised subdirectories carry their own `AGENTS.md` / `CLAUDE.md` files for local conventions
- [ ] **2.1j** A `docs/codebase-map.md` (or equivalent) exists where directory structure does not self-describe
- [ ] **2.2g** `AGENTS.md` lists every MCP server expected for normal agent work with its purpose and install instructions
- [ ] **2.3b** A reproducible environment exists with standard runnable targets
- [ ] **2.3c** Test surface covers critical contracts and a green result is genuine signal
- [ ] **2.3e** An LSP server for each primary language is documented and installable

### Tier 3 - Higher autonomy

- [ ] **2.1f** `docs/` is treated as the system of record; documentation freshness is mechanically enforced via CI and a recurring cleanup task
- [ ] **2.1k** A documented cadence (3-6 months) exists for reviewing and pruning agent configuration
- [ ] **2.2c** `AGENTS.md` is short (around 100 lines), acts as an index, and points to a structured `docs/` knowledge base rather than inlining all conventions and context
- [ ] **2.2d** Sub-agent library exists with single-responsibility files, including at least one explorer/editor split
- [ ] **2.2e** Skill set exists covering at minimum the release workflow
- [ ] **2.2h** A committed `.claude/` or `.agents/` directory declares hooks, sub-agents, skills, and MCP configuration as code
- [ ] **2.2i** A Stop hook (or equivalent) proposes `AGENTS.md` / `RULES.md` updates from session reflections for human approval
- [ ] **2.2j** Where shared across projects, harness configuration is packaged as a plugin or installable bundle (optional)
- [ ] **2.3d** Architectural invariants from `RULES.md` are encoded as CI linters or structural tests with agent-readable error messages that include remediation instructions
- [ ] **2.3f** Lint, format, and structural checks run as pre-edit / pre-commit hooks that block non-conforming writes
- [ ] **2.4d** Agent output review standard is documented in `CONTRIBUTING.md`
- [ ] **2.4e** A named person or team is documented in `AGENTS.md` as the DRI for agent configuration

### Tier 1 - Monorepo additions

- [ ] **3.1a** Root manifest includes a monorepo map of all packages and services
- [ ] **3.1b** Ownership is declared per package
- [ ] **3.2a** Each package has its own `AGENTS.md`
- [ ] **3.2b** Each package has its own scoped coding rules
- [ ] **3.4a** Cross-boundary change escalation rules are defined
- [ ] **3.4b** Shared library change escalation rules are defined

### Tier 2 - Monorepo additions

- [ ] **3.1c** Inter-service communication is fully documented in `ARCHITECTURE.md`
- [ ] **3.1d** Build order and dependency graph is declared
- [ ] **3.3a** Test and lint commands are scoped per package

### Tier 3 - Monorepo additions

- [ ] **3.2c** Package-specific sub-agents exist inside the package directory
- [ ] **3.2d** Shared libraries are declared and their governance documented
