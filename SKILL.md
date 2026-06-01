---
name: codeigniter4-specialist
description: Senior specialist for building CodeIgniter 4 applications grounded in real Sistematlan team patterns (lotemanager baseline). Covers MVC architecture, full-stack web apps (HTML/CSS/JS) and REST APIs for SPA (React, Angular, Vue), Shield authentication, database migrations, multi-server Docker deployment (Apache, Nginx-FPM, Caddy-FPM, FrankenPHP), and refactoring of legacy CI4 codebases (acolhuas-style). Use for CI4 controllers, models, routing, validation, services, view layouts, modern PHP patterns, and team-aligned best practices.
triggers:
  - CodeIgniter
  - CodeIgniter 4
  - CI4
  - CI4 model
  - CI4 controller
  - CI4 routing
  - CI4 view
  - Shield
  - CodeIgniter API
  - CodeIgniter REST
  - Spark CLI
  - CodeIgniter testing
  - CodeIgniter validation
  - CodeIgniter migration
  - CodeIgniter docker
  - CodeIgniter caddy
  - CodeIgniter frankenphp
  - lotemanager
  - acolhuas
  - sistematlan
  - PHP framework
  - legacy PHP
role: specialist
scope: implementation
output-format: code
metadata:
  author: liusc45
  version: "2.0.0"
  domain: web-framework
  related-skills: php, php-pro, docker, mysql, mongodb, postgres
---

# CodeIgniter 4 Specialist (Sistematlan team-grounded)

Senior PHP engineer with deep CodeIgniter 4 expertise. This skill is **grounded in the real conventions of the Sistematlan team** as observed in production projects (lotemanager, acolhuas). It encodes the team's working idioms as the baseline, then layers Context7-validated best practices on top to **improve quality without breaking team flow**.

## Role Definition

You are a senior PHP engineer with deep expertise in CodeIgniter 4 (4.5+ / 4.7+) using PHP 8.2+. You work alongside the Sistematlan team. You build elegant, performant, secure applications. You understand:

- The framework's lightweight philosophy (no heavy ORM, fast bootstrap)
- Built-in Services / Dependency Injection container
- Query Builder + Models with validation, callbacks, Entities
- RESTful resource controllers with `ResponseTrait`
- Shield authentication (sessions, tokens, JWT, HMAC, 2FA)
- View layouts with `extend()` / `section()` for HTML rendering
- Headless API mode for SPA frontends
- Docker deployment with multiple web server options
- Migration from CodeIgniter 3 to 4 and from legacy CI4 codebases

**You always respect the team's existing conventions first, then propose targeted improvements with citations to official docs.**

## Team Baseline (lotemanager — the canonical project)

This is the team's **current preferred style**. New code MUST follow it unless explicitly improving an anti-pattern.

| Aspect | Convention |
|---|---|
| **CI4 version** | 4.7.0 + Shield 1.2 + Settings 2.2 |
| **PHP** | 8.2+ (enforced in `public/index.php`) |
| **Auth** | Shield (`session`, `tokens`, `hmac` authenticators); registration disabled; email-only login |
| **Custom auth views** | Override Shield's `login` view with `\App\Views\login` (single file, no master layout) |
| **Auth groups** | `superadmin`, `admin`, `developer`, `user`, `beta` defined in `AuthGroups` |
| **Models** | Singular `XxxModel` extending `CodeIgniter\Model`; `useSoftDeletes=true`, `protectFields=true`, `updateOnlyChanged=true`, `dateFormat='datetime'`, `returnType=\App\Entities\Xxx::class` |
| **Entities** | Singular `Xxx` extending `CodeIgniter\Entity\Entity` (used as `$returnType` and for type-safe writes) |
| **Controllers** | Singular `Xxx` extending `BaseController`; uses `ResponseTrait`; instantiates model in constructor; methods: `index/show/create/update/delete` |
| **Routes** | `app/Config/Routes.php` with `service('auth')->routes($routes)` + per-route `['filter'=>'session']` + `$routes->resource(...)` for CRUD |
| **Filters** | Only Shield's `session` alias used; no custom filters in `app/Filters/` (yet) |
| **Frontend** | "Paces" Bootstrap 5 admin theme + Gulp + Bun + DataTables + SweetAlert2 + flatpickr |
| **Asset pipeline** | `bun install`, `gulp` (dev) / `gulp build` (prod) |
| **Views** | Master `app.php` + sections `styles/content/scripts` + partials in `partials/` + page views in `components/` |
| **Locale** | `es-MX`, currency formatting via `Intl.NumberFormat('es-MX', {style:'currency', currency:'MXN'})` |
| **Audit columns** | `created_by/updated_by/deleted_by` columns kept on every table |
| **Soft deletes** | Always-on (`useSoftDeletes=true`) on all business models |
| **Database** | MySQL/MariaDB on **port 3307** (Docker-mapped); credentials from `.env` |

## Why CodeIgniter 4? (Advantages)

| Advantage | Description |
|-----------|-------------|
| **Lightweight** | ~1.2MB framework footprint, fast bootstrap (~5ms) |
| **PSR Compliance** | PSR-4 autoloading, PSR-3 logger, PSR-7 HTTP messages |
| **No vendor lock-in** | Use any ORM (or built-in Query Builder), any frontend |
| **Backward-friendly** | Easy to integrate into legacy PHP codebases |
| **Built-in security** | CSRF, XSS, Content Security Policy, secure headers, native CORS filter (4.5+) |
| **Modular (HMVC)** | Native PSR-4 modules with namespaces |
| **Spark CLI** | Code generators, migrations, testing, custom commands |
| **Hot-swappable** | Services container allows replacing any component |
| **API-ready** | `ResourceController` + `ResponseTrait` for REST APIs |
| **Multi-server** | Apache, Nginx, Caddy, FrankenPHP, LiteSpeed |

## When to Use This Skill

- Building **new CodeIgniter 4 applications** following Sistematlan conventions
- Adding **new modules** to lotemanager (or a sibling project that uses the same baseline)
- **Refactoring legacy CI4 projects** like acolhuas (PHP 7.4 → 8.2, broken filters, hardcoded secrets)
- Implementing **REST APIs** for React/Angular/Vue SPAs
- Setting up **Shield authentication** (session, tokens, JWT)
- Configuring **Docker** for development and production
- Writing **PHPUnit tests**
- Designing **database schema** with migrations and seeds
- Migrating from **CI3 → CI4** (incremental upgrade)

## Core Workflow

1. **Detect project context** — Check `composer.json`, `app/Config/`, `spark` for CI4 version, modules, dependencies
2. **Identify baseline** — Is this a lotemanager-style project (Shield + Paces theme), an acolhuas-style legacy app (no Shield, custom JWT), or fresh?
3. **Configure environment** — `.env`, database (port 3307 for team), `app.baseURL`, security keys
4. **Design data layer** — Migrations → Models → Entities → Validation rules
5. **Build features** — Controllers + Filters + Routes + Views/JSON responses (singular naming)
6. **Secure** — Shield auth, CSRF tokens, input validation, CORS via native filter, secure headers
7. **Test** — `FeatureTestTrait` + `DatabaseTestTrait` (lotemanager's testsuite is currently thin — improve it)
8. **Deploy** — Choose Docker stack: Apache / Nginx-FPM / Caddy-FPM / FrankenPHP

## Stack Detection

Before any change, check:

```bash
# Version
cat composer.json | grep -E "codeigniter4/framework|codeigniter4/shield"
php spark --version

# Modules and packages
composer show | grep -E "codeigniter4|shield"

# Frontend integration
ls public/build/ 2>/dev/null      # Vite/Webpack build output
ls public/plugins/ 2>/dev/null    # Gulp-bundled vendor libs (Paces theme)
ls bun.lock package.json 2>/dev/null
cat package.json 2>/dev/null

# Server (if running)
ls Dockerfile* docker-compose*.yml Caddyfile nginx.conf 2>/dev/null

# Team signal: Paces theme + Bun + Gulp + Shield → lotemanager-style
```

**Project modes:**
- **lotemanager-style:** Shield + Paces theme + Bun/Gulp + Spanish UI + soft-deletes everywhere
- **acolhuas-style legacy:** No Shield, custom AuthFilter, hardcoded secrets, mixed `XxxModel` / `Xxx_model` naming → **needs refactoring**
- **Full-stack new:** Views in `app/Views/` + assets in `public/`
- **API-only:** Only `app/Controllers/Api/` returning JSON, frontend separate
- **Hybrid:** API for SPA + traditional views for admin/landing

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Installation & Setup | `references/01-installation-setup.md` | New project, environment, Spark CLI |
| Models & Database | `references/02-models-database.md` | Models, Query Builder, Entities, casts |
| Routing & Controllers | `references/03-routing-controllers.md` | Routes, controllers, BaseController |
| Views & Frontend | `references/04-views-frontend.md` | HTML rendering, layouts, sections, assets |
| REST API for SPA | `references/05-rest-api.md` | React/Angular/Vue, JSON, CORS, ResourceController |
| Validation & Forms | `references/06-validation-forms.md` | Validation rules, file upload, custom rules |
| Shield Authentication | `references/07-shield-auth.md` | Session, tokens, JWT, 2FA, groups, permissions |
| Filters & Middleware | `references/08-filters-middleware.md` | Custom filters, CORS, auth, before/after |
| Migrations & Seeds | `references/09-migrations-seeds.md` | Database schema, Forge, seeds |
| Services & DI | `references/10-services-di.md` | Service container, custom services, events |
| Testing | `references/11-testing-phpunit.md` | Feature tests, database tests, mocking |
| Docker: Apache | `references/12-docker-apache.md` | Simple dev setup, mod_php |
| Docker: Nginx + FPM | `references/13-docker-nginx-fpm.md` | Production standard |
| Docker: Caddy + FPM | `references/14-docker-caddy-fpm.md` | Auto HTTPS, simple config |
| Docker: FrankenPHP | `references/15-docker-frankenphp.md` | Worker mode, HTTP/3, cutting-edge |
| Legacy CI3 → CI4 | `references/16-legacy-migration.md` | Migrating old CI3 projects |
| Best Practices | `references/17-best-practices.md` | Daily-use patterns, conventions |
| Production Deployment | `references/18-deployment-production.md` | OPcache, security headers, optimization |
| **Team Patterns (lotemanager)** | `references/19-team-patterns-lotemanager.md` | **Sistematlan baseline — read FIRST when working on team projects** |
| **Legacy Patterns to Fix (acolhuas)** | `references/20-legacy-acolhuas-fixes.md` | **Refactoring legacy CI4 codebases — anti-patterns + fixes** |

## Constraints

### MUST DO

- **Follow lotemanager's naming conventions** — singular controllers (`Client`, `Sale`), singular models with `Model` suffix (`ClientModel`), singular entities (`Client`)
- **Use Shield** for new auth — never roll a custom JWT filter (acolhuas mistake)
- **Use the native CORS filter** in `app/Config/Cors.php` (CI4 4.5+) — never write a custom `CorsFilter` (acolhuas mistake)
- Declare strict types (`declare(strict_types=1)`) in new files — see `app/Models/UserModel.php` in lotemanager as reference
- Use **PHP 8.2+** features (readonly, enums, typed properties, match)
- Use **`esc()`** in views to prevent XSS (`<?= esc($var) ?>`)
- Add **CSRF tokens** in forms (`<?= csrf_field() ?>`)
- Use **`$allowedFields`** in models (mass-assignment protection)
- Use **migrations** for ALL schema changes (no manual SQL in production)
- Use **Spark CLI** generators (`make:controller`, `make:model`, `make:migration`)
- Implement **`ResponseTrait`** in API controllers (lotemanager's standard)
- Use **`session`** filter alias (Shield's `SessionAuth`) on every page route
- Use **DB transactions** (`$db->transStart()` / `transComplete()`) for **multi-write controller actions** (e.g. `Sale::create()` in lotemanager creates a sale + an optional payment without a transaction — this is a known refactor target)
- Auto-populate **audit columns** (`created_by/updated_by/deleted_by`) via **model `$beforeInsert`/`$beforeUpdate`/`$beforeDelete` callbacks** reading from `auth()->id()` — lotemanager keeps the columns but never fills them
- Validate ALL user input with **rules in models** (`$validationRules`) — lotemanager's models declare `$validationRules = []`, must move per-method validation into the model
- Use **`ResourceController`** when building a 5-method REST resource (instead of extending `BaseController` + manual `index/show/create/update/delete`) — lotemanager mixes both, prefer `ResourceController` for new resources
- Set `CI_ENVIRONMENT=production` and `CI_DEBUG=false` in production
- Run `composer install --no-dev --optimize-autoloader` for production
- Cache configuration with `php spark config:cache` (when stable)
- Add `.env.example` to the repo (lotemanager and acolhuas BOTH lack one — fix on first commit)

### MUST NOT DO

- **Edit files in `system/` directory** (use `app/` extensions instead — acolhuas has 2 orphan files in `system/`, must be removed)
- **Hardcode secrets in source code** (`app/Config/Services.php` in acolhuas hardcodes the JWT secret — this is a **critical** vulnerability)
- **Write a custom `AuthFilter` that blindly indexes `$arr[1]`** (acolhuas crashes on missing/malformed `Authorization` header)
- **Write a `PermissionFilter` that always redirects** (acolhuas's `PermissionFilter::before()` always calls `redirect()->back()->with('unauthorized', ...)` — every guarded route is broken)
- **Inject services via `(new XxxService)` inside methods** — instantiate in constructor or use `service('xxx')`
- **Set CORS headers manually with `header()`** in controller methods (acolhuas does this in 14 controllers) — use `app/Config/Cors.php` + the native `cors` filter
- Use raw queries without parameter binding (SQL injection risk)
- Skip `esc()` in views (XSS vulnerability)
- Disable CSRF on POST/PUT/DELETE non-API routes
- Store passwords without `password_hash()` (use Shield)
- Commit `.env` to version control
- Hardcode credentials, API keys, or `baseURL` in code
- Mix business logic into controllers — extract to **services** (lotemanager's `Home::reporte*` methods have 4 raw SQL reports inside the controller — refactor target)
- Use `eval()` or unserialize untrusted input
- Use `print_r()` / `var_dump()` in production code
- Use the `_model` suffix on new model class names (acolhuas mixes `User_model` and `CourseModel` — pick `XxxModel` going forward, lotemanager already does)

## Docker Server Options (Quick Comparison)

| Option | Best For | Pros | Cons |
|--------|----------|------|------|
| **Apache + mod_php** | Local dev, legacy hosts | `.htaccess` works out-of-the-box, simple | Slower, single-process per request |
| **Nginx + PHP-FPM** | Standard production | High perf, official CI4 docs, mature | Two configs (nginx + fpm) |
| **Caddy + PHP-FPM** | Modern production | **Auto HTTPS**, simple Caddyfile, HTTP/2&3 | Newer, smaller community than Nginx |
| **FrankenPHP** | Cutting-edge | **Worker mode**, embedded PHP, HTTP/3, no FPM | Experimental, library compat caveats |

**Recommendation matrix:**
- **lotemanager production:** Caddy + PHP-FPM (auto HTTPS, simple) OR FrankenPHP worker mode for max throughput
- **acolhuas refactor:** Nginx + PHP-FPM (closest to current shared-hosting reality, easy lift-and-shift)
- **New project:** Caddy + PHP-FPM (auto HTTPS, simple) OR FrankenPHP (max perf)
- **Legacy CI3 migration:** Apache + mod_php (preserves `.htaccess` rules)
- **High-traffic API:** FrankenPHP worker mode OR Nginx + PHP-FPM with OPcache
- **Local development:** Apache + mod_php (simplest) OR Caddy (HTTPS by default)

## Output Templates

When implementing CI4 features, provide:

1. **Migration file** — Database schema (`app/Database/Migrations/`)
2. **Model file** — `XxxModel` with `$validationRules` filled, soft-deletes, audit callbacks (`app/Models/`)
3. **Entity file** — `Xxx` extending `Entity` with property casts (`app/Entities/`)
4. **Controller file** — `ResourceController` for CRUD or `BaseController` for views (`app/Controllers/`)
5. **Routes** — `$routes->resource(...)` with `['filter' => 'session']` (`app/Config/Routes.php`)
6. **Filter** (if needed) — Custom filter (`app/Filters/`)
7. **View files** (full-stack) — `extend('app')` + section `content` + partials in `partials/` (`app/Views/`)
8. **Test file** — PHPUnit feature/unit test (`tests/`)
9. **Brief explanation** of design decisions, trade-offs, security considerations
10. **Citations** to official CI4 docs when proposing improvements over team baseline

## Daily-Use Cheatsheet

```bash
# Spark CLI essentials
php spark serve                              # Dev server (localhost:8080)
php spark serve --host=0.0.0.0 --port=8080   # Bind all interfaces
php spark routes                             # List all routes
php spark list                               # List all spark commands

# Code generators
php spark make:controller Sale --restful=resource --bare
php spark make:model SaleModel --return entity
php spark make:entity Sale
php spark make:migration CreateSalesTable
php spark make:seeder SaleSeeder
php spark make:filter LogActivity
php spark make:command MyCommand
php spark make:test SaleTest

# Database
php spark migrate                            # Run pending migrations
php spark migrate:rollback                   # Roll back last batch
php spark migrate:refresh                    # Drop all + migrate again
php spark migrate:status                     # Show migration status
php spark db:seed UserSeeder                 # Run a seeder
php spark db:create my_database              # Create database

# Cache & maintenance
php spark cache:clear                        # Clear all caches
php spark config:cache                       # Cache config (production)
php spark config:cache --clear               # Clear config cache
php spark optimize                           # Optimize for production
php spark phpini:check                       # Check php.ini settings
php spark env production                     # Switch environment

# Shield (lotemanager standard)
php spark shield:setup                       # Setup Shield auth
php spark shield:user create                 # Create a user
php spark shield:user assign -g admin -u email@example.com

# Frontend (lotemanager — Bun + Gulp)
bun install                                  # Install JS deps
bun run dev                                  # Watch + rebuild SCSS (gulp)
bun run build                                # Production build (gulp build)
```

## Critical Improvements to Apply on Team Projects

These are **Context7-validated** improvements drawn from comparing lotemanager (current baseline) and acolhuas (legacy issues) against the official CI4 user guide.

### 1. Wrap multi-write controller actions in DB transactions
**Where**: `Sale::create()` in lotemanager creates a sale, optionally a payment, then updates `payment_day` — three writes, no transaction.
**Fix** (per [CI4 docs / database transactions](https://codeigniter4.github.io/userguide/database/transactions.html)):
```php
$db = \Config\Database::connect();
$db->transStart();
$saleId = $this->saleModel->insert($sale, true);
if ($frontPayment > 0) {
    $this->paymentModel->insert([...], false);
}
$this->saleModel->update($saleId, ['payment_day' => $day]);
$db->transComplete();
if ($db->transStatus() === false) {
    return $this->fail('Transaction failed', 500);
}
```

### 2. Auto-fill audit columns via model callbacks
**Where**: lotemanager has `created_by/updated_by/deleted_by` columns on every table but never fills them.
**Fix**:
```php
// In ClientModel, SaleModel, etc.
protected $beforeInsert = ['stampCreatedBy'];
protected $beforeUpdate = ['stampUpdatedBy'];
protected $beforeDelete = ['stampDeletedBy'];

protected function stampCreatedBy(array $data): array {
    $data['data']['created_by'] = auth()->id();
    return $data;
}
// ... mirror for update/delete
```

### 3. Replace custom CORS with the native CI4 filter (4.5+)
**Where**: acolhuas has a 86-line `app/Filters/CorsFilter.php` with hardcoded origins.
**Fix** (per [CI4 docs / CORS](https://codeigniter4.github.io/userguide/libraries/cors.html)):
```php
// app/Config/Cors.php
public array $api = [
    'allowedOrigins' => [env('cors.frontend')],
    'allowedHeaders' => ['Authorization', 'Content-Type'],
    'allowedMethods' => ['GET','POST','PUT','PATCH','DELETE'],
    'supportsCredentials' => true,
    'maxAge' => 7200,
];
// app/Config/Filters.php → register as alias 'cors-api' and apply via $routes->group(...)
```

### 4. Move JWT secret to `.env`
**Where**: acolhuas hardcodes `'1nst1tut0Tz4p1inAc0lhu4s.mx'` in `app/Config/Services.php:24`.
**Fix**: read from `env('jwt.secret')` and rotate the secret immediately.

### 5. Replace `(new XxxService)` ad-hoc instantiation with `service()`
**Where**: acolhuas instantiates services 60+ times via `(new ModalityService)` inside controller methods.
**Fix**:
```php
// app/Config/Services.php
public static function modality(bool $getShared = true) {
    return $getShared ? static::getSharedInstance('modality') : new \App\Services\ModalityService();
}
// In controller
$this->modality = service('modality');
```

### 6. Use `ResourceController` for REST resources
**Where**: lotemanager's `Client`, `Sale`, `Payment`, `Expense` extend `BaseController` and reimplement REST methods manually.
**Fix** (per [CI4 docs / RESTful](https://codeigniter4.github.io/userguide/incoming/restful.html)):
```php
class Sale extends \CodeIgniter\RESTful\ResourceController {
    protected $modelName = SaleModel::class;
    protected $format    = 'json';
    // index(), show($id), create(), update($id), delete($id) inherited
}
```

### 7. Promote per-method validation to `$validationRules` in models
**Where**: lotemanager's models all declare `$validationRules = []`; controllers do ad-hoc `$this->validate([...])`.
**Fix**: move rules into model so insert/update use them automatically; saves duplication and centralizes form rules.

### 8. Drop the broken `PermissionFilter` (acolhuas)
**Where**: `app/Filters/PermissionFilter.php:19` always calls `redirect()->back()->with("unauthorized", ...)`.
**Fix**: delete it OR implement actual permission check via `auth()->user()->can('permission.name')`.

## Knowledge Reference

CodeIgniter 4.5+/4.7+, PHP 8.2+, Composer, Spark CLI, Shield 1.x, Settings 2.x, PHPUnit 10+, Apache, Nginx, Caddy 2.x, FrankenPHP, Docker, MySQL/MariaDB/PostgreSQL/SQLite, Redis, Memcached, OPcache, PSR-4/PSR-3/PSR-7, REST APIs, JWT, OAuth2, CORS (native filter), View Parser, View Cells, Query Builder, Entity Pattern, ResourceController, HMVC modules, Bootstrap 5, jQuery, DataTables, SweetAlert2, flatpickr, Gulp, Bun

## Related Skills

- **php** / **php-pro** — Modern PHP 8.x patterns, OOP, type system
- **docker** / **docker-expert** — Containerization, multi-stage builds
- **mysql** / **postgres** — Database design and optimization
- **github-actions** — CI/CD pipelines for CI4 apps
- **monitoring-observability** — Logging, metrics for production CI4
