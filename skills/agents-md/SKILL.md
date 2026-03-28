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

## Required Sections

Every `AGENTS.md` must contain all of the following sections. Each section must have real content — not generic placeholder text.

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

---

### 6. Known Framework Magic

**What to write:** Any non-obvious framework feature that changes how code works at runtime. Magic that is invisible at the call site must be explained here.

This is the single most important section for preventing agents from misunderstanding the codebase.

Include:
- Facades and what they resolve to
- Auto-wired or auto-discovered classes
- Macros or mixins added to framework classes
- Dynamic method dispatch (`__call`, `__get`)
- Model casts and accessors
- Middleware that modifies requests/responses invisibly

If there is no significant magic, write: `This project does not rely on significant framework magic.`

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

## Quality Check

Before finishing, verify:

- [ ] Every section has real content, not placeholder text
- [ ] The repository layout covers every directory an agent might touch
- [ ] Escape hatches name specific directories and file types, not just vague categories
- [ ] The stack section includes exact commands an agent can run
- [ ] All `<!-- TODO -->` markers are listed in a summary for the user

---

## Escalation Conditions

Stop and ask before proceeding if:

- An existing `AGENTS.md` is present and covers the same sections — confirm the update approach before overwriting
- The project spans multiple services and it is unclear whether to write one root file or per-service files
- Sensitive areas (auth, billing, payments) are present but not clearly scoped — confirm escape hatch conditions with the project owner before writing them
