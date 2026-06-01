# CodeIgniter 4 Skill

> **Senior CI4 specialist for your agent runtime (OpenCode, Claude Code, Cursor…).**
> Production-grade CodeIgniter 4 code — full-stack MVC, REST APIs for SPAs, Shield auth, Docker, CI3 → CI4 migrations — held to clean code, SOLID, PSR-12, PHPStan level 6.

[![Install](https://img.shields.io/badge/install-npx%20skills%20add%20liusc45%2FCodeIgniter4--skill-blue)](https://skills.sh/)
[![CI4](https://img.shields.io/badge/CodeIgniter-4.5%2B-orange)](https://codeigniter.com/)
[![PHP](https://img.shields.io/badge/PHP-8.2%2B-777BB4)](https://www.php.net/)
[![Shield](https://img.shields.io/badge/Shield-1.x-success)](https://shield.codeigniter.com/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## Table of contents

1. [What is this?](#what-is-this)
2. [Who is this for?](#who-is-this-for)
3. [5-minute quickstart](#5-minute-quickstart)
4. [Your first session with the agent](#your-first-session-with-the-agent)
5. [How to ask the agent (good vs bad prompts)](#how-to-ask-the-agent)
6. [Recommended learning path](#recommended-learning-path)
7. [Reference index — what to read and when](#reference-index)
8. [Common recipes (task → reference)](#common-recipes)
9. [Project structure shipped in this skill](#project-structure)
10. [Conventions this skill enforces](#conventions-this-skill-enforces)
11. [Troubleshooting](#troubleshooting)
12. [Compatibility](#compatibility)
13. [Contributing & extending the skill](#contributing--extending-the-skill)
14. [License & credits](#license--credits)

---

## What is this?

A reusable [Agent Skill](https://skills.sh/) that turns any compatible agent
(OpenCode, Claude Code, Cursor, …) into a **senior CodeIgniter 4 engineer**
specialized in the Sistematlan team's domain.

It ships with:

- **`SKILL.md`** — the manifest (persona, triggers, principles, must-do/must-not-do rules).
- **`references/`** — 21 deep-dive docs (~9 000 lines) covering every CI4 topic you need day to day.
- **Grounded knowledge** — every recommendation cites the official CI4 user guide (Context7-validated).

It works for **new projects**, **legacy CI3 → CI4 migrations**, and **incremental refactors** of older CI4 codebases (`acolhuas`-style anti-patterns → `lotemanager` target state).

---

## Who is this for?

| You are… | This skill helps you… |
|---|---|
| A new dev joining the Sistematlan team | Get productive on CI4 in hours, not weeks, with the team's conventions baked in. |
| A senior dev starting a greenfield project | Bootstrap a CI4 app with clean code, tests, Docker, and Shield in one go. |
| A tech lead reviewing PRs | Have a consistent baseline (PSR-12, PHPStan 6, thin controllers, audited models). |
| A maintainer of a legacy CI4 app | Migrate incrementally to the target state with a 1-file-at-a-time approach. |
| A consultant migrating CI3 → CI4 | Use a battle-tested translation map and Strangler-Fig strategy. |

> If you've never used CodeIgniter before, start with [references/01-installation-setup.md](references/01-installation-setup.md) and the official [CI4 user guide](https://codeigniter4.github.io/userguide/).

---

## 5-minute quickstart

### 1. Install the skill (one command)

```bash
npx skills add liusc45/CodeIgniter4-skill -g -y
```

The `-g` flag installs it globally so every project on your machine can use it.
Verify:

```bash
npx skills list | grep codeigniter4
# → codeigniter4-specialist
```

### 2. Restart your agent

Restart OpenCode / Claude Code / Cursor so it picks up the new skill.

### 3. Sanity-check the agent

In any project directory, ask:

> "What is your specialization? What conventions do you enforce for CodeIgniter 4?"

The agent should answer with a senior PHP engineer persona, mention PSR-12,
PHPStan level 6, fat-models/thin-controllers, DB transactions, native CORS,
Shield, and the Sistematlan baseline. If it doesn't, the skill is not loaded —
see [Troubleshooting](#troubleshooting).

### 4. Bootstrap a project (or point it at an existing one)

For a **new** project:

> "Scaffold a new CodeIgniter 4.7 app called `inventory` with MySQL, Shield auth, and PHPUnit."

For an **existing** project:

> "Explore this CodeIgniter 4 codebase and tell me which team-pattern reference
> (lotemanager or acolhuas) better describes it."

### 5. Build your first feature

> "Add a `Product` resource (migration, model, entity, controller, routes, tests)
> following the Sistematlan conventions. Include audit columns, validation rules,
> and pagination."

That's it — you should have a working, tested CRUD endpoint in under a minute.

---

## Your first session with the agent

Here's a realistic 30-minute onboarding session. Open your agent and run the
prompts in order:

| # | Prompt | What you'll get |
|---|--------|-----------------|
| 1 | *"What does the Sistematlan team-pattern document say about controllers?"* | The agent will pull from `references/19-team-patterns-lotemanager.md` and summarize. |
| 2 | *"Bootstrap a CI4 app named `acme` with MySQL, Shield, and PHPUnit."* | A new project with proper structure. |
| 3 | *"Add an `Order` resource with audit columns and pagination."* | Migration + Model + Entity + Controller + Routes + tests. |
| 4 | *"Dockerize for production using FrankenPHP (worker mode)."* | A `Dockerfile`, `docker-compose.yml`, and Caddy/FrankenPHP config. |
| 5 | *"Review the controllers you just created and rate them against the clean-code reference."* | Honest self-review against `references/21-clean-code-efficiency.md`. |
| 6 | *"Show me the most common acolhuas anti-pattern and how you'd fix it."* | Real example from `references/20-legacy-acolhuas-fixes.md`. |

After this session you should be able to navigate the codebase, the team's
conventions, and the skill's documentation confidently.

---

## How to ask the agent

The agent is excellent — but only if you ask well. Here are patterns that work.

### ✅ Good prompts

> **Be specific about what you want delivered.**
> *"Create a `Customer` resource with these fields: name (required, max 120), email (required, valid email, unique), phone (optional), notes (optional, max 500). Use audit columns, soft deletes, and Shield for auth. Generate the migration, model with validation, entity with casts, REST controller, routes with the `cors` filter, and a feature test for `create`."*

> **Cite the reference you want applied.**
> *"Refactor `Sale::create` to use a Service class and wrap it in a DB transaction, per the clean-code reference."*

> **Ask for trade-offs explicitly.**
> *"Compare FrankenPHP vs Nginx + PHP-FPM for a 50 RPS API. I want max throughput but I need easy debugging."*

> **Request self-review.**
> *"Now run PHPStan and PHP-CS-Fixer mentally on the code you wrote. Report any issues."*

### ❌ Bad prompts

> *"Make a CRUD."* — for what entity? with what fields? what auth? what server?

> *"Optimize this."* — optimize for what? latency, memory, readability?

> *"Add authentication."* — Shield session? tokens? JWT? HMAC? SPA or server-rendered?

### 🎯 Prompt template

Use this when you're not sure what to ask:

```text
Goal:        <what business outcome you want>
Stack:       <full-stack MVC | REST API for React | …>
Auth:        <Shield session | Shield tokens | none>
Database:    <MySQL | PostgreSQL | SQLite>
Server:      <Apache | Nginx-FPM | Caddy-FPM | FrankenPHP>
Constraints: <must support X, must not change Y, …>
Deliver:     <migration | model | controller | routes | tests | docker | all>
```

---

## Recommended learning path

If you have ~2 hours, read in this order. Each step builds on the previous.

| Step | Read | Why |
|------|------|-----|
| 1 | [`SKILL.md`](SKILL.md) | The agent's manifest — persona, principles, must-do/must-not-do. **Skim, don't memorize.** |
| 2 | [`references/01-installation-setup.md`](references/01-installation-setup.md) | Project layout, .env, Spark CLI basics. |
| 3 | [`references/03-routing-controllers.md`](references/03-routing-controllers.md) | How CI4 routes requests and how to write thin controllers. |
| 4 | [`references/02-models-database.md`](references/02-models-database.md) | Models, Query Builder, entities, casts, transactions. |
| 5 | [`references/09-migrations-seeds.md`](references/09-migrations-seeds.md) | Forge, foreign keys, seeders, Faker. |
| 6 | [`references/06-validation-forms.md`](references/06-validation-forms.md) | Rules, custom validators, file upload. |
| 7 | [`references/11-testing-phpunit.md`](references/11-testing-phpunit.md) | Unit, database, and feature tests. |
| 8 | [`references/21-clean-code-efficiency.md`](references/21-clean-code-efficiency.md) | **The reference that makes you a better engineer.** Naming, SRP, error handling, security, performance, tooling (PHPStan, Rector, CS-Fixer). |
| 9 | [`references/19-team-patterns-lotemanager.md`](references/19-team-patterns-lotemanager.md) | The Sistematlan baseline — what "good" looks like for this team. |
| 10 | [`references/20-legacy-acolhuas-fixes.md`](references/20-legacy-acolhuas-fixes.md) | 13 anti-patterns you'll see in older CI4 code, with fixes. |

For SPA / API work, also read:
- [`references/05-rest-api.md`](references/05-rest-api.md) — REST for React/Angular/Vue.
- [`references/07-shield-auth.md`](references/07-shield-auth.md) — sessions, tokens, JWT, HMAC, 2FA.
- [`references/08-filters-middleware.md`](references/08-filters-middleware.md) — CORS, rate limiting, auth.

For Docker / DevOps, also read:
- [`references/12` through `15`](references/) — pick the server that matches your platform.

For migration from CI3, read [`references/16-legacy-migration.md`](references/16-legacy-migration.md).

---

## Reference index

21 docs, grouped by purpose.

### Core framework

| # | Topic | Read when… |
|---|-------|-----------|
| 01 | [Installation & Setup](references/01-installation-setup.md) | You're starting a new project or onboarding. |
| 02 | [Models & Database](references/02-models-database.md) | You're writing or reviewing a model. |
| 03 | [Routing & Controllers](references/03-routing-controllers.md) | You need a new route or controller. |
| 04 | [Views & Frontend](references/04-views-frontend.md) | You're building server-rendered pages. |
| 09 | [Migrations & Seeds](references/09-migrations-seeds.md) | You're changing the schema. |
| 10 | [Services & DI](references/10-services-di.md) | You're writing non-trivial business logic. |

### Quality & security

| # | Topic | Read when… |
|---|-------|-----------|
| 06 | [Validation & Forms](references/06-validation-forms.md) | You accept user input (always). |
| 08 | [Filters & Middleware](references/08-filters-middleware.md) | You add cross-cutting concerns (CORS, auth, rate limit). |
| 11 | [Testing with PHPUnit](references/11-testing-phpunit.md) | You ship code (always). |
| 17 | [Best Practices](references/17-best-practices.md) | You want a quick checklist. |
| 21 | **[Clean Code & Efficiency](references/21-clean-code-efficiency.md)** | **You want to be a better engineer. Mandatory reading.** |

### API & auth

| # | Topic | Read when… |
|---|-------|-----------|
| 05 | [REST API for SPAs](references/05-rest-api.md) | You expose a JSON API to React/Angular/Vue. |
| 07 | [Shield Authentication](references/07-shield-auth.md) | You add login, tokens, 2FA, RBAC. |

### Deployment

| # | Topic | Read when… |
|---|-------|-----------|
| 12 | [Docker: Apache](references/12-docker-apache.md) | Legacy `.htaccess` compatibility, simple dev. |
| 13 | [Docker: Nginx + PHP-FPM](references/13-docker-nginx-fpm.md) | Standard production, max compatibility. |
| 14 | [Docker: Caddy + PHP-FPM](references/14-docker-caddy-fpm.md) | Auto HTTPS, simpler config, HTTP/2-3. |
| 15 | [Docker: FrankenPHP](references/15-docker-frankenphp.md) | Cutting-edge perf with Worker Mode. |
| 18 | [Production Deployment](references/18-deployment-production.md) | Pre-launch checklist, OPcache, CI/CD, zero-downtime. |

### Migration & team patterns

| # | Topic | Read when… |
|---|-------|-----------|
| 16 | [Legacy CI3 → CI4](references/16-legacy-migration.md) | You're migrating from CI3. |
| 19 | **[Team Patterns (lotemanager)](references/19-team-patterns-lotemanager.md)** | **You work on Sistematlan code. This is the baseline.** |
| 20 | **[Legacy Patterns to Fix (acolhuas)](references/20-legacy-acolhuas-fixes.md)** | **You inherit older CI4 code. 13 anti-patterns + fixes.** |

---

## Common recipes

| I want to… | Use this reference | Example prompt |
|------------|-------------------|----------------|
| Start a new project | [01](references/01-installation-setup.md) | *"Scaffold a new CI4 app called `acme` with MySQL, Shield, PHPUnit, and FrankenPHP."* |
| Add a CRUD resource | [02](references/02-models-database.md) + [03](references/03-routing-controllers.md) + [09](references/09-migrations-seeds.md) + [11](references/11-testing-phpunit.md) | *"Add an `Order` resource with audit columns, soft deletes, and feature tests."* |
| Expose a JSON API to a SPA | [05](references/05-rest-api.md) + [08](references/08-filters-middleware.md) | *"Add `/api/products` with CORS, JWT auth, and pagination for a React frontend."* |
| Add login / signup / 2FA | [07](references/07-shield-auth.md) | *"Add Shield with email + password, JWT tokens for mobile, and TOTP 2FA."* |
| Validate a form | [06](references/06-validation-forms.md) | *"Validate the contact form: name required, email valid, message min 10 chars."* |
| Refactor a fat controller | [10](references/10-services-di.md) + [21](references/21-clean-code-efficiency.md) | *"Extract the logic in `OrderController::create` into an `OrderService`."* |
| Add transactions | [02](references/02-models-database.md) | *"Wrap `Sale::create` in a transaction so partial writes never persist."* |
| Add audit columns | [19](references/19-team-patterns-lotemanager.md) | *"Add `created_by`, `updated_by`, `deleted_by` audit columns with model callbacks."* |
| Improve an old CI4 codebase | [20](references/20-legacy-acolhuas-fixes.md) | *"List the acolhuas anti-patterns in `app/Controllers/` and propose 1-file-at-a-time fixes."* |
| Migrate from CI3 | [16](references/16-legacy-migration.md) | *"Plan the migration of this CI3 app using Strangler Fig. Start with auth."* |
| Dockerize for production | [12–15](references/) + [18](references/18-deployment-production.md) | *"Dockerize with FrankenPHP worker mode and zero-downtime deploys."* |
| Add tests | [11](references/11-testing-phpunit.md) | *"Write feature tests for `ProductController::index`, `show`, `create`, `update`, `delete`."* |
| Static analysis | [21](references/21-clean-code-efficiency.md) | *"Set up PHPStan level 6, PHP-CS-Fixer (PSR-12), and Rector. Show the composer scripts."* |

---

## Project structure

```
CodeIgniter4-skill/
├── SKILL.md                          # Agent manifest (persona, principles, rules)
├── README.md                         # ← you are here
├── LICENSE                           # MIT
└── references/
    ├── 01-installation-setup.md
    ├── 02-models-database.md
    ├── 03-routing-controllers.md
    ├── 04-views-frontend.md
    ├── 05-rest-api.md
    ├── 06-validation-forms.md
    ├── 07-shield-auth.md
    ├── 08-filters-middleware.md
    ├── 09-migrations-seeds.md
    ├── 10-services-di.md
    ├── 11-testing-phpunit.md
    ├── 12-docker-apache.md
    ├── 13-docker-nginx-fpm.md
    ├── 14-docker-caddy-fpm.md
    ├── 15-docker-frankenphp.md
    ├── 16-legacy-migration.md
    ├── 17-best-practices.md
    ├── 18-deployment-production.md
    ├── 19-team-patterns-lotemanager.md   # Sistematlan baseline
    ├── 20-legacy-acolhuas-fixes.md      # 13 anti-patterns + fixes
    └── 21-clean-code-efficiency.md      # Mandatory clean-code reference
```

---

## Conventions this skill enforces

Every code sample the agent produces follows these rules. New team members
should learn them as the team's bar.

### Code style
- `declare(strict_types=1);` in **every** PHP file.
- PSR-12 formatting (enforced via PHP-CS-Fixer).
- PHPStan level 6 (no implicit types, no dead code, all params/returns typed).
- `final` classes by default; `final readonly` for DTOs/Value Objects.

### Architecture
- **Fat models, thin controllers.** Controllers orchestrate; business logic lives in Services.
- **ResourceController** for CRUD; **Services** for any non-trivial multi-step logic.
- **Dependency injection** via constructor; service registered in `Config/Services.php`.
- **No `global` helpers** in business code; only views and config.

### Data layer
- **Validation rules in the model** (`$validationRules`), not the controller.
- **DB transactions** on every multi-write action (`transStart` / `transComplete` / `transStatus`).
- **Audit columns** auto-filled via `$beforeInsert/Update/Delete` callbacks.
- **Entities** with property casts (`$casts`); prefer `final readonly` DTOs for inputs.

### Security
- Native `cors:api` filter for SPAs — never a custom CorsFilter.
- CSRF enabled on all state-changing forms; meta tag for AJAX.
- Parameterized queries (Query Builder / `$db->query()` with bindings); never string-concat.
- Shield for auth (session / tokens / JWT / HMAC / 2FA).

### Quality
- Tests for every endpoint (unit + database + feature).
- `phpstan analyse` and `php-cs-fixer fix` in CI.
- No `var_dump` / `dd` / `error_log` in production code.

The complete checklist lives in [`references/21-clean-code-efficiency.md`](references/21-clean-code-efficiency.md).

---

## Manual installation

If you don't want to use the `npx skills` CLI:

```bash
# OpenCode
git clone https://github.com/liusc45/CodeIgniter4-skill.git \
  ~/.config/opencode/skills/codeigniter4-specialist

# Claude Code
git clone https://github.com/liusc45/CodeIgniter4-skill.git \
  ~/.claude/skills/codeigniter4-specialist

# Cursor / other
git clone https://github.com/liusc45/CodeIgniter4-skill.git \
  ~/.cursor/skills/codeigniter4-specialist
```

Then **restart your agent**.

---

## Troubleshooting

### The agent doesn't know about the skill

1. Confirm the path matches your runtime:
   - OpenCode: `~/.config/opencode/skills/codeigniter4-specialist/SKILL.md`
   - Claude Code: `~/.claude/skills/codeigniter4-specialist/SKILL.md`
2. Restart the agent (skills are loaded at startup).
3. Ask the agent: *"What is your specialization?"* — it should mention CodeIgniter 4.

### The agent ignores the conventions

Make your prompt explicit:

> *"Follow the Sistematlan team patterns (references/19) and the clean-code reference (references/21). Cite the references you use."*

### The agent produces code without tests

> *"Add the feature tests for every new endpoint. Use database transactions so they rollback."*

### PHPStan or PHP-CS-Fixer fails on generated code

> *"Run PHPStan level 6 and PHP-CS-Fixer (PSR-12) on the code you just wrote. Fix any issues."*

### The agent uses a custom CorsFilter

That violates the conventions. Say:

> *"Use the native `cors:api` filter from `app/Config/Filters.php`, not a custom one. See references/08."*

### I want to migrate from CI3

Start with [`references/16-legacy-migration.md`](references/16-legacy-migration.md) and
ask the agent to use the **Strangler Fig** strategy — don't rewrite the whole app at once.

### I want to extend the skill

See [Contributing & extending the skill](#contributing--extending-the-skill).

---

## Compatibility

| Component | Supported |
|-----------|-----------|
| Agent runtimes | OpenCode, Claude Code, Cursor (any tool that supports the Skills format) |
| CodeIgniter | 4.5+ (4.7+ recommended) |
| PHP | 8.2+ (8.1 minimum) |
| Auth | Shield 1.x |
| Testing | PHPUnit 10+ |
| Package manager | Composer 2 |
| Frontend (optional) | Bootstrap 5, Bun, Gulp, DataTables, SweetAlert2, flatpickr (matches Sistematlan stack) |
| Servers | Apache, Nginx + PHP-FPM, Caddy + PHP-FPM, FrankenPHP, LiteSpeed |

---

## Contributing & extending the skill

PRs are welcome. The skill is a living document — when the team adopts a new
pattern, this repo should reflect it.

### To add a new reference

1. Pick the next number (`22-your-topic.md`).
2. Follow the structure of the existing files:
   - **Why** — when to use it
   - **What** — the key concepts
   - **How** — copy-pasteable, runnable examples
   - **Anti-patterns** — what NOT to do
   - **Cites** — link to the official CI4 user guide
3. Update the index in this README and in `SKILL.md`.
4. Keep examples self-contained (they should run in a fresh CI4 install).

### To update an existing reference

1. Cite the official CI4 docs when adding new content.
2. Use English for all prose (code samples may include `es-MX` strings if they match the team).
3. Keep examples copy-pasteable and runnable.
4. Update the cross-references in the other docs if you change structure.

### Style guide for references

- One H1 (`# Topic`).
- H2 for major sections (Why, What, How, Anti-patterns, Cites).
- H3 for subsections.
- Code fences tagged with the language (`php`, `bash`, `yaml`, …).
- Tables for comparisons.
- "Good" and "Bad" code blocks side-by-side when showing anti-patterns.

### Pull request checklist

- [ ] Examples are copy-pasteable and runnable.
- [ ] Each new recommendation cites the official CI4 user guide.
- [ ] No broken internal links (run `grep -r "\.md" references/` to verify).
- [ ] This README's reference index is updated.
- [ ] `SKILL.md` is updated if you added a new trigger keyword or capability.

---

## License

[MIT](LICENSE).

---

## Credits

Built on top of the excellent [CodeIgniter 4](https://codeigniter.com/) framework
and the [Skills CLI](https://skills.sh/) ecosystem.

Maintained by the [Sistematlan](https://github.com/liusc45) team.
