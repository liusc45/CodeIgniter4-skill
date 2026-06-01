# CodeIgniter 4 Skill

A comprehensive [Agent Skill](https://skills.sh/) for building modern CodeIgniter 4 applications — covering full-stack web apps (HTML/CSS/JS), REST APIs for SPA frontends (React/Angular/Vue), Shield authentication, and multi-server Docker deployments (Apache, Nginx + PHP-FPM, Caddy + PHP-FPM, FrankenPHP).

Works for **new projects** AND **legacy CI3 → CI4 migrations**.

## Install

Install globally with one command:

```bash
npx skills add liusc45/CodeIgniter4-skill -g -y
```

Or browse all skills at https://skills.sh/.

## What's Included

### `SKILL.md`

The main skill manifest with:

- Specialist persona (senior CI4 engineer, 10+ years PHP)
- Trigger keywords (CodeIgniter, CI4, Shield, Spark CLI, etc.)
- Stack detection workflow (full-stack vs API-only vs legacy)
- Why CI4? Advantages and use cases
- Daily-use Spark CLI cheat sheet
- Docker server option comparison matrix
- Output templates and constraints (MUST DO / MUST NOT DO)

### `references/`

18 deep-dive reference documents:

| # | Topic | Description |
|---|-------|-------------|
| 01 | Installation & Setup | Composer, .env, Spark CLI, project structure |
| 02 | Models & Database | Models, Query Builder, Entities, casts, transactions |
| 03 | Routing & Controllers | Routes, ResponseTrait, ResourceController, REST |
| 04 | Views & Frontend | Layouts, sections, View Cells, asset management, Vite |
| 05 | REST API for SPAs | React, Angular, Vue integration, CORS, tokens, JWT |
| 06 | Validation & Forms | Rules, file upload, custom validators, form helper |
| 07 | Shield Authentication | Session, tokens, JWT, HMAC, 2FA, groups, permissions |
| 08 | Filters & Middleware | CORS, rate limiting, auth, custom filters |
| 09 | Migrations & Seeds | Forge, foreign keys, seeders, Faker integration |
| 10 | Services & DI | Service container, custom services, events, cache |
| 11 | Testing with PHPUnit | Unit, database, feature tests, mocking |
| 12 | Docker: Apache | mod_php for legacy / simple dev |
| 13 | Docker: Nginx + PHP-FPM | Production standard, OPcache tuning |
| 14 | Docker: Caddy + PHP-FPM | Auto HTTPS, simpler config, HTTP/2-3 |
| 15 | Docker: FrankenPHP | Worker mode, embedded PHP, max performance |
| 16 | Legacy CI3 → CI4 | Migration strategies, code translation map |
| 17 | Best Practices | Naming, conventions, security, performance |
| 18 | Production Deployment | Checklist, OPcache, zero-downtime, CI/CD |

## Quick Tour

### Generate a complete REST API endpoint

When you ask the agent to *"Build a CRUD API for products"*, it will:

1. Generate a migration with proper indexes and foreign keys
2. Generate a Model with `$allowedFields`, validation rules, and casts
3. Generate an Entity (if relevant)
4. Generate a `ResponseTrait`-based API controller with:
   - Consistent envelope (`{success, data, errors, meta}`)
   - Pagination headers
   - Validation handling
   - Standard HTTP status codes
5. Add routes with the `cors` and `tokens` filters
6. Generate PHPUnit tests for each endpoint

### Choose a deployment stack

When you ask *"Dockerize for production"*, it presents the four official options:

| Option | Best For |
|--------|----------|
| **Apache + mod_php** | Legacy projects with `.htaccess`, simple dev |
| **Nginx + PHP-FPM** | Standard production, max compatibility |
| **Caddy + PHP-FPM** | Auto HTTPS, simpler config, HTTP/2-3 |
| **FrankenPHP** | Cutting-edge perf with Worker Mode |

…and generates the appropriate Dockerfile, server config, `docker-compose.yml`, and PHP tuning.

### Migrate from CodeIgniter 3

The skill includes a complete CI3 → CI4 migration guide with:

- Strategies (full rewrite vs Strangler Fig with reverse proxy)
- Code translation map (`$this->load->...` → modern equivalents)
- Auth migration to Shield (preserving existing bcrypt hashes)
- URL compatibility patterns (301 redirects, route aliases)
- Database coexistence patterns

## Why CodeIgniter 4?

- **Lightweight** — ~1.2 MB framework footprint, ~5 ms cold-start
- **PSR-compliant** — PSR-3 logger, PSR-4 autoload, PSR-7 HTTP messages
- **No vendor lock-in** — built-in Query Builder, but use any ORM you like
- **Backward-friendly** — easy to integrate into legacy PHP codebases
- **Built-in security** — CSRF, XSS, CSP, secure headers, URI filtering
- **Modular (HMVC)** — native PSR-4 modules with namespaces
- **Spark CLI** — generators, migrations, testing, custom commands
- **Multi-server** — Apache, Nginx, Caddy, FrankenPHP, LiteSpeed
- **API-ready** — `ResourceController` + `ResponseTrait` for REST APIs

## Manual Installation

If you don't want to use the `npx skills` CLI:

```bash
# Clone into your skills directory
git clone https://github.com/liusc45/CodeIgniter4-skill.git \
  ~/.config/opencode/skills/codeigniter4-specialist

# Or for Claude Code skills
git clone https://github.com/liusc45/CodeIgniter4-skill.git \
  ~/.claude/skills/codeigniter4-specialist
```

## Compatibility

- **Agent runtimes:** OpenCode, Claude Code, Cursor (any tool that supports the Skills format)
- **CodeIgniter version:** 4.5+
- **PHP version:** 8.2+ (8.1 minimum)
- **Tested with:** Shield 1.x, PHPUnit 10+, Composer 2

## Contributing

PRs welcome. Please:

1. Keep examples copy-pasteable and runnable
2. Cite the official docs when adding new content
3. Use English for all content
4. Follow the existing reference file structure

## License

MIT — see [`LICENSE`](LICENSE) for details.

## Credits

Built on top of the excellent [CodeIgniter 4](https://codeigniter.com/) framework
and the [Skills CLI](https://skills.sh/) ecosystem.
