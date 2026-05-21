---
name: "AGENTS.md Author"
description: "Create or update AGENTS.md (or CLAUDE.md) for a project following the AI Readiness Framework. Use when a project is missing its agent orientation file, when AGENTS.md is incomplete or outdated, when adding new services or packages that need coverage, or when an audit has flagged gaps in agent orientation."
---

# AGENTS.md Author

## What This Skill Does

Creates or updates the `AGENTS.md` file (or `CLAUDE.md` — the framework treats them as equivalent) that gives AI agents the orientation they need to work on a project safely and consistently.

A correct `AGENTS.md` answers: what does this project do, how is it structured, what are the rules, where are the boundaries, and when must the agent stop?

---

## Quick Start

1. Read any existing `AGENTS.md`, `CLAUDE.md`, or `README.md` at the project root
2. Read the top-level directory listing
3. Read the package manifest (`package.json`, `composer.json`, `Cargo.toml`, etc.)
4. Draft the file using the structure below
5. Mark anything that cannot be determined with `<!-- TODO: [description] -->`

If the file already exists, compare each required section against the spec below and update only what is missing or inaccurate. Do not rewrite content that is already correct.

---

## Section Set by Tier

The framework treats `AGENTS.md` as tier-progressive: a Tier 0 file is a few paragraphs; a Tier 3 file is a short index pointing into `docs/`. Do not push a Tier 0 project to author Tier 3 content — pick the section set that matches the project's current (or target) tier.

| Section | T0 | T1 | T2 | T3 |
|---|---|---|---|---|
| 1. Project Purpose                    | required | required | required | required (short) |
| 2. Stack                              | required | required | required | summary + link |
| 3. Repository Layout                  | —        | required | required | summary + link |
| 4. Key Conventions                    | —        | required | required | summary + link |
| 5. Running the Project                | —        | required | required | one-line + link to manifest |
| 6. Known Framework Magic (1.2b)       | —        | required | required | link to `docs/conventions/magic.md` |
| 7. Escape Hatches                     | required (restricted areas only, 2.4a) | required (full, 2.4b) | required | required |
| 8. Change Isolation (2.4c)            | —        | required | required | required |
| 9. Out of Scope                       | —        | recommended | recommended | recommended |
| 10. External Dependencies (1.11b)     | —        | —        | required | summary + link |
| 11. Environment Variables (1.11a)     | —        | —        | required (link to `.env.example`) | link only |
| 12. Observability (1.11c, 1.9c)       | —        | —        | required | summary + link |
| 13. MCP Servers (2.2g)                | —        | —        | required | summary + link |
| 14. Language Server (2.3e)            | —        | —        | required | one-line |
| 15. Dependency Currency (1.10a, 1.10b)| —        | —        | required | link only |
| 16. Agent Configuration Owner (2.4e)  | —        | —        | recommended | required |

**T0 file** is typically 20–40 lines: purpose, stack, restricted areas, one runnable verification command. Nothing else.

**T1 file** typically sits at 100–180 lines. Aim under 200 (framework rule 2.2f); if over, split into `docs/conventions/<topic>.md` linked from `AGENTS.md`.

**T2 file** is the natural maximum size. Same 200-line ceiling. Use sub-files aggressively if a section grows past 30 lines.

**T3 file** is a ~100-line index (framework rule 2.2c). Almost every section collapses to a one-sentence summary plus a link into a structured `docs/` tree. The full conventions, MCP inventory, observability guidance, and external-service docs live in `docs/`, and `AGENTS.md` exists to point at them — nothing more.

---

## Required Sections

Each section below shows its content and target tier. Sections marked `(T0)` are the minimum; everything else is added at the tier indicated above. Every included section must have real content — not generic placeholder text.

---

### 1. Project Purpose

**What to write:** 2–3 sentences. What does this system do? Who uses it? What problem does it solve?

**What to avoid:** Do not copy marketing copy. Do not write "this project is a [framework] application". Write what it actually does.

**Example:**
```
This service processes payment transactions for the merchant platform. It validates card details,
communicates with the payment gateway, records the result, and dispatches events that downstream
services use to fulfil orders and send receipts.
```

---

### 2. Stack

**What to write:** Bullet list of every technology an agent needs to know to work on this project.

Include:
- Primary language and version (e.g. PHP 8.3, TypeScript 5.4, Rust 1.78)
- Framework and version
- Test runner and command
- Linter and command
- Key libraries (ORM, HTTP client, queue driver, etc.)
- Runtime environment (Node.js version, container base image)

**Example:**
```
- PHP 8.3 / Laravel 11
- Test: `php artisan test` (PHPUnit)
- Lint: `./vendor/bin/pint`
- ORM: Eloquent
- Queue: Laravel Horizon / Redis
- HTTP client: Guzzle (wrapped in app/Http/Client/)
```

---

### 3. Repository Layout

**What to write:** A directory map explaining what each significant top-level directory owns. Every directory that an agent might touch needs an entry. Include constraint notes where appropriate.

**Example:**
```
app/Actions/        Single-action classes — one public method, one responsibility
app/Http/           Controllers and middleware — HTTP concerns only, no business logic
app/Models/         Eloquent models — attribute definitions, relationships, scopes
app/Services/       Business logic — no HTTP, no direct DB writes
app/Repositories/   All database queries — services call these, never query directly
database/           Migrations and factories
resources/views/    Blade templates
tests/              Mirrors app/ structure; Feature tests for HTTP, Unit tests for services
```

---

### 4. Key Conventions

**What to write:** The specific patterns an agent must follow. These should be derived from what the project actually does, not generic best practice.

Cover:
- How business logic is structured (action classes, service classes, use cases, etc.)
- How errors are handled (exceptions, result types, error codes)
- How dependencies are injected
- How tests are written and organised
- Any naming conventions specific to this project

**Example:**
```
- Business logic lives in Action classes (app/Actions/) — single public method `execute()`,
  accepts a typed DTO, returns a typed result or throws a typed exception
- Errors: throw typed domain exceptions (app/Exceptions/); never return null to signal failure
- All dependencies injected via constructor; never resolve from the container mid-method
- Tests: Feature tests for every HTTP endpoint; Unit tests for every Action
- Models must declare $fillable explicitly; never use $guarded = []
```

---

### 5. Running the Project

**What to write:** The exact commands for the standard operations an agent uses to close its feedback loop.

```
Test:   php artisan test
Lint:   ./vendor/bin/pint --test
Build:  (none required)
Start:  php artisan serve
```

If a Makefile or task runner exists, document the standard targets:
```
make test       Run the fast test suite
make lint       Run the linter
make check      Run lint + test together
```

**If a machine-readable manifest exists (framework rule 2.1d)**, link to it rather than restating its content:

```
The canonical command list and inter-service dependencies are declared in `project.json` at the
repository root. The commands above are mirrored from that file; if the two disagree, the manifest
is authoritative.
```

**If canonical data or API contracts exist (framework rule 2.1e)**, link to them so agents know which file is the source of truth for shapes and endpoints, rather than inferring contracts from controller code:

```
API contracts: `docs/api/openapi.yaml` — the OpenAPI spec is authoritative for request and
response shapes. Database schema: `schema.sql` — kept current with migrations.
```

---

### 6. Known Framework Magic (framework rule 1.2b)

**What to write:** Any non-obvious framework feature that changes how code works at runtime, plus any facade, static accessor, or global container that is exempted from the project's "no ambient dependencies" rule. Framework rule 1.2b requires that facades and static globals which agents are still permitted to use be documented here as known exceptions, with their scope.

This is the single most important section for preventing agents from misunderstanding the codebase.

Include:
- Facades and what they resolve to, plus the scope they may be used in (controllers only? anywhere? service classes never?)
- Auto-wired or auto-discovered classes
- Macros or mixins added to framework classes
- Dynamic method dispatch (`__call`, `__get`)
- Model casts and accessors
- Middleware that modifies requests/responses invisibly
- Any other ambient resolution (e.g. `app()`, `resolve()`, `container.get()`) that is permitted in this codebase, and where

If there is no significant magic, write: `This project does not rely on significant framework magic.` and confirm that no facades, static accessors, or container lookups appear in the codebase.

**Example:**
```
- `Cache::` is a Laravel facade that resolves to the configured cache driver (Redis in production)
- All models auto-cast `created_at` / `updated_at` to Carbon via the base Model
- `app/Providers/MacroServiceProvider.php` adds `Collection::groupByMany()` — see that file for docs
- The `authenticate` middleware attaches the authenticated user to `$request->user()` — controllers
  can rely on this being non-null in any route group using the `auth` middleware
```

---

### 7. Escape Hatches — Stop and Ask When

**What to write:** Specific conditions that require human approval before proceeding. These must be concrete, not vague.

Every project must have at minimum:

```
- The change requires adding or removing a dependency
- The change touches [list the sensitive directories/files specific to this project]
- The change modifies a database migration in a destructive way (dropping columns, changing types, removing indexes)
- The change affects more than the stated scope
- The correct pattern is genuinely ambiguous and two reasonable interpretations exist
- The required behaviour is not covered by existing tests and the correct behaviour is unclear
```

Add project-specific conditions. Examples:
```
- The change touches `app/Billing/` or `app/Auth/` — these are high-risk areas
- The change affects a shared package under `packages/`
- The change modifies an OpenAPI spec file
- The change affects the database schema of a table consumed by another service
```

---

### 8. Change Isolation

**What to write:** How the project manages units of work and what an agent should use as its natural work boundary.

**Example:**
```
- One logical change per branch
- Conventional commits: feat, fix, refactor, test, docs, chore
- PRs must not exceed 400 lines unless the change is purely mechanical (e.g. renaming, formatting)
- Do not mix refactoring and feature work in the same commit
```

---

### 9. Out of Scope

**What to write:** What this project explicitly does not do. This prevents agents from implementing things that belong in another service or a different layer.

**Example:**
```
- This service does not send emails — dispatch events and let the notification service handle delivery
- This service does not render HTML — it is a JSON API only
- Authentication is handled by the auth service — do not implement auth logic here
- Do not add third-party analytics or tracking code
```

---

### 10. External Dependencies

**What to write:** For each external service the project calls, document: what it does, how authentication works, key rate limits or quotas, sandbox vs. production distinction, and any known quirks.

**Example:**
```
### Stripe (payments)
- Purpose: card tokenisation and charge processing
- Auth: secret key in `STRIPE_SECRET_KEY`; use `STRIPE_WEBHOOK_SECRET` to validate webhooks
- Sandbox: set `STRIPE_KEY` to a test key; test card numbers in Stripe docs
- Rate limits: 100 req/s in live mode
- Quirk: idempotency keys required for charge creation — use order UUID

### SendGrid (transactional email)
- Purpose: receipt and notification emails
- Auth: API key in `SENDGRID_API_KEY`
- Note: this service does not call SendGrid directly — it dispatches events; the notification
  service handles delivery
```

---

### 11. Environment Variables

**What to write:** Where environment variables are documented and any that are critical enough to call out explicitly.

If `.env.example` exists: `Environment variables are documented in .env.example.`

If it does not exist: note that it should be created, and list the key variables here.

---

### 12. Observability

**What to write:** What logging, tracing, and metrics tooling is in use, and the minimum an agent must do when writing new code.

**Example:**
```
- Logger: Monolog via Laravel's Log facade — use `Log::info()`, `Log::error()`, etc.
- Required fields on every log statement: `service` (always 'payments-api'), `trace_id` (from
  request header), `user_id` where available
- Do not use `dd()`, `dump()`, `var_dump()`, `echo`, or `print_r`
- New service methods should log on entry (INFO) and on failure (ERROR)
- Metrics: not yet implemented — do not add custom metrics tooling without discussion
```

---

### 13. MCP Servers (framework rule 2.2g)

**What to write:** Every MCP server an agent is expected to have connected when working on this project, with the purpose of each and the install command. If none are required, write `None required` so the absence is intentional rather than missing.

**Example:**
```
- Linear MCP — ticket lookup and status. Install: `claude mcp install linear`
- Sentry MCP — error context and stack traces. Install: `claude mcp install sentry`
- Internal docs MCP — engineering wiki search. Install: see https://wiki.internal/mcp-setup
```

This section is the single source of truth for which agent tooling the project assumes; it should match what `.claude/mcp.json` declares if the project commits one (framework rule 2.2h).

---

### 14. Language Server (framework rule 2.3e)

**What to write:** Which LSP server(s) the project uses, the install command, and whether the project's standard setup installs them automatically. Agents that have an LSP connected do symbol-precise navigation; agents without one fall back to string-based grep.

**Example:**
```
- TypeScript: `tsserver` — installed automatically via `pnpm install`
- PHP: `phpactor` — install with `make setup-lsp`; required before running tests
```

If the project does not currently provide an LSP, mark it explicitly:

```
- No LSP currently configured. <!-- TODO: install phpactor / tsserver / gopls before reaching Tier 2 -->
```

---

### 15. Dependency Currency (framework rules 1.10a, 1.10b)

**What to write:** Any dependency that cannot be updated to its current version, and the reason. Agents will otherwise write code against the current public API documentation and produce calls that do not exist in the installed version.

If every dependency is reasonably current, write: `All major dependencies are within one minor version of current. No pinned-back constraints.`

If any dependency is pinned to an older major version, document each one:

**Example:**
```
- `library-x` pinned to 2.x — 3.x introduces a breaking API change that requires a migration we
  have not scheduled. Do not use the 3.x API patterns; the installed version is 2.7.4.
- `framework-y` pinned to 10.x — upgrading to 11.x is tracked in [LINEAR-1234]; planned for Q3.
  Use the 10.x patterns until then.
- Node runtime pinned to 18 LTS — 20.x has not been validated against our deploy target.
```

This section covers 1.10a (dependencies are reasonably current) by documenting the gap when they are not, and 1.10b (known constraints are documented) directly.

---

### 16. Agent Configuration Owner (framework rule 2.4e)

**What to write:** The named person or team that owns the agent configuration for this project. Required at Tier 3 and recommended at Tier 2 so contributors know whom to contact when configuration breaks.

**Example:**
```
Owner: @platform-dx
Contact: #agent-platform on Slack
Responsible for: quarterly review of `.claude/`, `AGENTS.md`, `RULES.md` (framework rule 2.1k)
```

---

### Cross-references to subdirectory `AGENTS.md` files (framework rule 2.1h)

If the project has hierarchical context files in deep or specialised subdirectories, link to them from the root `AGENTS.md` so agents discover them on first read.

**Example:**
```
## See also (local context files)

- `app/Payments/AGENTS.md` — payment gateway conventions, idempotency rules
- `app/Billing/AGENTS.md` — billing logic, PCI scope notes
- `frontend/AGENTS.md` — frontend conventions, design system usage
```

For monorepos this responsibility is covered by package-level `AGENTS.md` files (framework rule 3.2a) instead.

---

## Monorepo Additions

If the project is a monorepo, also add:

### Monorepo Map

```
services/api        REST API — PHP / Laravel
services/worker     Background jobs — same codebase, different entry point
apps/web            User frontend — TypeScript / Next.js
packages/types      Shared TypeScript types — consumed by web and any TS tooling
```

### Ownership

Which team or person owns each package. Reference `CODEOWNERS` if it exists.

### Cross-Boundary Rules

```
- Do not reach into another service's database directly
- Cross-service calls go through the declared HTTP API or queue interface only
- Stop if your change modifies files in more than one top-level service directory
- Stop if your change modifies any file under packages/ — these are shared
```

### Package-Level Files

Note that each service directory should have its own `AGENTS.md`. This root file covers shared conventions only.

---

## Length Budget

`AGENTS.md` is loaded as orientation context at the start of every agent session. If it exceeds 200 lines the agent will skim or truncate, and detail beyond that point is effectively invisible.

Targets:

- **T0 projects**: 20–40 lines.
- **T1 and T2 projects**: under 200 lines (framework rule 2.2f).
- **T3 projects**: around 100 lines (framework rule 2.2c) — the file is an *index*, not an encyclopedia. This is a hard structural shift, not a softer line count.

### T3 collapse rules

At Tier 3 the file should contain, inline, only:

- Project purpose (1–3 sentences)
- Stack summary (a single bulleted list, no commands)
- Repository layout (a short table)
- Escape hatches (the conditions themselves, in full)
- Change isolation (the conditions themselves, in full)
- Agent configuration owner

Every other section reduces to a one-sentence summary plus a link into `docs/`. If a section at T3 still spans more than three lines inline, it has not collapsed — move the body into a `docs/` page and replace it with the link. Specifically:

- Key Conventions → `docs/conventions/` (one page per topic) or `RULES.md` sub-files
- Known Framework Magic → `docs/conventions/magic.md`
- External Dependencies → `docs/integrations/<service>.md` (one per service)
- Observability → `docs/observability.md`
- MCP Servers → `docs/agents/mcp.md` (or mirror `.claude/mcp.json`)
- Language Server → `docs/agents/lsp.md` or the project's `setup` doc
- Dependency Currency → `docs/dependencies.md`

When working at lower tiers and content would push the file over the 200-line budget, apply the same splits early — the Tier 3 structure does not have to wait until Tier 3 to be useful.

After writing or updating, count lines (`wc -l AGENTS.md`). If over budget, propose splits before finalising.

---

## Quality Check

Before finishing, verify:

- [ ] Every section has real content, not placeholder text
- [ ] The repository layout covers every directory an agent might touch
- [ ] Escape hatches name specific directories and file types, not just vague categories
- [ ] The stack section includes exact commands an agent can run
- [ ] All `<!-- TODO -->` markers are listed in a summary for the user
- [ ] The file is under 200 lines (T1/T2) or under ~100 lines (T3); if longer, content has been moved into linked sub-files

---

## Escalation Conditions

Stop and ask before proceeding if:

- An existing `AGENTS.md` is present and covers the same sections — confirm the update approach before overwriting
- The project spans multiple services and it is unclear whether to write one root file or per-service files
- Sensitive areas (auth, billing, payments) are present but not clearly scoped — confirm escape hatch conditions with the project owner before writing them
