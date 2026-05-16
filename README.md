# Agent Skills — anhthqb97

[![skills.sh](https://skills.sh/b/anhthqb97/skills)](https://skills.sh/anhthqb97/skills)

A collection of reusable agent skills for AI coding assistants (Cursor, Claude Code, Codex, and more).

## Available Skills

### `laravel-specialist`

Expert Laravel 13 / PHP 8.3+ engineer skill. Covers:

- Module-based architecture (`nwidart/laravel-modules`)
- Action/UseCase + Repository pattern
- Eloquent ORM with PHP Attributes (`#[Fillable]`, `#[ObservedBy]`, etc.)
- JSON:API resources, FormRequest validation
- Queue jobs with PHP 8.3 Attributes
- Sanctum authentication
- Bilingual i18n (`lang/en` + `lang/vi`)

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
