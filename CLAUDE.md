# AI Readiness Framework — Project Instructions

This document defines how AI coding agents should interact with the **ai-readiness-framework** project.

---

## Project Overview

The AI Readiness Framework is a specification and skill library for making codebases safe for AI agent work. It defines a four-tier maturity model (Supervised, Guided, Safe, Autonomous) that codebases can implement to enable reliable autonomous code generation.

**Primary components:**

- `framework.md` — The specification document defining tier requirements, practices, and implementation guidance
- `RULES.md` — Coding rules for agents working on this repository
- `skills/` — Ten agent skills for implementing the framework in other projects
- `AGENTS.md` — Agent workflow and problem-solving process documentation
- `skill.json` / `opencode.json` — Skill package manifests for distribution

**Tech stack:**

- Pure documentation (Markdown)
- No build tooling or runtime dependencies
- Distributed as agent skills via Automatic MCP service

---

## Build & Run Commands

This project has no build process or runtime. All content is static Markdown documentation.

**Validation:**

- Review Markdown syntax and structure manually
- Verify all internal links resolve correctly
- Ensure skill `SKILL.md` files follow the template structure defined in `framework.md`

**Distribution:**

- Skills are consumed by agents via the Automatic MCP service (`automatic_read_skill`)
- No packaging or compilation step required

---

## Architecture Overview

**Root-level documentation:**

- `framework.md` — Core specification document; single source of truth for tier definitions and requirements
- `AGENTS.md` — Agent workflow, problem-solving phases, coding patterns, and constitutional rules
- `RULES.md` — Project-specific coding conventions for agents working on this repository
- `README.md` — Public-facing introduction and skill catalogue
- `LICENSE` — Legal terms

**Skills directory (`skills/`):**

Each skill lives in `skills/<skill-name>/SKILL.md` and follows a standard structure:

- **Assessment skills** (`ai-readiness-audit`, `ai-readiness-scaffold`) — Evaluate projects against framework tiers
- **Document authoring skills** (`agents-md`, `rules-md`, `architecture-md`) — Generate or update framework files
- **Code quality remediation skills** (`dead-code-cleanup`, `error-handling-audit`, `boundary-audit`) — Fix common agent-hostile patterns
- **Ongoing compliance skills** (`escalation-check`, `convention-check`) — Verify adherence during development

**Automatic integration (`.automatic/`):**

- `project.json` — Automatic MCP project registration
- `snapshots/` — Auto-generated context snapshots for agent reference

---

## Coding Conventions

**File structure:**

- All documentation uses Markdown with ATX-style headings (`##`, not underlines)
- Skill files must be named `SKILL.md` and placed in `skills/<skill-name>/`
- Root-level files use UPPERCASE names for framework documents (`AGENTS.md`, `RULES.md`, `ARCHITECTURE.md`)

**Writing style:**

- Direct, imperative tone ("Do X", not "You should do X")
- Short, scannable sentences
- No jargon, idioms, or colloquialisms
- Bullet points over prose where possible
- Code examples in fenced blocks with language identifiers

**Naming patterns:**

- Skill names use kebab-case (`ai-readiness-audit`, not `AIReadinessAudit`)
- Heading anchors should be predictable (lowercase, hyphen-separated)
- Cross-references use relative paths (`[framework](framework.md)`, not absolute URLs)

**Content principles:**

- **Concision** — Every sentence must carry information; remove filler
- **Specificity** — Concrete examples over abstract descriptions
- **Verifiability** — Observable criteria over subjective judgement ("tests pass" not "code is good")
- **No placeholders** — If content is incomplete, mark it explicitly with `TODO` or `PLACEHOLDER`

**Error handling:**

Not applicable (documentation-only project).

**Typing:**

Not applicable (documentation-only project).

---

## Agent Guidance

### What agents SHOULD do

- **Follow AGENTS.md workflow** — Always execute the seven-phase problem-solving process (Understand, Context, Plan, Communicate, Implement, Verify, Summarise)
- **Read the framework first** — Before modifying `framework.md` or skills, read the current version to understand tier definitions and requirements
- **Maintain consistency** — Match the tone, structure, and formatting of existing documents when editing
- **Verify links** — Check that all Markdown links resolve to existing files or valid anchors
- **Update both places** — If a requirement changes, update `framework.md` AND the relevant skill instructions
- **Declare uncertainty** — If a framework interpretation is ambiguous, surface it rather than guessing
- **Test skill instructions** — Mentally trace through skill steps to verify they are complete and unambiguous

### What agents SHOULD NOT do

- **Do not invent tiers or requirements** — The four-tier model is fixed; do not add Tier 4 or redefine tier semantics
- **Do not add runtime code** — This is a documentation project; do not introduce build scripts, linters, or executables
- **Do not break skill structure** — Every skill must have Purpose, When to Use, Inputs, Process, Outputs, Success Criteria sections
- **Do not commit secrets** — No API keys, tokens, or credentials (though none should exist in a docs-only project)
- **Do not delete skills without confirmation** — Removing a skill from the catalogue requires explicit user approval
- **Do not silently reformat** — If reformatting Markdown for consistency, summarise what changed and why

### Before committing

- **Run a self-check:**
  - Are all skill names in `README.md` consistent with `skills/` directory names?
  - Do all cross-references resolve correctly?
  - Does the change align with the framework's stated principles (verifiable criteria, no placeholders, minimal scope)?
- **Summarise the change** — State what was modified, why, and any assumptions made
- **Declare incomplete work** — If a TODO was introduced or a section left unfinished, name it explicitly

### Escalation triggers

Stop and ask the user if:

- A requested change contradicts the existing tier model
- A skill modification would make it incompatible with the Automatic MCP service
- Multiple valid interpretations exist for a framework requirement
- You are asked to remove or merge skills without clear rationale
- The change requires knowledge of external systems (e.g., how a specific IDE plugin works)

---

## Working with the Automatic MCP Service

This project is registered with Automatic. At session start:

1. Call `automatic_list_skills` to see available skills
2. Call `automatic_search_memories` with keywords like `ai-readiness`, `framework`, `tier` to retrieve past decisions
3. Call `automatic_read_project` with `"ai-readiness-framework"` to load project configuration

When modifying skills, test them by:

1. Calling `automatic_read_skill` with the skill name to verify the content is parseable
2. Mentally executing the skill's process steps to confirm they are complete and unambiguous

Before ending a session, call `automatic_store_memory` to capture:

- Design decisions about framework requirements
- Clarifications about tier criteria
- Patterns for writing skill instructions
- User preferences about tone or structure

Use hierarchical memory keys like `framework/tier-definitions`, `skills/instruction-template`, `decisions/skill-structure`.

<!-- automatic:groups:start -->
## Related Projects
The following projects are related to this one. They are provided for context — explore or reference them when relevant to the current task.

### Velvet Tiger
Miscellaneous Velvet Tiger projects
**velvet**
Location: `../velvet`
**healthcloud**
Location: `../healthcloud`
**keos-nextjs**
Location: `../keos/keos-nextjs`
**worldtime**
Location: `../../speckitty-test/worldtime`

<!-- automatic:groups:end -->
