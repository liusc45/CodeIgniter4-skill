# 10 - Services & Dependency Injection

CodeIgniter 4's service container and DI patterns.

## What Are Services?

Services are shared singleton instances managed by a service container. They hold the wiring of common framework components: request, response, database, session, cache, etc.

Two ways to access:

```php
// Helper function (preferred for brevity)
$request  = service('request');
$response = service('response');
$session  = service('session');
$cache    = service('cache');

// Direct call
$request = \Config\Services::request();
```

## Built-in Services

| Name | Returns | Use Case |
|------|---------|----------|
| `request` | `IncomingRequest` | HTTP request data |
| `response` | `Response` | HTTP response builder |
| `router` | `Router` | Route resolution |
| `session` | `Session` | Session manager |
| `database` | `BaseConnection` | Database connection |
| `validation` | `Validation` | Validation manager |
| `cache` | `CacheInterface` | Cache (file/redis/memcached) |
| `email` | `Email` | Email sender |
| `encrypter` | `Encrypter` | Encryption |
| `image` | `BaseHandler` | Image manipulation |
| `logger` | `Logger` | PSR-3 logger |
| `parser` | `Parser` | View parser (template variables) |
| `renderer` | `View` | View renderer |
| `security` | `Security` | CSRF and security utilities |
| `throttler` | `Throttler` | Rate limiting |
| `timer` | `Timer` | Performance timer |
| `toolbar` | `Toolbar` | Debug toolbar |
| `uri` | `URI` | URI helper |

## Get Shared vs New Instance

```php
// Shared (singleton — same instance across calls)
$session = service('session');                  // shared by default
$session = service('session', null, true);      // explicit shared
$session = \Config\Services::session();         // shared

// New instance (each call returns a different object)
$session = service('session', null, false);     // not shared
$session = \Config\Services::session(null, false);
```

## Custom Services

Create a custom service by adding a method to `app/Config/Services.php`:

```php
<?php

namespace Config;

use App\Libraries\PaymentGateway;
use App\Libraries\NotificationDispatcher;
use CodeIgniter\Config\BaseService;

class Services extends BaseService
{
    public static function paymentGateway(?string $apiKey = null, bool $getShared = true): PaymentGateway
    {
        if ($getShared) {
            return static::getSharedInstance('paymentGateway', $apiKey);
        }

        $apiKey ??= getenv('STRIPE_API_KEY') ?: '';

        return new PaymentGateway($apiKey);
    }

    public static function notifier(bool $getShared = true): NotificationDispatcher
    {
        if ($getShared) {
            return static::getSharedInstance('notifier');
        }

        return new NotificationDispatcher(
            service('email'),
            service('cache'),
        );
    }
}
```

Use:

```php
$gateway = service('paymentGateway');
$gateway->charge($order);

$notifier = service('notifier');
$notifier->send($user, 'Welcome!');
```

## Replacing a Service (Testing & Customization)

You can swap a service implementation, useful for tests or custom behavior:

```php
// In a test setup or service provider
\Config\Services::injectMock('cache', new \App\Mocks\FakeCache());

// Now everywhere using service('cache') gets the fake one
$cache = service('cache');
$cache->save('key', 'value');   // Hits FakeCache
```

## Constructor Injection (Manual DI)

CI4 doesn't auto-wire DI like Laravel/Symfony, but you can inject manually:

```php
<?php

namespace App\Libraries;

use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\Cache\CacheInterface;

class ReportGenerator
{
    public function __construct(
        protected RequestInterface $request,
        protected CacheInterface $cache,
        protected string $apiKey,
    ) {}

    public function generate(int $userId): array
    {
        $cacheKey = "report.user.{$userId}";

        if ($cached = $this->cache->get($cacheKey)) {
            return $cached;
        }

        // Generate report...
        $report = ['user_id' => $userId, 'data' => '...'];

        $this->cache->save($cacheKey, $report, 3600);
        return $report;
    }
}
```

Wire it via a service:

```php
// app/Config/Services.php
public static function reportGenerator(bool $getShared = true): \App\Libraries\ReportGenerator
{
    if ($getShared) {
        return static::getSharedInstance('reportGenerator');
    }

    return new \App\Libraries\ReportGenerator(
        service('request'),
        service('cache'),
        getenv('REPORTS_API_KEY') ?: ''
    );
}
```

Use it: `service('reportGenerator')->generate($userId);`

## Factories Pattern

For things instantiated multiple times (models, libraries, configs), use Factories:

```php
// Models
$users = model('UserModel');                    // shared
$users = model('UserModel', false);             // new
$users = \Config\Factories::models('UserModel');

// Configs
$auth = config('Auth');                          // shared
$auth = config('Auth', false);                   // new

// Libraries
$lib = \Config\Factories::libraries('SomeLib');
```

## Events System

CI4 ships with an Events system (replacement for CI3's Hooks).

`app/Config/Events.php`:

```php
<?php

namespace Config;

use CodeIgniter\Events\Events;
use CodeIgniter\Exceptions\FrameworkException;

Events::on('pre_system', static function () {
    // Runs before the framework starts
});

Events::on('post_controller_constructor', static function () {
    // Runs after controller is built
});

Events::on('user.registered', static function ($user) {
    log_message('info', "New user: {$user->email}");
    service('notifier')->send($user, 'welcome');
});

Events::on('order.placed', ['App\Listeners\SendOrderConfirmation', 'handle']);
```

Trigger events:

```php
\CodeIgniter\Events\Events::trigger('user.registered', $user);
\CodeIgniter\Events\Events::trigger('order.placed', $order);
```

### Built-in Events

| Event | When |
|-------|------|
| `pre_system` | Before framework boots |
| `post_controller_constructor` | After controller's `__construct()` |
| `post_system` | After response sent |
| `email` | When email is sent |
| `migrate` | When a migration runs |
| `DBQuery` | On every database query (dev) |

## Cache Service

```php
$cache = service('cache');

// Save
$cache->save('key', $data, 3600);          // 1 hour
$cache->save('users.list', $users, 600);   // 10 min

// Read
$data = $cache->get('key');

// Delete
$cache->delete('key');
$cache->deleteMatching('users.*');         // pattern delete
$cache->clean();                            // delete ALL

// Get-or-set pattern
$users = $cache->remember('users.active', 300, function () {
    return model('UserModel')->where('is_active', 1)->findAll();
});
```

Configure backend in `app/Config/Cache.php` (file, redis, memcached, predis, dummy).

## Logger Service

```php
log_message('debug', 'Debug message');     // dev only by default
log_message('info', 'User {id} logged in', ['id' => $user->id]);
log_message('notice', 'Notice');
log_message('warning', 'Warning');
log_message('error', 'Error');
log_message('critical', 'Critical');
log_message('alert', 'Alert');
log_message('emergency', 'Emergency');

// Threshold in .env
// logger.threshold = 4 (info and below in production)
```

Logs go to `writable/logs/log-YYYY-MM-DD.log`.

## Best Practices

- **Always use the shared instance** unless you specifically need a new one
- **Define custom services** in `app/Config/Services.php` for any non-trivial dependency
- **Inject services via constructor** for testable libraries
- **Use Events** for cross-cutting concerns (logging, notifications)
- **Use the Cache service** liberally — it's drop-in fast
- **Use `injectMock()`** in tests for service replacement
- **Don't bypass the container** — `new \App\Libraries\X()` skips wiring
- **Use Factories** for models and configs to share instances
