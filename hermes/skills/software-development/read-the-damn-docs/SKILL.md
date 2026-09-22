---
name: read-the-damn-docs
description: "Read official docs before coding third-party integrations."
---

# Read The Damn Docs

Do not guess where authoritative docs can answer the question. The most common
right move is to web-search for the current official docs, open the relevant
pages, and read them before coding. For APIs, versions, provider behavior,
config, limits, lifecycle hooks, or security-sensitive flows, ground the answer
in what the docs actually say.

## Docs-First Triggers

Read docs before proceeding when any of these are true:

- The user asks for "latest", "current", "official", "supported", "best
 practice", "recommended", "today", "now", or "look it up".
- The needed docs are not already in the repo or supplied by the user. Search
 the web for the official docs rather than hoping model memory is current.
- The task adds, upgrades, configures, or imports a package, SDK, framework,
 plugin, CLI, model, cloud resource, or provider integration.
- The API is fast-moving or version-sensitive: AI SDKs, OpenAI/Anthropic/Google
 APIs, Next.js, React, Tailwind, Vite, Nitro, Drizzle, Prisma, Stripe, GitHub,
 Slack, Notion, browser APIs, deployment platforms, auth libraries, and similar.
- The implementation depends on auth, OAuth scopes, permissions, secrets,
 webhooks, billing, payments, PII, encryption, data retention, migrations,
 retries, rate limits, quotas, caching, deploys, or compliance.
- An error mentions deprecation, unknown options, missing exports, invalid
 config, unsupported fields, changed defaults, or version mismatch.
- A repo has local docs, ADRs, generated schemas, OpenAPI specs, route/action
 registries, design-system docs, or package-level READMEs that could define the
 contract.
- The choice is expensive to reverse: public wire formats, database schema,
 migration strategy, persistent IDs, event names, customer-visible behavior, or
 external automation contracts.
- You catch yourself about to write "usually", "probably", "I think", "from
 memory", or code copied from model memory for an external API.

## What Counts As Docs

Use the most authoritative source available:

- Local repo docs, specs, ADRs, schemas, generated types, package READMEs, and
 tests for project-specific behavior.
- Official product docs, API references, migration guides, changelogs, release
 notes, and SDK source/types for third-party behavior. Find these with web
 search when you do not already have the exact URL.
- Package registry metadata for versions. Before adding a dependency, confirm the
 exact package name, install command, and current version from the registry
 or official docs.

## What NOT To Rely On

- Model memory from training data for version-specific behavior, config options,
 API signatures, or defaults. It may be outdated.
- Stack Overflow answers older than 18 months unless you can verify the behavior
 still holds.
- Another project's code as proof of correct usage. It may be wrong.
- README snippets that do not match the installed version.

## How To Read Docs

1. **Search for the official docs.** Use `web_search` or `web_extract` to find
   the current official documentation.
2. **Read the actual page.** Do not skim the first result and assume. Read the
   relevant section.
3. **Check the version.** Make sure the docs match the version in use.
4. **Cross-reference with source if needed.** For critical behavior, check the
   package source code or type definitions.
5. **Cite the source.** When stating a fact from docs, note the URL or file
   path so the user can verify.

## Application To This Project

For h-dashboard (Laravel 13.x, Livewire 4, MaryUI, Tailwind 4, PHP 8.3):

- Use Laravel Boost MCP `search_docs` to find version-specific Laravel docs.
- Use Context7 MCP `query_docs` with libraryId `/laravel/docs` for framework
  questions.
- Use `codegraph explore` for code-structure questions before grep/read_file.
- Always check the actual installed package versions via `composer.json` or
  Laravel Boost `application_info` before assuming API behavior.