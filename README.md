# Agent Skills — anhthqb97

[![skills.sh](https://skills.sh/b/anhthqb97/skills)](https://skills.sh/anhthqb97/skills)

A collection of reusable agent skills for AI coding assistants (Cursor, Claude Code, Codex, and more).

## Available Skills

| Skill | Description |
|-------|-------------|
| `laravel-specialist` | Core architecture — Controller → Action → Repository → Model, PHP Attributes, Eloquent, FormRequest |
| `laravel-testing` | Pest PHP — feature tests, unit tests, factories, mocking, TDD |
| `laravel-api` | REST API design — versioning, JSON:API resources, rate limiting, response envelope |
| `laravel-security` | Policies, Gates, Sanctum scopes, OWASP, mass assignment, input hardening |
| `laravel-performance` | N+1 fixes, eager loading, Redis caching, DB indexing, chunking |
| `laravel-queue` | Horizon, job batching, chaining, retry strategies, failed job handling |
| `laravel-modules` | nwidart/laravel-modules — bounded contexts, cross-module events, service providers |
| `laravel-livewire` | Livewire 3 + Alpine.js — real-time forms, file uploads, lazy components |

## Install

```bash
# Install all skills
npx skills add anhthqb97/skills

# Install a specific skill
npx skills add anhthqb97/skills --skill laravel-specialist

# Install globally (available across all projects)
npx skills add anhthqb97/skills -g
```

## Usage

Once installed, simply ask your agent about any Laravel topic and the skill activates automatically. Example prompts:

- *"Create an Asset module with CRUD API following the project architecture"*
- *"Add a queue job that sends a notification when order status changes"*
- *"Write a FormRequest for creating a product with validation"*

## License

MIT
