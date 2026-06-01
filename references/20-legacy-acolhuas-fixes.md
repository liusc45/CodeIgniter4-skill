# Reference 20 — Legacy Patterns to Fix (acolhuas case study)

> Use this reference when refactoring older Sistematlan CI4 codebases (acolhuas) or when you encounter the same anti-patterns elsewhere. Every recommendation here is **grounded in the official CodeIgniter 4 user guide** (Context7-validated).

The acolhuas project (`~/Desktop/acolhuas`) is a generation older than lotemanager. Composer requires `php ^7.4 || ^8.0`, uses `codeigniter4/framework ^4.0` (no version pin), no Shield, custom JWT auth, 27 controllers, 39 models with mixed naming, 32 services with ad-hoc instantiation, 6 filters (3 of which are broken or risky). It is a goldmine of "what to fix" examples.

---

## 1. **CRITICAL — Hardcoded JWT secret in source code**

**Where:** `app/Config/Services.php:22-26`

```php
class Services extends BaseService
{
    public static function getSecretKey()
    {
        return '1nst1tut0Tz4p1inAc0lhu4s.mx';   // ⚠ COMMITTED TO GIT
    }
}
```

**Why it's risky:** anyone with read access to the repo (including future contractors, leaked backups, public mirrors) can forge JWTs and impersonate any user. This is a **critical** security vulnerability.

**Fix:**

```php
// app/Config/Services.php
public static function getSecretKey(): string
{
    $secret = env('jwt.secret');
    if (empty($secret)) {
        throw new \RuntimeException('jwt.secret is not configured. Set it in .env');
    }
    return $secret;
}
```

```bash
# .env
jwt.secret = "<generated-with-openssl-rand-base64-64>"
```

**Plus rotate immediately** — the old secret is compromised. After rotation, all existing JWTs become invalid (force users to re-login).

**Same fix applies to lotemanager?** Not currently — Shield manages secrets internally. But the pattern (`env()` for any secret) applies universally.

---

## 2. **CRITICAL — Broken `PermissionFilter`**

**Where:** `app/Filters/PermissionFilter.php:14-21`

```php
public function before(RequestInterface $request, $arguments = null)
{
    $methodToGo = __METHOD__;                       // unused
    $permission = session()->get("permissions");    // value never checked
    
    redirect()->back()->with("unauthorized", "No tienes permitido el acceso");
    // ⚠ UNCONDITIONAL — every guarded route returns this redirect
}
```

**Why it's risky:** the filter ignores its input and always emits the same redirect. Any route configured under this filter is permanently broken. The redirect isn't even returned (it's evaluated but discarded), so the request continues with a side-effect flash message but no actual block.

**Fix:** delete the file OR implement properly. If you decide to keep RBAC custom (instead of adopting Shield), the correct shape is:

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class PermissionFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $required = $arguments[0] ?? null;
        if ($required === null) return;

        $userPermissions = session()->get('permissions') ?? [];
        if (!in_array($required, $userPermissions, true)) {
            return redirect()->to('/')
                ->with('unauthorized', 'No tienes permitido el acceso')
                ->setStatusCode(403);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Then route:
```php
$routes->get('/admin/*', 'Admin::dashboard', ['filter' => 'permission:admin.access']);
```

**Recommended replacement:** adopt **Shield** + use its built-in `permission` and `group` filters (per [CI4 user guide / Shield](https://codeigniter4.github.io/userguide/extending/authentication.html)):

```php
$routes->get('/admin/*', 'Admin::dashboard', ['filter' => 'permission:admin.access']);
$routes->get('/staff/*', 'Staff::index',     ['filter' => 'group:admin,developer']);
```

**Same fix applies to lotemanager?** Yes — lotemanager defines 5 groups in `AuthGroups` but never enforces them via `group:` or `permission:` filters. Apply the Shield filter pattern.

---

## 3. **CRITICAL — Unsafe array access in `AuthFilter`**

**Where:** `app/Filters/AuthFilter.php:32-35`

```php
public function before(RequestInterface $request, $arguments = null)
{
    $key        = Services::getSecretKey();
    $authHeader = $request->getServer('HTTP_AUTHORIZATION');
    $arr        = explode(' ', $authHeader);
    $token      = $arr[1];                          // ⚠ undefined index if header missing or malformed
    // ...
}
```

**Why it's risky:** any request without `Authorization: Bearer <token>` or with a single-word value triggers `Undefined array key 1` (PHP 8 fatal). An attacker can DoS or fingerprint by sending malformed headers.

**Fix:**

```php
public function before(RequestInterface $request, $arguments = null)
{
    $authHeader = $request->getServer('HTTP_AUTHORIZATION') ?? '';

    if (!preg_match('/^Bearer\s+(\S+)$/', $authHeader, $matches)) {
        return service('response')
            ->setStatusCode(ResponseInterface::HTTP_UNAUTHORIZED)
            ->setJSON(['error' => 'Missing or malformed Authorization header']);
    }
    $token = $matches[1];

    try {
        $user = JWT::decode($token, new Key(Services::getSecretKey(), 'HS256'));
        log_message('info', 'JWT user: ' . json_encode($user));
        // Optional: store user in request attributes for downstream controllers
    } catch (\Throwable $e) {
        return service('response')
            ->setStatusCode(ResponseInterface::HTTP_UNAUTHORIZED)
            ->setJSON(['error' => 'Invalid token']);
    }
}
```

**Recommended replacement:** install Shield and use its built-in `tokens` or `jwt` filters instead of rolling your own. Shield handles malformed headers, expiry, key rotation, and rate-limiting.

**Same fix applies to lotemanager?** N/A — lotemanager already uses Shield.

---

## 4. **HIGH — Custom `CorsFilter` with hardcoded origins**

**Where:** `app/Filters/CorsFilter.php:36-67`

```php
$allowed_domains = array(
    'https://127.0.0.1',
    'http://127.0.0.1',
    'http://localhost:8080',
    'https://inacolhua.com.mx',
    // ... 11 hardcoded domains
);

if (in_array($origin, $allowed_domains)) {
    header('Access-Control-Allow-Origin: ' . $origin);
}
header("Access-Control-Allow-Headers: Origin, X-API-KEY, ...");
header("Access-Control-Allow-Methods: GET, PUT, POST, DELETE, PATCH, OPTIONS");
header("Access-Control-Allow-Credentials: true");
```

**Why it's risky:**
- Origin list lives in code (deploy to add a domain).
- Bypasses CI4's response object (calls `header()` directly + `die()`).
- Always sends `Access-Control-Allow-Credentials: true` even when origin doesn't match — leaks intent.
- No logging on rejected origins.

**Fix** — use the **native CI4 4.5+ CORS filter** (per [CI4 user guide / CORS](https://codeigniter4.github.io/userguide/libraries/cors.html)):

```php
// app/Config/Cors.php
<?php
namespace Config;

use CodeIgniter\Config\BaseConfig;

class Cors extends BaseConfig
{
    public array $default = [
        'allowedOrigins'      => [],
        'allowedHeaders'      => [],
        'allowedMethods'      => [],
        'supportsCredentials' => false,
        'maxAge'              => 0,
    ];

    public array $api = [
        'allowedOrigins'         => [
            env('cors.frontend.production'),
            env('cors.frontend.staging'),
        ],
        'allowedOriginsPatterns' => ['/^https:\/\/[a-z0-9-]+\.inacolhua\.com\.mx$/i'],
        'allowedHeaders'         => ['Authorization', 'Content-Type', 'X-Requested-With', 'X-CSRF-TOKEN'],
        'exposedHeaders'         => [],
        'allowedMethods'         => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
        'supportsCredentials'    => true,
        'maxAge'                 => 7200,
    ];
}
```

```php
// app/Config/Filters.php
public array $aliases = [
    'cors' => \CodeIgniter\Filters\Cors::class,
    // ...
];

public array $filters = [
    'cors:api' => [
        'before' => ['api/*'],
        'after'  => ['api/*'],
    ],
];
```

Then **delete `app/Filters/CorsFilter.php`** and remove the `cors` route registrations.

**Same fix applies to lotemanager?** lotemanager already has `app/Config/Cors.php` (default), but no API routes need it yet. When the SPA backend ships, configure the `api` named config.

---

## 5. **HIGH — `app/Config/Services.php:42-45` (controllers) — manual CORS headers in every method**

**Where:** repeated 14× across acolhuas controllers, e.g. `app/Controllers/Course.php:42-45`

```php
public function index($search=null, $value=null)
{
    header('Access-Control-Allow-Origin: *');                              // ⚠
    header("Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept");
    header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE');
    header('Content-Type: application/json');
    // ...
}
```

**Why it's risky:**
- `*` origin with credentials is rejected by browsers (silently broken in production).
- Bypasses CI4's response pipeline → headers not testable, not loggable, can collide with the `CorsFilter`.
- 14× duplication means 14 places to change when policy moves.

**Fix:** delete every `header('Access-Control-...')` call. Configure CORS once via the native filter (see issue #4). Controllers should only ever do:

```php
return $this->response->setJSON($data);
```

---

## 6. **MEDIUM — Mixed model naming `XxxModel` vs `Xxx_model`**

**Where:** `app/Models/` has both styles:

```
ActionModel.php          ← good
CourseModel.php          ← good
GroupModel.php           ← good
PromoModel.php           ← good
User_model.php           ← legacy CI3 style
Campaign_model.php       ← legacy
Suscription_model.php    ← legacy
Category_model.php       ← legacy
... (15 files in legacy style)
```

**Why it's risky:** inconsistency increases cognitive load and grep noise; new devs guess the wrong name; auto-routing/DataTables conventions break.

**Fix:** rename all `Xxx_model.php` → `XxxModel.php`. This is a multi-step refactor:

1. `git mv app/Models/User_model.php app/Models/UserModel.php`
2. Rename the class inside.
3. Find all references: `rg -l 'User_model' app/`
4. Replace `\App\Models\User_model` → `\App\Models\UserModel` and `new User_model()` → `new UserModel()`.
5. Run tests, deploy, repeat per model.

Stage the rename across multiple PRs (one model at a time) to keep diffs reviewable.

**Same fix applies to lotemanager?** N/A — lotemanager already follows `XxxModel`.

---

## 7. **MEDIUM — Ad-hoc service instantiation `(new XxxService)` inside controller methods**

**Where:** repeated 60+ times across acolhuas, e.g. `app/Controllers/Course.php:60, 65`

```php
public function create(): string
{
    $data["categories"] = $this->category->get_all();
    $data['modalities'] = (new ModalityService)->getAll();          // ⚠ inline new
    $data["instructors"] = (new User_service)->getInstructors();    // ⚠ inline new
    return view("Courses/course", $data);
}
```

**Why it's risky:**
- Cannot mock for tests.
- New instance every call → no shared state, repeated DB connections.
- Hidden dependencies (controller looks like it depends on `category` only, but pulls 2 more services).

**Fix** (per [CI4 user guide / Services](https://codeigniter4.github.io/userguide/general/common_functions.html)):

Register each service in `app/Config/Services.php`:

```php
// app/Config/Services.php
public static function modality(bool $getShared = true)
{
    return $getShared
        ? static::getSharedInstance('modality')
        : new \App\Services\ModalityService();
}

public static function userService(bool $getShared = true)
{
    return $getShared
        ? static::getSharedInstance('userService')
        : new \App\Services\User_service();
}
```

Then in controllers:

```php
public function __construct()
{
    $this->modality     = service('modality');
    $this->userService  = service('userService');
    $this->category     = service('categoryService');
}

public function create(): string
{
    $data['modalities']  = $this->modality->getAll();
    $data['instructors'] = $this->userService->getInstructors();
    return view('Courses/course', $data);
}
```

For testability, swap in `service('modality', false)` to get a fresh instance, or use `Services::injectMock('modality', $mock)` in tests.

---

## 8. **MEDIUM — Unsafe `system/` overrides**

**Where:** acolhuas has files in `system/` at the project root (orphan framework overrides). These were copied from the framework and edited.

**Why it's risky:**
- Framework upgrades (`composer update codeigniter4/framework`) won't touch them — the codebase silently runs **two versions** of the same class.
- Future devs assume `system/` is part of the framework and don't audit changes.

**Fix:**
1. `diff system/<File>.php vendor/codeigniter4/framework/system/<File>.php` to see what was changed.
2. Move the change to a proper extension in `app/`:
   - For library overrides → `app/Libraries/MyClass.php` + register in `app/Config/Services.php`.
   - For helper overrides → `app/Helpers/my_helper.php`.
3. Delete the `system/` overrides.

---

## 9. **MEDIUM — Mixed base controllers**

**Where:** some controllers extend `BaseController`, others extend `\CodeIgniter\Controller` directly. No consistent approach.

**Fix:** every controller MUST extend `App\Controllers\BaseController`. Even API-only controllers — put a small `JsonResponseTrait` or pre-load `'url'` helper in BaseController for them.

```php
// app/Controllers/BaseController.php
abstract class BaseController extends Controller
{
    protected $helpers = ['url', 'form'];
    protected $session;

    public function initController(RequestInterface $request, ResponseInterface $response, LoggerInterface $logger)
    {
        parent::initController($request, $response, $logger);
        $this->session = service('session');
    }
}
```

---

## 10. **MEDIUM — Empty `$validationRules` in models, validation lives in controllers**

**Where:** `app/Models/CourseModel.php:55-57`

```php
protected $validationRules    = [];
protected $validationMessages = [];
protected $skipValidation     = false;
```

…then `app/Controllers/Course.php:99-108` does:

```php
$minimalInfo = $this->validate([
    "title" => "required",
    "subtitle" => "required",
    // ...
]);
```

**Why it's risky:** validation rules duplicated across `create` and `update` (drift). Direct model `save()` calls (e.g. from a Service) bypass validation. Tests can't rely on the model enforcing constraints.

**Fix** — move rules into the model:

```php
protected $validationRules = [
    'title'           => 'required|min_length[3]|max_length[150]',
    'subtitle'        => 'required|max_length[200]',
    'description'     => 'required',
    'modules'         => 'required',
    'price'           => 'required|decimal',
    'hours'           => 'required|integer',
    'modality'        => 'required|integer',
    'course_category' => 'required',
];

protected $validationMessages = [
    'title' => [
        'required'   => 'El título es obligatorio',
        'min_length' => 'Mínimo 3 caracteres',
    ],
];
```

Controllers shrink to:

```php
if (!$this->courseModel->insert($data)) {
    return $this->fail($this->courseModel->errors());
}
```

---

## 11. **LOW — `app/Common.php` empty stub**

**Where:** every CI4 app ships an empty `app/Common.php`. It's a fine place to put **truly cross-cutting helpers** loaded automatically on every request.

**Useful additions** (only if actually needed — don't pollute):

```php
<?php
// app/Common.php

if (!function_exists('current_user_id')) {
    function current_user_id(): ?int {
        return auth()->id() ?? null;
    }
}

if (!function_exists('format_currency_mxn')) {
    function format_currency_mxn(float $amount): string {
        $f = new NumberFormatter('es_MX', NumberFormatter::CURRENCY);
        return $f->formatCurrency($amount, 'MXN');
    }
}
```

---

## 12. **LOW — No `.env.example` checked in**

**Where:** neither acolhuas nor lotemanager commits a `.env.example`.

**Fix:** create one with all keys, no secrets, then commit:

```bash
# .env.example
CI_ENVIRONMENT = development

# ----- App -----
app.baseURL = "http://localhost:8080/"
app.indexPage = ""
app.cspEnabled = false
app.title = "LoteManager"

# ----- Database -----
database.default.hostname = localhost
database.default.database =
database.default.username =
database.default.password =
database.default.DBDriver = MySQLi
database.default.port = 3307

# ----- Email -----
email.fromEmail =
email.fromName =
email.SMTPHost =
email.SMTPUser =
email.SMTPPass =        # leave empty in .env.example
email.SMTPPort = 587
email.SMTPCrypto = tls

# ----- Auth (Shield) -----
auth.allowRegistration = false
auth.recordActiveDate = true

# ----- Custom -----
jwt.secret =            # generate with: openssl rand -base64 64
cors.frontend.production = "https://app.example.com"
cors.frontend.staging = "https://staging.example.com"
```

---

## 13. Refactor sequencing for an acolhuas-style codebase

Apply fixes in this order to minimize regression risk:

1. **Move secrets to `.env`** (#1) — no behavioral change, just file moves. PR-able in <1 hour.
2. **Add `.env.example`** (#12) — no risk.
3. **Replace `CorsFilter` with native CORS** (#4 + #5) — single feature flip, easy rollback. Test the SPA endpoints.
4. **Fix `AuthFilter` defensively** (#3) — prevents fatals; behavior preserved.
5. **Delete `PermissionFilter`** (#2) — currently broken anyway, deletion can only improve things.
6. **Standardize BaseController** (#9) — touches every controller, but mechanical.
7. **Service registration in `Services.php`** (#7) — incremental, one service at a time.
8. **Promote validation to models** (#10) — incremental, one model at a time.
9. **Rename `Xxx_model` → `XxxModel`** (#6) — one model per PR.
10. **Remove `system/` overrides** (#8) — last, since it forces re-testing of overridden behavior.

After this sequence, the codebase looks much closer to lotemanager and is ready to **adopt Shield** as the next major step (replacing custom JWT entirely).

---

## 14. References

- [CI4 user guide / BaseController](https://codeigniter4.github.io/userguide/extending/basecontroller.html)
- [CI4 user guide / Services](https://codeigniter4.github.io/userguide/concepts/services.html)
- [CI4 user guide / Filters](https://codeigniter4.github.io/userguide/incoming/filters.html)
- [CI4 user guide / CORS](https://codeigniter4.github.io/userguide/libraries/cors.html)
- [CI4 user guide / RESTful resources](https://codeigniter4.github.io/userguide/incoming/restful.html)
- [CI4 user guide / Database transactions](https://codeigniter4.github.io/userguide/database/transactions.html)
- [Shield user guide](https://codeigniter4.github.io/shield/)
