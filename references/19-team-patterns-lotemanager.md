# Reference 19 — Sistematlan Team Patterns (lotemanager baseline)

> **Read this FIRST when working on Sistematlan CI4 projects.**
> This document encodes the team's existing working style as observed in `lotemanager` (production). All new code should follow these conventions unless the user explicitly approves a deviation. Improvements over this baseline are listed in `SKILL.md → Critical Improvements to Apply on Team Projects` and grounded in Context7-validated CI4 official docs.

## 1. Stack snapshot

| Layer | Choice |
|---|---|
| CI4 | `codeigniter4/framework ^4` (locked at **4.7.0**) |
| Auth | `codeigniter4/shield ^1.2` (locked at **1.2.0**) |
| Settings | `codeigniter4/settings ^2.2` |
| PHP | **8.2+** (enforced in `public/index.php:13` and `spark:39`; **note:** `composer.json` still says `^8.1` — bump on next composer update) |
| DB | MySQL/MariaDB on **port 3307** (Docker-mapped, non-default) |
| Frontend | "Paces" Coderthemes admin (Bootstrap 5 + Tabler icons) |
| JS deps | **Bun** (`bun.lock` — not npm/yarn) |
| Asset bundler | **Gulp 4** (`gulpfile.js`) + custom `plugins.config.js` |
| Locale | `es-MX` (Mexican Spanish) |
| Currency | MXN, formatted via `Intl.NumberFormat('es-MX', {style:'currency', currency:'MXN'})` |

## 2. Directory layout

```
app/
├── Common.php                       # empty stub (placeholder)
├── Config/                          # 17 config files + Boot/{development,production,testing}.php
│   ├── Auth.php                     # extends Shield Auth — registration disabled, email-only login
│   ├── AuthGroups.php               # extends Shield AuthGroups — superadmin/admin/developer/user/beta
│   ├── AuthToken.php                # extends Shield AuthToken — Authorization header, 1-year token life
│   ├── Boot/{development,production,testing}.php
│   ├── ContentSecurityPolicy.php
│   ├── Cors.php                     # CI4 native CORS (4.5+)
│   ├── Database.php                 # MySQLi @ port 3307
│   ├── Email.php
│   ├── Events.php                   # default + debug toolbar
│   ├── Feature.php
│   ├── Filters.php                  # only registers framework aliases — no custom filters
│   ├── Modules.php
│   ├── Routes.php                   # service('auth')->routes($routes) + per-route session filter
│   ├── Security.php                 # CSRF, randomized tokens
│   ├── Session.php                  # FileHandler, 7200s, cookie 'ci_session'
│   └── Validation.php
├── Controllers/
│   ├── BaseController.php           # abstract, empty initController, $helpers = []
│   ├── Home.php                     # dashboard + 4 reports + page routers
│   ├── Client.php  Sale.php  Payment.php  Expense.php  User.php
├── Database/
│   ├── Migrations/                  # 6 files dated 2026-02-*
│   └── Seeds/                       # empty
├── Entities/                        # Client, Sale, Payment, Expense (extend Entity)
├── Filters/                         # EMPTY (Shield's session alias used inline in routes)
├── Helpers/                         # EMPTY
├── Language/en/Validation.php       # empty array (placeholder)
├── Libraries/                       # EMPTY
├── Models/                          # ClientModel, SaleModel, PaymentModel, ExpenseModel, UserModel (extends Shield's)
├── Services/                        # EMPTY (no app-level services yet)
├── ThirdParty/                      # EMPTY
└── Views/
    ├── app.php                      # master layout
    ├── login.php                    # standalone Shield login override
    ├── components/                  # page views
    ├── partials/                    # head-css, topbar, sidenav, page-title, footer, footer-scripts, toast, customizer
    └── modals/editClient.php

public/
├── index.php
├── css/{app.css, app.min.css, vendors.min.css, bootstrap/, fonts/}
├── js/{app.js, config.js, vendors.min.js}
├── plugins/                         # 37 vendored libraries copied by Gulp
└── scss/                            # SCSS sources

builds/                              # phpunit cache + coverage HTML
writable/{cache, debugbar, logs, session, uploads}/
tests/                               # _support + database + session + unit + (no feature tests yet)

bun.lock
gulpfile.js
package.json
phpunit.xml.dist
preload.php                          # CI4 sample, untouched
spark
```

## 3. Naming conventions

| Element | Convention | Example |
|---|---|---|
| Controller | Singular PascalCase | `Client`, `Sale`, `Payment` |
| Model | Singular + `Model` suffix | `ClientModel`, `SaleModel` |
| Entity | Singular PascalCase | `Client`, `Sale` |
| Table | Lowercase plural | `clients`, `sales`, `payments` |
| Primary key | `id` (auto-increment, INT(5) UNSIGNED) |
| FK | Singular noun | `client` (FK to `clients.id`), `sale` (FK to `sales.id`) |
| Audit fields | Always present | `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `deleted_by` |
| Migration filename | `YYYY-MM-DD-HHMMSS_DescriptiveName.php` | `2026-02-11-190607_AddSaleTable.php` |

**Do NOT use:**
- Underscored model names like `User_model` (legacy acolhuas style)
- Plural controller names like `Clients`

## 4. BaseController convention

```php
<?php
namespace App\Controllers;

use CodeIgniter\Controller;
use CodeIgniter\HTTP\CLIRequest;
use CodeIgniter\HTTP\IncomingRequest;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use NumberFormatter;
use Psr\Log\LoggerInterface;

abstract class BaseController extends Controller
{
    /** @var CLIRequest|IncomingRequest */
    protected $request;
    protected $helpers = [];
    private NumberFormatter $formatter;

    public function initController(RequestInterface $request, ResponseInterface $response, LoggerInterface $logger)
    {
        parent::initController($request, $response, $logger);
        // No app-level service preload yet.
    }
}
```

> **Improvement opportunity** (Context7-validated): preload `'url'`, `'form'` helpers and the session service here. See `SKILL.md → Critical Improvements`.

## 5. Controller convention (resource style)

```php
<?php
namespace App\Controllers;

use App\Entities\Client;
use App\Models\ClientModel;
use CodeIgniter\API\ResponseTrait;
use CodeIgniter\HTTP\ResponseInterface;
use CodeIgniter\I18n\Time;

class Client extends BaseController
{
    use ResponseTrait;

    protected ClientModel $clientModel;

    public function __construct()
    {
        $this->clientModel = new ClientModel();
    }

    public function index(): ResponseInterface
    {
        return $this->respond($this->clientModel->findAll());
    }

    public function show($id): ResponseInterface
    {
        return $this->response->setJSON($this->clientModel->find($id));
    }

    public function create(): ResponseInterface
    {
        $entity = new Client($this->request->getPost());
        try {
            $ok = $this->clientModel->save($entity);
        } catch (\ReflectionException $e) {
            return $this->fail($e->getMessage());
        }
        return $ok
            ? $this->response->setJSON($this->clientModel->find($this->clientModel->getInsertID()))
            : $this->fail('Save failed', ResponseInterface::HTTP_INTERNAL_SERVER_ERROR);
    }

    public function update($id): ResponseInterface
    {
        $client = $this->clientModel->find($id);
        if (!$client) return $this->failNotFound();
        $client->fill($this->request->getRawInput());
        $client->updated_at = Time::now();
        try {
            $this->clientModel->save($client);
        } catch (\ReflectionException $e) {
            return $this->fail($e->getMessage());
        }
        return $this->response->setJSON($this->clientModel->find($id));
    }

    public function delete($id): ResponseInterface
    {
        try {
            $this->clientModel->delete($id);
        } catch (\ReflectionException $e) {
            return $this->fail($e->getMessage());
        }
        return $this->respondDeleted(['id' => $id]);
    }
}
```

**Notes:**
- Always `use ResponseTrait` for JSON endpoints.
- Catch `\ReflectionException` thrown by Entity hydration when unknown fields are POSTed.
- For new resources prefer `extends \CodeIgniter\RESTful\ResourceController` and skip the boilerplate (Context7-validated improvement).

## 6. Model convention

```php
<?php
declare(strict_types=1);

namespace App\Models;

use App\Entities\Client;
use CodeIgniter\Model;

class ClientModel extends Model
{
    protected $table            = 'clients';
    protected $primaryKey       = 'id';
    protected $useAutoIncrement = true;

    protected $returnType       = Client::class;
    protected $useSoftDeletes   = true;

    protected $protectFields    = true;
    protected $allowedFields    = [
        'name', 'last_name', 'phone', 'email',
        'created_at', 'created_by',
        'updated_at', 'updated_by',
        'deleted_at', 'deleted_by',
    ];

    protected bool $allowEmptyInserts = false;
    protected bool $updateOnlyChanged = true;

    protected $useTimestamps = true;
    protected $dateFormat    = 'datetime';
    protected $createdField  = 'created_at';
    protected $updatedField  = 'updated_at';
    protected $deletedField  = 'deleted_at';

    protected $validationRules    = [];
    protected $validationMessages = [];
    protected $skipValidation     = false;

    protected $allowCallbacks = true;
    protected $beforeInsert   = ['toLower'];
    protected $afterInsert    = [];
    protected $beforeUpdate   = [];
    protected $afterUpdate    = [];
    protected $beforeFind     = [];
    protected $afterFind      = [];
    protected $beforeDelete   = [];
    protected $afterDelete    = [];

    protected function toLower(array $data): array
    {
        if (isset($data['data']['name'])) {
            $data['data']['name'] = ucfirst(strtolower($data['data']['name']));
        }
        if (isset($data['data']['last_name'])) {
            $data['data']['last_name'] = ucfirst(strtolower($data['data']['last_name']));
        }
        return $data;
    }
}
```

## 7. Routes convention

```php
<?php
use CodeIgniter\Router\RouteCollection;

/** @var RouteCollection $routes */

$filter = ['filter' => 'session'];

service('auth')->routes($routes);              // Mounts Shield's named routes

// Pages (rendered HTML)
$routes->get('/',          'Home::index',     $filter);
$routes->get('/clients',   'Home::clients',   $filter);
$routes->get('/sales',     'Home::sales',     $filter);
$routes->get('/payments',  'Home::payments',  $filter);
$routes->get('/expenses',  'Home::expenses',  $filter);
$routes->get('/sale/(:num)', 'Home::saleDetail/$1', $filter);

// REST resources (JSON)
$routes->get('/sale/user/(:any)', 'Sale::byClient/$1', $filter);
$routes->resource('client',   ['filter' => 'session']);
$routes->resource('sale',     ['filter' => 'session']);
$routes->resource('payment',  ['filter' => 'session']);
$routes->resource('expense',  ['filter' => 'session']);
$routes->resource('user',     ['filter' => 'session']);
```

**Notes:**
- All routes pass through Shield's `session` filter. Public routes are not allowed; everything is gated.
- `$routes->resource('foo')` auto-generates the 5 REST endpoints — controller method names match.
- Custom routes (like `/sale/user/(:any)`) are declared **before** the resource line so they take precedence.

## 8. Filters convention

`app/Config/Filters.php`:

```php
public array $aliases = [
    'csrf'          => CSRF::class,
    'toolbar'       => DebugToolbar::class,
    'honeypot'      => Honeypot::class,
    'invalidchars'  => InvalidChars::class,
    'secureheaders' => SecureHeaders::class,
    'cors'          => Cors::class,                  // CI4 4.5+ native
    // Shield registers: session, tokens, hmac, chain, jwt, auth-rates, group, permission, force-reset
];

public array $required = [
    'before' => ['forcehttps', 'pagecache'],
    'after'  => ['pagecache', 'performance', 'toolbar'],
];

public array $globals = [
    'before' => [],   // CSRF, secureheaders not active globally — apply per-route
    'after'  => [],
];

public array $filters = [];
```

`session` is applied **inline per route**. CSRF is currently NOT enforced globally — when enabling for non-API routes, configure with `except` patterns for `/api/*`.

## 9. View layout convention

`app/Views/app.php` (master layout):

```php
<!DOCTYPE html>
<html lang="es">
<head>
    <?= view('partials/title-meta', ['title' => $title ?? '']) ?>
    <?= view('partials/head-css') ?>
    <?= $this->renderSection('styles') ?>
</head>
<body>
    <?= view('partials/topbar') ?>
    <?= view('partials/sidenav') ?>
    <?= view('partials/toast') ?>
    <main>
        <?= view('partials/page-title', ['title' => $title ?? '', 'subtitle' => $subtitle ?? '']) ?>
        <?= $this->renderSection('content') ?>
    </main>
    <?= view('partials/footer') ?>
    <?= view('partials/customizer') ?>
    <?= view('partials/footer-scripts') ?>

    <script>
        // Global currency formatter, used by per-page DataTables renderers
        const formatter = new Intl.NumberFormat('es-MX', { style: 'currency', currency: 'MXN' });
        // SweetAlert + Toast helpers (showToast, deleteThing, editThing, wip)
    </script>
    <?= $this->renderSection('scripts') ?>
</body>
</html>
```

Pages extend like this:

```php
<?= $this->extend('app') ?>

<?= $this->section('styles') ?>
<link rel="stylesheet" href="/plugins/datatables/datatables.min.css">
<?= $this->endSection() ?>

<?= $this->section('content') ?>
<div class="container">
    <h2><?= esc($title) ?></h2>
    <table id="clients-table" class="table"></table>
</div>
<?= $this->endSection() ?>

<?= $this->section('scripts') ?>
<script src="/plugins/datatables/datatables.min.js"></script>
<script>
    const dt_table = $('#clients-table').DataTable({
        ajax: '/client',
        columns: [
            { data: 'id' },
            { data: 'name' },
            { data: 'email' },
            // ...
        ]
    });
</script>
<?= $this->endSection() ?>
```

## 10. Frontend toolchain (Bun + Gulp)

`package.json` scripts:
```json
{
  "scripts": {
    "dev": "gulp",
    "build": "gulp build",
    "rtl": "gulp rtl",
    "rtl-build": "gulp rtlBuild"
  }
}
```

Workflow:
```bash
bun install            # install JS deps
bun run dev            # watches SCSS and rebuilds
bun run build          # one-shot production build
```

**Gulp tasks:**
- `default` → `plugins → styles + watch`
- `build` → `plugins → styles` (one-shot)
- `rtl` / `rtlBuild` → mirror with `gulp-rtlcss`

**SCSS pipeline:** dart-sass → autoprefixer → pixrem → optional cssnano → `*.min.css`.

**Plugin pipeline:** `plugins.config.js` lists 37 vendor libs (jquery, datatables, apexcharts, sweetalert2, fullcalendar, quill, leaflet, flatpickr, choices.js, …) → copies assets to `public/plugins/{name}/` → concatenates JS into `public/js/vendors.min.js` and CSS into `public/css/vendors.min.css`.

## 11. Auth convention (Shield)

```php
// app/Config/Auth.php
public array $views = [
    'login' => '\App\Views\login',                // custom login view
    // Other views fall back to Shield defaults.
];

public array $redirects = [
    'register'           => '/',
    'login'              => '/',
    'logout'             => 'login',
    'force_reset'        => '/',
    'permission_denied'  => '/',
    'group_denied'       => '/',
];

public array $authenticators = [
    'tokens'  => AccessTokens::class,
    'session' => Session::class,
    'hmac'    => HmacSha256::class,
    // 'jwt' => JWT::class,                        // commented out
];

public string $defaultAuthenticator = 'session';
public array $authenticationChain   = ['session', 'tokens', 'hmac'];

public bool $allowRegistration = false;            // disabled — only seeded users
public bool $allowMagicLinkLogins = false;
public bool $recordActiveDate = true;

public array $sessionConfig = [
    'field'              => 'user',
    'allowRemembering'   => true,
    'rememberCookieName' => 'remember',
    'rememberLength'     => 15 * DAY,
];

public int $minimumPasswordLength = 8;
public array $passwordValidators = [
    'CodeIgniter\Shield\Authentication\Passwords\CompositionValidator',
    'CodeIgniter\Shield\Authentication\Passwords\NothingPersonalValidator',
    'CodeIgniter\Shield\Authentication\Passwords\DictionaryValidator',
];
public array $validFields = ['email'];             // username login disabled
```

`app/Models/UserModel.php` extends Shield's `UserModel` and only overrides `initialize()` as a hook for future custom fields.

## 12. Database convention

`app/Config/Database.php`:

```php
public array $default = [
    'DSN'      => '',
    'hostname' => 'localhost',
    'username' => '',                              // from .env: database.default.username
    'password' => '',
    'database' => '',
    'DBDriver' => 'MySQLi',
    'DBPrefix' => '',
    'pConnect' => false,
    'DBDebug'  => true,
    'charset'  => 'utf8mb4',
    'DBCollat' => 'utf8mb4_general_ci',
    'swapPre'  => '',
    'encrypt'  => false,
    'compress' => false,
    'strictOn' => false,
    'failover' => [],
    'port'     => 3307,                            // ⚠ non-default port — Docker mapping
];
```

`.env` keys (NOT in repo — must be created):
```bash
CI_ENVIRONMENT = development
app.baseURL = "http://localhost:8080/"
app.indexPage = ""
app.cspEnabled = false

database.default.hostname = localhost
database.default.database = lotemanager
database.default.username = root
database.default.password = root
database.default.DBDriver = MySQLi
database.default.DBPrefix =
database.default.port = 3307

email.fromEmail = noreply@lotemanager.local
email.fromName  = "LoteManager"
```

## 13. Reports convention

`Home::reporte*` methods produce server-rendered tables (NOT JSON DataTables), filtered via GET params, validated with regex/`ctype_digit`. Patterns observed:

- **Cobranza** (collections aging): `DATEDIFF(NOW(), order_date)` bucketed at 0-30 / 31-60 / 61-90 / +90 días, with `WHERE DATE_FORMAT(payment_date, '%Y-%m') = ?` filter.
- **Financiero** (income vs expenses): 4 raw queries — `SUM(amount)` by month, `payment_method` breakdowns. `LIMIT 12` for trailing year.
- **Clientes** (customer statement): per-sale rows + `LEFT JOIN payments` with `COALESCE(SUM(p.amount), 0) AS total_pagos`.
- **Vendedores** (sales-by-user): `GROUP BY created_by` + 6-month rolling trend.

> **Improvement opportunity:** extract these reports to dedicated `App\Services\ReportService` to remove SQL from the controller layer. See `SKILL.md → Critical Improvements`.

## 14. Testing convention

```xml
<!-- phpunit.xml.dist -->
<testsuites>
    <testsuite name="App">
        <directory>./tests</directory>
    </testsuite>
</testsuites>
```

`tests/`:
- `unit/HealthTest.php` — config sanity check (`APPPATH` defined, `app.baseURL` is a valid URL)
- `database/ExampleDatabaseTest.php` — `DatabaseTestTrait` example with seeder
- `session/ExampleSessionTest.php` — session helpers
- `_support/{Database/{Migrations,Seeds},Libraries,Models}` — fixtures

> **Improvement opportunity:** add **feature tests** for the resource controllers (`Client`, `Sale`, `Payment`, `Expense`) using `FeatureTestTrait`. Currently there are none — tests/coverage are thin.

## 15. Quick "do this on team projects" checklist

When adding a new business resource to lotemanager (or a sibling project):

```bash
# 1. Migration
php spark make:migration CreateOrdersTable

# 2. Entity
php spark make:entity Order

# 3. Model (with --return entity flag so $returnType is set)
php spark make:model OrderModel --return entity

# 4. Controller (resource-style)
php spark make:controller Order --restful=resource --bare

# 5. Wire route
# in app/Config/Routes.php:
# $routes->resource('order', ['filter' => 'session']);

# 6. Add a page route + view (if it has a UI)
# $routes->get('/orders', 'Home::orders', $filter);
# Create app/Views/components/orders.php extending 'app'.
# Create entry in partials/sidenav.php.

# 7. Wire DataTable in the view
# In components/orders.php: <table id="orders-table"> + JS dt = $('#orders-table').DataTable({ ajax: '/order' })

# 8. Run migration
php spark migrate

# 9. Add a feature test
php spark make:test OrderTest
```

## 16. Known gaps (refactor backlog)

These are the team's current technical-debt items observed in lotemanager. Address them when touching adjacent code:

1. **No DB transaction in `Sale::create()`** (multi-write — sale + payment + payment_day update)
2. **Audit columns never auto-filled** (`created_by/updated_by/deleted_by` stay NULL)
3. **`$validationRules = []`** on every model — rules live in controllers
4. **No `.env.example`** in repo — every new dev has to ask for keys
5. **Hardcoded asset URLs** in views (`/css/vendors.min.css`) instead of `base_url('css/vendors.min.css')`
6. **Two number-formatting APIs** in views: `Intl.NumberFormat('es-MX', …)` (page-level) and `new NumberFormatter('es_MX', CURRENCY)` (per-page) — pick one
7. **`SaleModel` instantiated inside `Home::index()`** (line 28) instead of in constructor like the others
8. **`composer.json` says `php ^8.1`** but `public/index.php` and `spark` require `8.2` — bump composer constraint
9. **No `app/Language/es/`** translations — Spanish strings hardcoded in views
10. **`flatpickr` referenced** in `components/newExpense.php` but no partial loads its `.js`/`.css` — add to `partials/footer-scripts.php` or page-level `scripts` section
11. **No RBAC** — `AuthGroups` defines 5 groups + 7 permissions, but every route is gated only by `session`. Add `group:admin` or `permission:users.manage` filters where appropriate
12. **No CSRF on AJAX endpoints** — `csrf_meta()` not in head, `$.ajax` calls don't add the token. If CSRF is ever enabled globally these will break
13. **No custom 404 / Exception handler** — sticks with CI4 defaults
14. **Tests are thin** — only 3 example tests; no feature tests for the actual controllers
