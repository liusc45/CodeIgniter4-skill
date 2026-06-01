# 17 - Best Practices (Daily Use)

Patterns and conventions for everyday CodeIgniter 4 development.

## Project Conventions

### Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Class names | PascalCase | `UserController`, `OrderModel` |
| File names | Match class | `UserController.php` |
| Method names | camelCase | `getUserById()`, `createOrder()` |
| Constants | SCREAMING_SNAKE | `MAX_LOGIN_ATTEMPTS` |
| DB tables | snake_case plural | `users`, `order_items` |
| DB columns | snake_case | `created_at`, `user_id` |
| View files | snake_case | `user_profile.php` |
| Model class | PascalCase + `Model` suffix | `UserModel`, `OrderItemModel` |
| Entity class | PascalCase singular | `User`, `OrderItem` |
| Controller class | PascalCase plural or singular | `Users`, `OrderController` |

### Folder organization

```
app/
├── Controllers/
│   ├── BaseController.php
│   ├── Web/                      # Server-rendered web routes
│   │   ├── HomeController.php
│   │   └── ProfileController.php
│   ├── Api/                      # JSON APIs
│   │   ├── BaseApiController.php
│   │   └── V1/
│   │       ├── UserController.php
│   │       └── PostController.php
│   └── Admin/                    # Admin panel
├── Models/
├── Entities/
├── Filters/
├── Libraries/                    # Custom libs (third-party-like reusable code)
├── Services/                     # Business logic services (custom)
├── ValueObjects/                 # Domain value objects
└── Views/
    ├── layouts/
    ├── partials/
    └── pages/
```

## Code Standards

### Always declare strict types

```php
<?php

declare(strict_types=1);

namespace App\Controllers;

// ...
```

### Type-hint everything

```php
public function getUser(int $id): ?User
{
    return model('UserModel')->find($id);
}

public function processOrders(array $orders, OrderProcessor $processor): int
{
    // ...
}
```

### Use readonly + constructor promotion (PHP 8.1+)

```php
final class OrderTotal
{
    public function __construct(
        public readonly float $subtotal,
        public readonly float $tax,
        public readonly float $shipping,
    ) {}

    public function grand(): float
    {
        return $this->subtotal + $this->tax + $this->shipping;
    }
}
```

### Use Enums (PHP 8.1+)

```php
enum OrderStatus: string
{
    case Pending  = 'pending';
    case Paid     = 'paid';
    case Shipped  = 'shipped';
    case Cancelled = 'cancelled';

    public function isFinal(): bool
    {
        return match ($this) {
            self::Shipped, self::Cancelled => true,
            default => false,
        };
    }
}
```

### Prefer match over switch

```php
// Bad
switch ($role) {
    case 'admin':       $access = 'all'; break;
    case 'editor':      $access = 'content'; break;
    case 'subscriber':  $access = 'read'; break;
    default:            $access = 'none';
}

// Good
$access = match ($role) {
    'admin'      => 'all',
    'editor'     => 'content',
    'subscriber' => 'read',
    default      => 'none',
};
```

### Throw specific exceptions

```php
use CodeIgniter\Exceptions\PageNotFoundException;
use CodeIgniter\HTTP\Exceptions\HTTPException;

public function show(int $id): string
{
    $user = model('UserModel')->find($id);
    if ($user === null) {
        throw PageNotFoundException::forPageNotFound("User {$id}");
    }
    return view('users/show', ['user' => $user]);
}
```

## Controller Best Practices

### Keep controllers thin

Controllers should:
- Accept HTTP input
- Validate input
- Call services/models
- Return response

Controllers should NOT:
- Contain business logic
- Make heavy DB queries directly
- Handle email sending, file processing, etc.

```php
// Bad: business logic in controller
public function placeOrder()
{
    $items = $this->request->getPost('items');
    $total = 0;
    foreach ($items as $item) {
        $product = model('ProductModel')->find($item['id']);
        $total += $product['price'] * $item['qty'];
    }
    // Tax calc, inventory check, email, etc...
    // 100+ lines later...
}

// Good: delegate to a service
public function placeOrder(OrderService $orders): ResponseInterface
{
    $data = $this->request->getJSON(true);
    if (! $this->validate('placeOrder')) {
        return $this->failValidationErrors($this->validator->getErrors());
    }

    try {
        $order = $orders->place(auth()->user(), $data['items']);
        return $this->respondCreated($order);
    } catch (InsufficientStockException $e) {
        return $this->fail($e->getMessage(), 422);
    }
}
```

### Use service classes

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\OrderModel;
use App\Models\ProductModel;

final class OrderService
{
    public function __construct(
        private OrderModel $orders,
        private ProductModel $products,
    ) {}

    public function place(User $user, array $items): array
    {
        // Validate stock
        // Calculate totals
        // Create order
        // Decrement inventory
        // Send email
        // Return order
    }
}
```

Wire it via Services:

```php
public static function orderService(bool $getShared = true): \App\Services\OrderService
{
    if ($getShared) {
        return static::getSharedInstance('orderService');
    }

    return new \App\Services\OrderService(
        model('OrderModel'),
        model('ProductModel'),
    );
}
```

## Model Best Practices

### Always set `$allowedFields`

Without it, mass-assignment fails — but more importantly, it prevents users from injecting unexpected fields.

```php
protected $allowedFields = ['name', 'email'];   // not 'is_admin', 'role_id', etc.
```

### Define validation rules in the model

Single source of truth for "what makes a valid User":

```php
protected $validationRules = [
    'email' => 'required|valid_email|is_unique[users.email,id,{id}]',
];
```

The `{id}` placeholder excludes the current row when updating.

### Use callbacks for cross-cutting concerns

```php
protected $beforeInsert = ['hashPassword', 'normalizeEmail'];
protected $beforeUpdate = ['hashPassword'];

protected function hashPassword(array $data): array
{
    if (isset($data['data']['password'])) {
        $data['data']['password'] = password_hash($data['data']['password'], PASSWORD_DEFAULT);
    }
    return $data;
}

protected function normalizeEmail(array $data): array
{
    if (isset($data['data']['email'])) {
        $data['data']['email'] = strtolower(trim($data['data']['email']));
    }
    return $data;
}
```

### Use entities for business logic

```php
class User extends Entity
{
    public function isAdmin(): bool { return $this->role === 'admin'; }
    public function fullName(): string { return "{$this->first_name} {$this->last_name}"; }
    public function avatarUrl(): string {
        return $this->avatar_path
            ? base_url("uploads/{$this->avatar_path}")
            : '/assets/img/default-avatar.png';
    }
}
```

## API Best Practices

- **Version your API** (`/api/v1/`) from day one
- **Use consistent response shape** (`success`, `data`, `errors`)
- **Use proper HTTP status codes**
- **Paginate everything** (default 20, max 100)
- **Filter, sort, search via query strings** (`?status=active&sort=-created_at`)
- **Document with OpenAPI** (Swagger UI)
- **Rate-limit auth endpoints** (5 attempts / 15 min)
- **Don't expose internal fields** (passwords, tokens, soft-delete columns)
- **Return 204 for empty success** (DELETE, etc.)
- **Use ETag for caching** GET endpoints

## Security Best Practices

### Input

- Validate ALL user input — never trust the client
- Use `$this->validator->getValidated()` to get only valid fields
- Use `$allowedFields` in models
- Cast types explicitly (`(int)`, `(string)`)

### Output

- ALWAYS escape with `esc()` in views
- Use proper context: `esc($val, 'attr')`, `esc($val, 'url')`, `esc($val, 'js')`
- Enable Content Security Policy (`$CSPEnabled = true`)
- Add security headers (`secureheaders` filter)

### Auth

- Use Shield for new projects
- Hash passwords with `password_hash(PASSWORD_DEFAULT)`
- Use `password_verify()` to check
- Add rate limiting to login (`auth-rates` filter)
- Use HTTPS in production (`forceGlobalSecureRequests = true`)
- Set secure session cookies (`$cookieSecure = true`)

### Database

- Use Query Builder or parameter-bound queries — never concatenate SQL
- Use `is_unique[table.field,id,{id}]` for UPDATE uniqueness
- Use foreign keys with `ON DELETE CASCADE`/`SET NULL`
- Backup before migrations in production

### Files

- Validate uploaded files (`uploaded`, `is_image`, `mime_in`, `max_size`)
- Store outside `public/` (`writable/uploads/`)
- Use `getRandomName()` to avoid filename collisions/path injection
- Set proper permissions (`664` for files, `775` for dirs)

## Performance Best Practices

### Caching

```php
// Cache expensive queries
$users = service('cache')->remember('users.active', 300, function () {
    return model('UserModel')->where('is_active', 1)->findAll();
});

// Per-route cache (4.5+)
#[\CodeIgniter\Router\Attributes\Cache(for: HOUR)]
public function index(): string { ... }

// Manual page cache
public function home() {
    if ($cached = service('response')->getCache(60 * 5)) {
        return $cached;
    }
    return view('home');
}
```

### Database

- Add indexes to FK columns and frequently filtered columns
- Use `select()` to fetch only needed columns
- Use `paginate()` for lists
- Avoid N+1: load related data via `whereIn()` or joins
- Use `countAllResults()` carefully (full scan)

### OPcache (production)

```ini
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0      ; rebuild image to update code
opcache.preload = /var/www/html/preload.php
```

### Composer

```bash
composer install --no-dev --optimize-autoloader --classmap-authoritative
```

## Logging Best Practices

```php
// Use appropriate levels
log_message('debug',   'Query: {query}', ['query' => $sql]);   // dev only
log_message('info',    'User {id} logged in', ['id' => $user->id]);
log_message('warning', 'Slow API response: {ms}ms', ['ms' => $elapsed]);
log_message('error',   'Failed to send email: {err}', ['err' => $e->getMessage()]);
log_message('critical','Database connection lost');
```

Configure threshold in `.env`:
```dotenv
logger.threshold = 4   # info and below in production (4-9)
```

## Testing Best Practices

- Aim for **>80% coverage** of your code (not framework code)
- Run tests in CI on every PR
- Use **DatabaseTestTrait** with SQLite `:memory:` for speed
- Mock external services (email, payment, third-party APIs)
- Write tests **before** fixing bugs (regression protection)
- Name tests descriptively: `test_user_cannot_delete_others_post`

## Git / VCS Best Practices

- **Don't commit `.env`** (in `.gitignore`)
- **Don't commit `vendor/`** (composer install in deploy)
- **Don't commit `writable/cache`, `writable/logs`, `writable/session`**
- **Do commit `composer.lock`** (reproducible builds)
- **Do commit `app/Config/`** (with sensitive values via env vars only)
- **Use feature branches** + PRs
- **Keep migrations immutable** — once committed, don't modify
- **Tag releases** (`v1.0.0`)
