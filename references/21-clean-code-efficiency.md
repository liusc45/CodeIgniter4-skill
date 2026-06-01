# Reference 21 — Clean Code & Efficiency Rules for CI4

> **The skill's mission is to make team code cleaner, faster, and more maintainable.**
> This reference encodes the standards that every new line of code MUST meet, and that every legacy line SHOULD migrate toward. When the team's current style conflicts with these rules, **prefer these rules** and explain the improvement.

These rules are derived from:
- *Clean Code* (Robert C. Martin) — naming, functions, comments, error handling
- PSR-1, PSR-12 — coding standards
- CodeIgniter 4 official user guide (Context7-validated)
- Real-world observations from `lotemanager` and `acolhuas`

---

## 1. Naming

### 1.1 Use intention-revealing names

```php
// ❌ Bad — ambiguous
$d = 7;
$arr = $this->m->find();
public function calc($x, $y) { ... }

// ✅ Good — name reveals intent and unit
$retentionDays = 7;
$activeSales = $this->saleModel->findActive();
public function calculateMonthlyInterest(float $principal, float $annualRate): float { ... }
```

### 1.2 Class naming convention (mandatory)

| Element | Pattern | Example |
|---|---|---|
| Controller | Singular PascalCase | `Client`, `Sale` (no `Clients`, no `ClientController`) |
| Model | Singular + `Model` | `ClientModel` (never `Client_model`, never `ClientsModel`) |
| Entity | Singular | `Client` |
| Service | Singular + `Service` | `BillingService`, `ReportService` (never `Billing_service`) |
| Filter | Verb or noun + `Filter` | `AuthFilter`, `RateLimitFilter` |
| Migration | `YYYY-MM-DD-HHMMSS_VerbNoun.php` | `2026-03-15-101530_CreateInvoicesTable.php` |
| Test | Class + `Test` | `SaleTest`, `BillingServiceTest` |

### 1.3 Boolean naming — use `is/has/can/should`

```php
// ❌ Bad
public bool $deleted;
public bool $active;
public function checkPermission(): bool { ... }

// ✅ Good
public bool $isDeleted;
public bool $isActive;
public function canEdit(): bool { ... }
public function hasPermission(string $name): bool { ... }
```

### 1.4 Avoid Hungarian notation, prefixes, and abbreviations

```php
// ❌ Bad
$strName, $intCount, $arrUsers, $usrSrv, $tmpVar
public function fnGetData() { ... }

// ✅ Good
$name, $count, $users, $userService, $temporaryReport
public function getData() { ... }
```

### 1.5 Constants and enums

```php
// ❌ Bad — magic numbers/strings
if ($status === 1) { ... }
if ($type === 'taller') { ... }

// ✅ Good — enum (PHP 8.1+)
enum SaleStatus: int {
    case Pending   = 1;
    case Confirmed = 2;
    case Cancelled = 3;
}

enum CourseType: string {
    case Workshop      = 'taller';
    case WorkshopPlus  = 'taller-plus';
    case Course        = 'curso';
    case Diploma       = 'diplomado';
}

if ($sale->status === SaleStatus::Confirmed) { ... }
```

> Replace lotemanager's `$types` array (`Course.php:26-31`) with a `CourseType` enum. Replace acolhuas's hardcoded `["taller", "taller-plus", "curso", "diplomado"]` everywhere.

---

## 2. Functions

### 2.1 Functions should be small (< 20 lines)

If a function exceeds 20 lines, it's doing too much. Extract.

```php
// ❌ Bad — Sale::create() does 4 things (lotemanager pattern, ~40 lines)
public function create()
{
    $entity = new Sale($this->request->getPost());
    try { $ok = $this->saleModel->save($entity); } catch (...) { ... }
    $saleId = $this->saleModel->getInsertID();
    if ($entity->front_payment > 0) {
        $this->paymentModel->insert([...]);
    }
    $day = (int) date('d', strtotime($entity->order_date));
    $paymentDay = match (true) {
        $day <= 7  => 1,
        $day <= 21 => 15,
        default    => 1,
    };
    $this->saleModel->update($saleId, ['payment_day' => $paymentDay]);
    return $this->response->setJSON($this->saleModel->find($saleId));
}

// ✅ Good — single responsibility per method, transaction-wrapped
public function create(): ResponseInterface
{
    $sale = new Sale($this->request->getPost());
    return $this->saleService->createWithFrontPayment($sale)
        ? $this->respondCreated($sale)
        : $this->failServerError('Could not create sale');
}

// app/Services/SaleService.php
final class SaleService
{
    public function __construct(
        private SaleModel $sales,
        private PaymentModel $payments,
        private \CodeIgniter\Database\BaseConnection $db,
    ) {}

    public function createWithFrontPayment(Sale $sale): bool
    {
        $this->db->transStart();
        $saleId = $this->sales->insert($sale, true);
        $sale->id = $saleId;
        if ($sale->front_payment > 0) {
            $this->createFrontPayment($sale);
        }
        $this->sales->update($saleId, ['payment_day' => $this->calculatePaymentDay($sale->order_date)]);
        $this->db->transComplete();
        return $this->db->transStatus();
    }

    private function createFrontPayment(Sale $sale): void
    {
        $this->payments->insert([
            'sale'           => $sale->id,
            'amount'         => $sale->front_payment,
            'payment_method' => $sale->payment_method,
            'payment_date'   => Time::now(),
        ]);
    }

    private function calculatePaymentDay(string $orderDate): int
    {
        $day = (int) Time::parse($orderDate)->day;
        return match (true) {
            $day <= 7  => 1,
            $day <= 21 => 15,
            default    => 1,
        };
    }
}
```

### 2.2 Functions should do one thing (Single Responsibility Principle)

A function description should not contain "and" or "or".

```php
// ❌ Bad — three responsibilities in one method
public function loginAndAuditAndRedirect($user) { ... }

// ✅ Good
public function login(User $user): bool { ... }
public function audit(User $user, string $action): void { ... }
public function redirectAfterLogin(User $user): RedirectResponse { ... }
```

### 2.3 Maximum 3 parameters; prefer DTOs / Entities for more

```php
// ❌ Bad — 6 parameters
public function createSale(int $clientId, string $block, int $lot, float $amount, float $front, string $method): int { ... }

// ✅ Good — DTO
final readonly class NewSaleData
{
    public function __construct(
        public int $clientId,
        public string $block,
        public int $lot,
        public float $amount,
        public float $frontPayment,
        public string $paymentMethod,
    ) {}
}

public function createSale(NewSaleData $data): int { ... }
```

### 2.4 No side effects in queries / getters

```php
// ❌ Bad — getCourse() mutates state
public function getCourse(int $id): Course
{
    $course = $this->courseModel->find($id);
    $this->logModel->insert(['action' => 'view_course', ...]); // side effect
    return $course;
}

// ✅ Good — separate read from write
public function getCourse(int $id): Course
{
    return $this->courseModel->find($id);
}

public function recordCourseView(int $courseId, int $userId): void
{
    $this->logModel->insert(['action' => 'view_course', 'course_id' => $courseId, 'user_id' => $userId]);
}
```

### 2.5 Use early returns, avoid nesting

```php
// ❌ Bad — pyramid of doom (acolhuas style)
public function update($id)
{
    if ($id) {
        $client = $this->clientModel->find($id);
        if ($client) {
            if ($this->validate($rules)) {
                if ($this->clientModel->save($client)) {
                    return $this->respond($client);
                } else {
                    return $this->fail('Save failed');
                }
            } else {
                return $this->fail($this->validator->getErrors());
            }
        } else {
            return $this->failNotFound();
        }
    } else {
        return $this->fail('No ID');
    }
}

// ✅ Good — guard clauses, single happy path
public function update($id): ResponseInterface
{
    if (!$id) {
        return $this->fail('Missing ID');
    }

    $client = $this->clientModel->find($id);
    if (!$client) {
        return $this->failNotFound();
    }

    if (!$this->validate($this->clientModel->getValidationRules())) {
        return $this->fail($this->validator->getErrors());
    }

    $client->fill($this->request->getRawInput());
    if (!$this->clientModel->save($client)) {
        return $this->fail('Save failed');
    }

    return $this->respond($client);
}
```

---

## 3. Controllers — keep them thin

### 3.1 Controller responsibilities (and only these)

A controller may:
1. Parse the request
2. Validate input shape (delegate semantic validation to the model)
3. Call **one** service (or model directly for simple CRUD)
4. Return a response

A controller MUST NOT:
- Run multiple SQL queries
- Calculate business rules
- Format dates/currency for storage
- Send emails directly (delegate to a service)
- Mutate session/cookies (delegate to filters or services)

### 3.2 Maximum file size: 200 lines

When a controller exceeds 200 lines, **split it**:

```
// ❌ Bad — one Home.php with dashboard + 4 reports + 8 page renderers (lotemanager: ~300 lines)
app/Controllers/Home.php

// ✅ Good — split by responsibility
app/Controllers/DashboardController.php       // index() with stats
app/Controllers/Reports/CollectionsController.php   // reporteCobranza()
app/Controllers/Reports/FinancialController.php     // reporteFinanciero()
app/Controllers/Reports/CustomersController.php     // reporteClientes()
app/Controllers/Reports/SalesAgentsController.php   // reporteVendedores()
app/Controllers/PageController.php             // simple view renderers (clients, sales, payments, expenses)
```

### 3.3 Use `ResourceController` for REST CRUD

```php
// ❌ Bad — manual REST in BaseController (lotemanager pattern)
class Sale extends BaseController {
    use ResponseTrait;
    public function index() { ... }
    public function show($id) { ... }
    public function create() { ... }
    public function update($id) { ... }
    public function delete($id) { ... }
}

// ✅ Good — leverage framework
class Sale extends \CodeIgniter\RESTful\ResourceController
{
    protected $modelName = SaleModel::class;
    protected $format    = 'json';
    // Default index/show/create/update/delete inherited;
    // override only when business logic requires it.
}
```

---

## 4. Models — fat models, thin controllers

### 4.1 Validation lives in the model

```php
// ❌ Bad — validation in controller (lotemanager + acolhuas pattern)
public function create() {
    if (!$this->validate(['title' => 'required', ...])) { ... }
}

// ✅ Good — model owns validation
class CourseModel extends Model
{
    protected $validationRules = [
        'title'    => 'required|min_length[3]|max_length[150]',
        'price'    => 'required|decimal',
        'modality' => 'required|integer|is_not_unique[modality.id]',
    ];

    protected $validationMessages = [
        'title'    => ['required' => 'El título es obligatorio'],
    ];
}
```

### 4.2 Audit columns auto-filled via callbacks

```php
// ✅ Good — every business model
class BaseAuditedModel extends Model
{
    protected $beforeInsert = ['stampCreatedBy'];
    protected $beforeUpdate = ['stampUpdatedBy'];
    protected $beforeDelete = ['stampDeletedBy'];

    protected function stampCreatedBy(array $data): array {
        $userId = auth()->id();
        if ($userId) $data['data']['created_by'] = $userId;
        return $data;
    }
    protected function stampUpdatedBy(array $data): array {
        $userId = auth()->id();
        if ($userId) $data['data']['updated_by'] = $userId;
        return $data;
    }
    protected function stampDeletedBy(array $data): array {
        $userId = auth()->id();
        if ($userId && isset($data['id'])) {
            $this->builder()->where($this->primaryKey, $data['id'])->update(['deleted_by' => $userId]);
        }
        return $data;
    }
}

// Then:
class ClientModel extends BaseAuditedModel { ... }
class SaleModel extends BaseAuditedModel { ... }
```

### 4.3 Encapsulate query logic in named methods

```php
// ❌ Bad — query builder spread across controller (acolhuas Course.php:78-87)
$builder = (new CourseModel)->builder()
    ->select("course.id_course, ...")
    ->join("course_category", "course_category.id_course = course.id_course", "left")
    ->join("modality", "modality.id_modality = course.modality", "left")
    ->where("course.deleted_at is null")
    ->where("active <2")
    ->groupBy("course.id_course");

// ✅ Good — named scope on the model
class CourseModel extends Model
{
    public function builderForListing(): BaseBuilder
    {
        return $this->builder()
            ->select('course.id_course, course.uuid, course.active, course.title, course.suscriptions, course.hours, modality.name AS modality_name')
            ->join('course_category', 'course_category.id_course = course.id_course', 'left')
            ->join('modality', 'modality.id_modality = course.modality', 'left')
            ->where('course.deleted_at IS NULL')
            ->where('course.active <', 2)
            ->groupBy('course.id_course');
    }
}

// Controller:
$return = DataTable::of($this->courseModel->builderForListing())->toJson(true);
```

---

## 5. Services & Dependency Injection

### 5.1 Register every service in `app/Config/Services.php`

```php
// ✅ Good
class Services extends BaseService
{
    public static function billingService(bool $getShared = true): \App\Services\BillingService
    {
        return $getShared
            ? static::getSharedInstance('billingService')
            : new \App\Services\BillingService(
                model('SaleModel'),
                model('PaymentModel'),
                \Config\Database::connect(),
            );
    }
}
```

### 5.2 Constructor injection — never `new XxxService()` inside methods

```php
// ❌ Bad — acolhuas pattern (60+ occurrences)
public function index() {
    $modalities = (new ModalityService)->getAll();
}

// ✅ Good
public function __construct() {
    $this->modalityService = service('modalityService');
}
public function index() {
    $modalities = $this->modalityService->getAll();
}
```

### 5.3 Services contain business logic, not HTTP

A service must NOT call `$this->request`, `$this->response`, `redirect()`, or `view()`. If it does, it's actually a controller.

---

## 6. Database & Performance

### 6.1 Wrap multi-write actions in transactions

Per [CI4 docs / Database transactions](https://codeigniter4.github.io/userguide/database/transactions.html):

```php
$db = \Config\Database::connect();
$db->transStart();

$this->orderModel->insert($order);
$this->inventoryModel->decrement($order->productId, $order->qty);
$this->paymentModel->insert($payment);

$db->transComplete();

if ($db->transStatus() === false) {
    return $this->failServerError('Transaction failed');
}
```

### 6.2 Avoid N+1 queries

```php
// ❌ Bad — N+1 (one query per sale)
$sales = $this->saleModel->findAll();
foreach ($sales as $sale) {
    $sale->client = $this->clientModel->find($sale->client_id);
}

// ✅ Good — single JOIN
$sales = $this->saleModel
    ->select('sales.*, clients.name AS client_name, clients.email AS client_email')
    ->join('clients', 'clients.id = sales.client', 'left')
    ->findAll();
```

### 6.3 Always parameter-bind, never concatenate SQL

```php
// ❌ Bad — SQL injection
$db->query("SELECT * FROM users WHERE email = '" . $_GET['email'] . "'");

// ✅ Good
$db->query("SELECT * FROM users WHERE email = ?", [$email]);
$builder->where('email', $email)->get();
```

### 6.4 Index foreign keys and frequently-filtered columns

In migrations:

```php
$this->forge->addKey('client');                       // index on FK
$this->forge->addKey(['order_date', 'order_status']); // composite index for reports
$this->forge->addUniqueKey('email');                  // unique constraint
$this->forge->addForeignKey('client', 'clients', 'id', 'CASCADE', 'RESTRICT');
```

### 6.5 Cache expensive reads

```php
// app/Services/ReportService.php
public function collections(string $month): array
{
    return cache()->remember("report.collections.{$month}", 300, function () use ($month) {
        // ... heavy SQL
    });
}
```

For lotemanager's `Home::reporte*` methods (which run on every page load), wrapping in 5-minute cache is the easiest 10× speedup.

### 6.6 Use `pConnect = false` (already team default) and configure connection pooling at the database, not PHP

`pConnect = true` causes connection leaks under load. Stick with team default.

---

## 7. Error Handling

### 7.1 Throw typed exceptions; catch at the boundary

```php
// app/Exceptions/SaleException.php
namespace App\Exceptions;

class SaleException extends \DomainException
{
    public static function paymentExceedsAmount(): self {
        return new self('Front payment exceeds sale amount');
    }
}

// Service throws
public function create(NewSaleData $data): Sale
{
    if ($data->frontPayment > $data->amount) {
        throw SaleException::paymentExceedsAmount();
    }
    // ...
}

// Controller catches and translates to HTTP
public function create() {
    try {
        $sale = $this->saleService->create($data);
        return $this->respondCreated($sale);
    } catch (SaleException $e) {
        return $this->fail($e->getMessage(), 422);
    }
}
```

### 7.2 Never `catch (\Exception $e)` and silently continue

```php
// ❌ Bad — swallows errors
try { $this->doStuff(); } catch (\Exception $e) {}

// ✅ Good — log + rethrow OR translate
try {
    $this->doStuff();
} catch (\Throwable $e) {
    log_message('error', '[doStuff] ' . $e->getMessage());
    throw $e;
}
```

### 7.3 Use `\Throwable`, not `\Exception`

PHP 7+ has `Error` (TypeError, ParseError) that don't extend `Exception`. Catch `\Throwable` for top-level handlers.

---

## 8. Security

### 8.1 Never hardcode secrets

```php
// ❌ Bad (acolhuas Services.php:24)
return '1nst1tut0Tz4p1inAc0lhu4s.mx';

// ✅ Good
return env('jwt.secret') ?? throw new \RuntimeException('jwt.secret not set');
```

### 8.2 Validate `Authorization` header defensively

```php
// ❌ Bad (acolhuas AuthFilter.php:35)
$token = explode(' ', $authHeader)[1];   // crashes if missing

// ✅ Good
if (!preg_match('/^Bearer\s+(\S+)$/', $authHeader ?? '', $m)) {
    return service('response')->setStatusCode(401);
}
$token = $m[1];
```

### 8.3 Use the native CORS filter, never `header()` calls

See `references/20-legacy-acolhuas-fixes.md` issue #4 + #5.

### 8.4 CSRF for non-GET routes that mutate state

```php
// app/Config/Filters.php
public array $globals = [
    'before' => [
        'csrf' => ['except' => ['api/*']],
    ],
];
```

For AJAX clients, ship the token via `csrf_meta()` and add to jQuery defaults:

```javascript
const csrfName  = $('meta[name="csrf-name"]').attr('content');
const csrfToken = $('meta[name="csrf-token"]').attr('content');
$.ajaxSetup({ headers: { [csrfName]: csrfToken } });
```

### 8.5 Always escape view output

```php
// ❌ Bad — XSS
<div><?= $user->bio ?></div>

// ✅ Good
<div><?= esc($user->bio) ?></div>
<div><?= esc($user->bio, 'attr') ?></div>   <!-- when injecting into HTML attribute -->
```

---

## 9. View Layer

### 9.1 No business logic in views

Views may:
- Loop, conditionally render
- Call helpers (`esc`, `base_url`, `csrf_field`)
- Format display values (currency, dates)

Views must NOT:
- Query the database
- Mutate state
- Send emails
- Make HTTP calls

### 9.2 Use `base_url()`, never hardcoded paths

```php
// ❌ Bad (lotemanager pattern in app.php)
<link rel="stylesheet" href="/css/vendors.min.css">
<script src="/js/vendors.min.js"></script>

// ✅ Good — works in subdirectory deploys
<link rel="stylesheet" href="<?= base_url('css/vendors.min.css') ?>">
<script src="<?= base_url('js/vendors.min.js') ?>"></script>
```

### 9.3 One number-formatting API per project

Pick `Intl.NumberFormat` (browser) **or** `NumberFormatter` (PHP). Don't use both like lotemanager does.

### 9.4 i18n — move strings to language files

```php
// ❌ Bad — Spanish strings hardcoded in views
<h2>Reporte de Cobranza</h2>

// ✅ Good
// app/Language/es/Reports.php
return ['collections' => 'Reporte de Cobranza'];

// view
<h2><?= lang('Reports.collections') ?></h2>
```

---

## 10. Testing

### 10.1 Coverage targets

| Layer | Min coverage |
|---|---|
| Services (business logic) | 90% |
| Models (with `$validationRules`) | 80% |
| Controllers | 60% (feature tests) |
| Filters | 100% (small surface area) |

### 10.2 One test per behavior, not per method

```php
// ✅ Good
class SaleServiceTest extends CIUnitTestCase
{
    use DatabaseTestTrait;

    public function test_creates_sale_without_front_payment(): void { ... }
    public function test_creates_sale_with_front_payment_and_payment_record(): void { ... }
    public function test_rolls_back_when_payment_insertion_fails(): void { ... }
    public function test_calculates_payment_day_for_first_week(): void { ... }
    public function test_throws_when_front_payment_exceeds_amount(): void { ... }
}
```

### 10.3 Use Faker, never hardcoded test data

```php
// app/Database/Seeds/TestDataSeeder.php
public function run() {
    $faker = \Faker\Factory::create('es_MX');
    foreach (range(1, 100) as $_) {
        $this->db->table('clients')->insert([
            'name' => $faker->firstName,
            'last_name' => $faker->lastName,
            'email' => $faker->unique()->safeEmail,
            'phone' => $faker->phoneNumber,
        ]);
    }
}
```

### 10.4 Feature tests for every REST endpoint

```php
class SaleControllerTest extends CIUnitTestCase
{
    use FeatureTestTrait, DatabaseTestTrait;

    public function test_index_returns_paginated_sales(): void
    {
        $result = $this->withSession(['user' => 1])->get('/sale');
        $result->assertOK()->assertJSONFragment(['client_name' => 'Test']);
    }

    public function test_create_requires_authentication(): void
    {
        $this->call('POST', '/sale', [...])->assertStatus(401);
    }
}
```

---

## 11. Documentation

### 11.1 Code comments explain *why*, not *what*

```php
// ❌ Bad — comment restates the code
// Increment counter by 1
$counter++;

// ✅ Good — comment explains intent
// Cap at 3 to prevent runaway recursion in cycles in the category tree
if ($depth >= 3) return;
```

### 11.2 Class docblock with one-line purpose

```php
/**
 * Calculates aging buckets for accounts receivable.
 *
 * Buckets follow Mexican fiscal convention: 0-30, 31-60, 61-90, +90 days
 * from the original sale date. Soft-deleted records are excluded.
 */
final class CollectionsAgingService { ... }
```

### 11.3 Public methods have full docblocks; private/protected may have minimal docs

---

## 12. Tooling — enforce automatically

Add these to `composer.json`:

```json
{
    "require-dev": {
        "phpstan/phpstan": "^1.11",
        "rector/rector": "^1.2",
        "friendsofphp/php-cs-fixer": "^3.64",
        "phpunit/phpunit": "^10.5"
    },
    "scripts": {
        "lint": "php-cs-fixer fix --dry-run --diff",
        "lint:fix": "php-cs-fixer fix",
        "analyse": "phpstan analyse --level=6 app/",
        "refactor": "rector process app/ --dry-run",
        "refactor:fix": "rector process app/",
        "test": "phpunit"
    }
}
```

`phpstan.neon`:
```neon
parameters:
    level: 6
    paths:
        - app/
    bootstrapFiles:
        - vendor/codeigniter4/framework/system/bootstrap.php
    excludePaths:
        - app/Database/Migrations
```

`rector.php` (incrementally upgrade to PHP 8.2 idioms):
```php
return RectorConfig::configure()
    ->withPaths([__DIR__ . '/app'])
    ->withPhpSets(php82: true)
    ->withTypeCoverageLevel(30)
    ->withDeadCodeLevel(30)
    ->withCodeQualityLevel(30);
```

`.php-cs-fixer.dist.php`:
```php
return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR12'                       => true,
        '@PHP82Migration'              => true,
        'declare_strict_types'         => true,
        'array_syntax'                 => ['syntax' => 'short'],
        'ordered_imports'              => true,
        'no_unused_imports'            => true,
        'single_quote'                 => true,
        'trailing_comma_in_multiline'  => true,
        'no_extra_blank_lines'         => true,
    ])
    ->setFinder(
        PhpCsFixer\Finder::create()
            ->in(__DIR__ . '/app')
    );
```

Run before commit:
```bash
composer lint:fix && composer analyse && composer test
```

---

## 13. Refactor playbook — applying these rules to existing code

When asked to "improve this controller / service / model":

1. **Read it fully.** Don't refactor blind.
2. **Identify the single responsibility.** If you can't state it in one sentence, the file needs splitting.
3. **Extract magic numbers/strings to enums or constants.**
4. **Move validation to the model.**
5. **Move business logic to a service.**
6. **Inject dependencies via constructor (Services::register).**
7. **Wrap multi-write flows in `transStart/transComplete`.**
8. **Add `final readonly` to DTOs/Entities where state is immutable.**
9. **Replace `array $rules = [...]` with typed properties or DTOs.**
10. **Add `declare(strict_types=1);` at file top.**
11. **Add return types to every method.**
12. **Add a feature test that covers the happy path before changing anything else.**
13. **Run `composer analyse` — fix every level-6 PHPStan issue introduced.**
14. **Run `composer lint:fix` — let CS-Fixer handle formatting.**
15. **Commit per logical change** — don't squash a 12-step refactor into one commit.

---

## 14. Quick "is this clean?" checklist

Before opening a PR, every file MUST satisfy:

- [ ] `declare(strict_types=1);` at top
- [ ] All methods have parameter and return types
- [ ] No method exceeds 20 lines (unless it's a pure data structure builder)
- [ ] No file exceeds 200 lines (controllers) / 300 lines (services) / 400 lines (models)
- [ ] No `header()` calls (use `$this->response`)
- [ ] No `(new XxxService)` inside methods (constructor or `service()`)
- [ ] No magic numbers/strings (use enum or const)
- [ ] No N+1 queries (verify with `php spark db:query:log` or debug toolbar)
- [ ] No `catch (\Exception $e) {}` (log, rethrow, or translate)
- [ ] CSRF protection enabled for state-mutating endpoints
- [ ] All output is `esc()`-ed in views
- [ ] At least one feature test covers the new endpoint
- [ ] PHPStan level 6 passes
- [ ] CS-Fixer applies no changes (already conformant)

---

## 15. References

- [CI4 user guide / Models](https://codeigniter4.github.io/userguide/models/model.html)
- [CI4 user guide / Database transactions](https://codeigniter4.github.io/userguide/database/transactions.html)
- [CI4 user guide / Services](https://codeigniter4.github.io/userguide/concepts/services.html)
- [CI4 user guide / RESTful resources](https://codeigniter4.github.io/userguide/incoming/restful.html)
- [CI4 user guide / CORS](https://codeigniter4.github.io/userguide/libraries/cors.html)
- [CI4 user guide / Testing](https://codeigniter4.github.io/userguide/testing/index.html)
- *Clean Code* — Robert C. Martin
- PSR-1 / PSR-12 — https://www.php-fig.org/psr/
- PHPStan — https://phpstan.org/
- Rector — https://getrector.com/
- PHP-CS-Fixer — https://cs.symfony.com/
