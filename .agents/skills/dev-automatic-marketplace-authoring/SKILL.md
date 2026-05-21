---
name: dev-automatic-marketplace-authoring
description: Instructions on how to add and manage new marketplace items in Automatic
---

# Automatic Marketplace Authoring

## Overview

Three JSON catalogues live in `src-tauri/assets/marketplace/`. They are
embedded in the binary at compile time and written to
`~/.automatic/marketplace/` on every app startup. Editing the source files
and rebuilding the app is how new entries ship; once the remote endpoint at
`tryautomatic.app` is live, entries can also be pushed over the network
without a rebuild.

| Catalogue | Source file | What it powers |
|---|---|---|
| MCP Servers | `featured-mcp-servers.json` | MCP Marketplace tab |
| Collections | `collections.json` | Collections tab |
| Project Templates | `project-templates/*.json` | Template Marketplace tab |
| Featured Skills | `featured-skills.json` | Skill Store landing page |

---

## MCP Servers (`featured-mcp-servers.json`)

Append an object to the top-level array. Every field is described below.

### Full schema

```json
{
  "slug": "provider-short-name",
  "name": "reverse.dns.namespace/package-identifier",
  "title": "Human Readable Title",
  "description": "One or two sentence description of what the server does.",
  "provider": "Provider Company Name",
  "icon": "provider-domain.com",
  "classification": "official",
  "repository_url": "https://github.com/org/repo",
  "docs_url": "https://docs.example.com/mcp",
  "remote": {
    "transport": "streamable-http",
    "url": "https://mcp.example.com/mcp"
  },
  "local": {
    "registry": "npm",
    "package": "@org/mcp-server",
    "version": null,
    "transport": "stdio",
    "command": "npx -y @org/mcp-server"
  },
  "auth": {
    "method": "api_key",
    "env_vars": [
      {
        "name": "EXAMPLE_API_KEY",
        "description": "API key from https://example.com/settings",
        "secret": true
      }
    ]
  },
  "companion_skill": {
    "name": "skill-name",
    "title": "Skill Display Title",
    "description": "One line — what the skill instructs the agent to do.",
    "url": "https://raw.githubusercontent.com/org/repo/main/SKILL.md",
    "github_source": "org/repo"
  }
}
```

### Field rules

**`slug`** — URL-safe, kebab-case unique identifier. Convention:
`{provider}-{short-product-name}`. Examples: `github`, `amplitude-eu`,
`microsoft-playwright`. Must be unique across the entire array.

**`name`** — Registry-style namespaced identifier. Conventions by source:
- GitHub-hosted: `io.github.{owner}/{repo-name}`
- Official vendor package: `com.{vendor}/{product}`
- PulseMCP mirror: `com.pulsemcp.mirror/{original-slug}`

**`classification`** — One of:
- `"official"` — published or endorsed by the originating company
- `"reference"` — the Anthropic MCP reference server collection
- `"community"` — third-party, not vendor-endorsed

**`icon`** — A Brandfetch-resolvable domain (e.g. `"github.com"`). Omit the
field entirely if no brand domain is available — do not use `null`.

**`docs_url`** — Optional. Include only when there is dedicated setup
documentation distinct from the repository README. Omit otherwise.

**`remote`** — Present when the server supports a remote transport. Use
`"streamable-http"` for modern servers; `"sse"` for legacy SSE-only servers.
Set to `null` (not omitted) when remote is not supported.

**`local`** — Present when the server can be run locally. `registry` values:
`"npm"`, `"pypi"`, `"oci"`, `"nuget"`. `version` should be `null` unless
pinning to a specific version is intentional. Set to `null` when local
install is not available. Do not omit — the UI uses `null` as a signal.

**`auth.method`** — One of `"none"`, `"api_key"`, `"oauth"`.
- `"none"` — no credentials required; `env_vars` must be `[]`
- `"api_key"` — one or more env vars the user must supply; `secret: true`
  for tokens/keys, `secret: false` for non-sensitive config values
- `"oauth"` — handled by the app's OAuth flow; `env_vars` must be `[]`

**`companion_skill`** — Optional. Use when the server works significantly
better with an accompanying skill. `url` must be a raw, publicly accessible
URL to the SKILL.md content. `github_source` is `"owner/repo"` (or just
`"owner"`) used to track the skill's origin.

### Minimal example (local-only, no auth)

```json
{
  "slug": "acme-database",
  "name": "io.github.acme/database-mcp",
  "title": "Acme Database",
  "description": "Query and manage Acme databases through natural language.",
  "provider": "Acme",
  "icon": "acme.com",
  "classification": "official",
  "repository_url": "https://github.com/acme/database-mcp",
  "remote": null,
  "local": {
    "registry": "npm",
    "package": "@acme/database-mcp",
    "version": null,
    "transport": "stdio",
    "command": "npx -y @acme/database-mcp"
  },
  "auth": {
    "method": "none",
    "env_vars": []
  }
}
```

### Remote-only with OAuth example

```json
{
  "slug": "acme-cloud",
  "name": "com.acme/cloud-mcp",
  "title": "Acme Cloud",
  "description": "Access Acme Cloud resources through natural language.",
  "provider": "Acme",
  "icon": "acme.com",
  "classification": "official",
  "repository_url": null,
  "docs_url": "https://docs.acme.com/mcp",
  "remote": {
    "transport": "streamable-http",
    "url": "https://mcp.acme.com"
  },
  "local": null,
  "auth": {
    "method": "oauth",
    "env_vars": []
  }
}
```

---

## Project Templates (`project-templates/*.json`)

Each template is a **separate JSON file** in
`src-tauri/assets/marketplace/project-templates/`. The filename must match
the `name` field exactly: `{name}.json`.

After adding a file, register it in
`src-tauri/src/core/project_templates.rs` in the `BUNDLED_TEMPLATES` const:

```rust
(
    "your-template-name",
    include_str!("../../assets/marketplace/project-templates/your-template-name.json"),
),
```

### Full schema

```json
{
  "name": "kebab-case-unique-name",
  "display_name": "Human Readable Name",
  "icon": "framework-domain.com",
  "description": "Two to three sentence description of the stack and use case.",
  "category": "Web Application",
  "tags": ["tag1", "tag2"],
  "skills": ["skill-name-one", "skill-name-two"],
  "mcp_servers": [],
  "providers": [],
  "agents": [],
  "project_files": [],
  "unified_instruction": "# Template Title\n\n## Stack\n...",
  "unified_rules": [],
  "_author": { "type": "provider", "name": "Automatic", "url": "https://automatic.sh" }
}
```

### Field rules

**`name`** — Unique kebab-case slug. Used as the filename and as the lookup
key in `BUNDLED_TEMPLATES`. No spaces, no uppercase.

**`display_name`** — Title-case human name shown in the marketplace grid.

**`icon`** — Brandfetch domain for the primary framework/language. Omit the
field if none applies.

**`category`** — One of the existing values:
`"Web Application"`, `"API / Backend"`, `"Data & Analytics"`,
`"Desktop App"`, `"Infrastructure"`, `"Frontend"`, `"General"`.
Add a new string only if none of the above fit.

**`tags`** — Lowercase, hyphenated. Used for search matching. Include the
primary language, framework, and domain keywords.

**`skills`** — Array of skill name strings. Only list skills that are bundled
with the app; the dependency checker uses this list to auto-install on
template import.

**`mcp_servers`** — Array of MCP server config name strings. Only include
servers that are commonly needed for the stack — leave empty when not
applicable.

**`project_files`** — Array of `{ "filename": "...", "content": "..." }`
objects for files written to the project directory on import. Leave empty
when not needed.

**`unified_instruction`** — Markdown string written to the agent instruction
file when the template is applied. Include: stack summary, directory
structure, key conventions, and common commands. Use `\n` for line breaks
inside the JSON string.

**`unified_rules`** — Array of rule IDs to attach to the project. Reference
rules that already exist in `~/.automatic/rules/`.

**`_author`** — Always include. For Automatic-authored templates:
```json
{ "type": "provider", "name": "Automatic", "url": "https://automatic.sh" }
```

### Example

```json
{
  "name": "remix-fullstack",
  "display_name": "Remix Full Stack",
  "icon": "remix.run",
  "description": "Full-stack web app with Remix, Prisma, and Tailwind CSS. Includes nested routing, loader/action patterns, and a pre-configured database layer.",
  "category": "Web Application",
  "tags": ["remix", "react", "typescript", "tailwind", "prisma"],
  "skills": ["vercel-react-best-practices", "tailwindcss-development"],
  "mcp_servers": [],
  "providers": [],
  "agents": [],
  "project_files": [],
  "unified_instruction": "# Remix Full Stack\n\n## Stack\n- Remix with nested routing\n- Tailwind CSS v3\n- Prisma + PostgreSQL\n- TypeScript strict mode\n",
  "unified_rules": [],
  "_author": { "type": "provider", "name": "Automatic", "url": "https://automatic.sh" }
}
```

---

## Collections (`collections.json`)

Append an object to the top-level array.

### Full schema

```json
{
  "id": "owner/collection-slug",
  "name": "Collection Display Name",
  "slug": "collection-slug",
  "description": "Two to three sentence description of the collection's theme and contents.",
  "author": {
    "type": "provider",
    "name": "Provider Name",
    "url": "https://provider.com",
    "repository_url": "https://github.com/owner/repo"
  },
  "icon": "provider-domain.com",
  "tags": ["tag1", "tag2"],
  "skills": [
    {
      "name": "skill-name",
      "display_name": "Skill Display Name",
      "description": "One sentence description.",
      "source": "owner/repo",
      "id": "owner/repo/skill-name",
      "kind": "bundled"
    }
  ],
  "mcp_servers": [
    {
      "name": "config-key-name",
      "display_name": "Server Display Name",
      "description": "One sentence description.",
      "config": {}
    }
  ],
  "templates": [
    {
      "name": "template-name",
      "display_name": "Template Display Name",
      "description": "One sentence description."
    }
  ]
}
```

### Field rules

**`id`** — `{owner}/{slug}` — globally unique. Mirrors the GitHub repo path
for GitHub-sourced collections.

**`slug`** — URL-safe kebab-case. Must match the slug portion of `id`.

**`author.type`** — `"provider"` for official/vendor collections; `"github"`
for community collections.
- `"provider"`: include `name`, `url`, optionally `repository_url`
- `"github"`: include `repo` (e.g. `"owner/repo"`)

**`icon`** — Brandfetch domain. For `"provider"` authors use the provider's
brand domain. Omit the field for community collections without a clear brand.

**`skills[].kind`** — `"bundled"` for skills shipped with the Automatic app;
`"github"` for skills sourced from a GitHub repo.

**`skills[].source`** — `"owner/repo"` GitHub path the skill is sourced
from. For bundled skills use `"automatic/automatic-app"`.

**`skills[].id`** — `"{source}/{name}"` — must be unique within the
collection.

**`mcp_servers[].name`** — The config key used when the server is installed
locally (same as the filename without `.json` in
`~/.automatic/mcp_servers/`).

**`mcp_servers[].config`** — The full MCP config object that would be
written on install. Can be left as `{}` when the server is only referenced
conceptually in the collection.

**`templates[].name`** — Must match the `name` field of a template in
`project-templates/*.json`.

### Example

```json
{
  "id": "acme/devops-collection",
  "name": "Acme DevOps",
  "slug": "acme-devops",
  "description": "A curated set of skills, MCP servers, and templates for DevOps and infrastructure work. Covers CI/CD, Terraform, and cloud deployment workflows.",
  "author": {
    "type": "provider",
    "name": "Acme",
    "url": "https://acme.com",
    "repository_url": "https://github.com/acme/devops-collection"
  },
  "icon": "acme.com",
  "tags": ["devops", "infrastructure", "terraform", "cicd"],
  "skills": [
    {
      "name": "terraform-skill",
      "display_name": "Terraform",
      "description": "Terraform module authoring, testing, and state management.",
      "source": "automatic/automatic-app",
      "id": "automatic/automatic-app/terraform-skill",
      "kind": "bundled"
    }
  ],
  "mcp_servers": [],
  "templates": [
    {
      "name": "terraform-aws-infrastructure",
      "display_name": "Terraform AWS Infrastructure",
      "description": "Production-ready Terraform modules for AWS."
    }
  ]
}
```

---

## Featured Skills (`featured-skills.json`)

The landing page grid in the Skill Store. Append an object to the array.

### Schema

```json
{
  "name": "skill-name",
  "source": "owner/repo",
  "id": "owner/repo/skill-name",
  "displayName": "Skill Display Name",
  "description": "One sentence description of what the skill does.",
  "installs": 0,
  "license": "MIT"
}
```

### Field rules

**`name`** — Must exactly match the skill's directory name on skills.sh /
GitHub.

**`source`** — `"owner/repo"` GitHub path.

**`id`** — `"{source}/{name}"`.

**`installs`** — Approximate install count from skills.sh. Use `0` for new
skills; update once real data is available.

**`license`** — Optional. SPDX identifier. Omit the field if the license is
unknown or not applicable.

---

## Checklist before committing

- [ ] `slug` / `name` / `id` is unique within its catalogue file
- [ ] `classification` / `category` / `author.type` uses an existing value
- [ ] `remote` and `local` are explicitly `null` when not applicable (not omitted)
- [ ] `auth.env_vars` is `[]` (not omitted) when `method` is `"none"` or `"oauth"`
- [ ] For templates: filename matches `name`, entry added to `BUNDLED_TEMPLATES` in `project_templates.rs`
- [ ] `cargo check` passes
- [ ] `npm run build` passes
