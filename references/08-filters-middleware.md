# 08 - Filters & Middleware

Filters are CodeIgniter 4's middleware. They run before/after controllers.

## Filter Lifecycle

```
HTTP Request
    │
    ▼
[Global before filters]
    │
    ▼
[Route-specific before filters]
    │
    ▼
Controller method
    │
    ▼
[Route-specific after filters]
    │
    ▼
[Global after filters]
    │
    ▼
HTTP Response
```

## Built-in Filters

| Alias | Class | Purpose |
|-------|-------|---------|
| `csrf` | `CSRF` | CSRF protection |
| `toolbar` | `DebugToolbar` | Debug toolbar (dev only) |
| `honeypot` | `Honeypot` | Honeypot anti-spam |
| `invalidchars` | `InvalidChars` | Reject invalid characters |
| `secureheaders` | `SecureHeaders` | Security HTTP headers |
| `forcehttps` | `ForceHTTPS` | Force HTTPS |
| `pagecache` | `PageCache` | Page-level caching |
| `cors` | `Cors` | CORS handling (4.5+) |

## Creating a Custom Filter

```bash
php spark make:filter Auth
```

```php
<?php

declare(strict_types=1);

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class AuthFilter implements FilterInterface
{
    /**
     * Runs BEFORE the controller.
     * Return a Response to short-circuit (don't run controller).
     * Return null/void to continue.
     */
    public function before(RequestInterface $request, $arguments = null)
    {
        if (! session()->get('isLoggedIn')) {
            return redirect()->to('/login')->with('error', 'Please log in.');
        }

        // Optional role check via filter arguments
        if ($arguments !== null) {
            $userRole = session()->get('role');
            if (! in_array($userRole, $arguments, true)) {
                return service('response')
                    ->setStatusCode(403)
                    ->setBody('Forbidden');
            }
        }
    }

    /**
     * Runs AFTER the controller, before sending response.
     * Modify the response if needed.
     */
    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        // Example: Add custom header
        $response->setHeader('X-Powered-By', 'CodeIgniter 4');
    }
}
```

## Registering Filters

`app/Config/Filters.php`:

```php
<?php

namespace Config;

use CodeIgniter\Config\BaseConfig;
use CodeIgniter\Filters\CSRF;
use CodeIgniter\Filters\DebugToolbar;
use CodeIgniter\Filters\Honeypot;
use CodeIgniter\Filters\InvalidChars;
use CodeIgniter\Filters\SecureHeaders;

class Filters extends BaseConfig
{
    /**
     * Filter aliases — short names used in routes/globals.
     */
    public array $aliases = [
        'csrf'          => CSRF::class,
        'toolbar'       => DebugToolbar::class,
        'honeypot'      => Honeypot::class,
        'invalidchars'  => InvalidChars::class,
        'secureheaders' => SecureHeaders::class,
        'cors'          => \CodeIgniter\Filters\Cors::class,

        // Custom
        'auth'          => \App\Filters\AuthFilter::class,
        'role'          => \App\Filters\RoleFilter::class,
        'throttle'      => \App\Filters\RateLimitFilter::class,
        'apilog'        => \App\Filters\ApiLogFilter::class,
    ];

    /**
     * Filters that run on EVERY request (unless excepted).
     */
    public array $globals = [
        'before' => [
            // 'honeypot',
            'csrf' => ['except' => ['api/*']],
            'invalidchars',
        ],
        'after' => [
            'toolbar',
            // 'honeypot',
            'secureheaders',
        ],
    ];

    /**
     * Filters by HTTP method.
     */
    public array $methods = [
        // 'POST' => ['csrf'],
        // 'GET'  => [],
    ];

    /**
     * Filters by URI pattern.
     */
    public array $filters = [
        'auth' => [
            'before' => ['admin/*', 'dashboard/*'],
        ],
        'apilog' => [
            'before' => ['api/*'],
            'after'  => ['api/*'],
        ],
    ];
}
```

## Using Filters in Routes

```php
// Single filter
$routes->group('admin', ['filter' => 'auth'], static function ($routes) {
    $routes->get('dashboard', 'Admin\Dashboard::index');
});

// With arguments (after the colon)
$routes->group('admin', ['filter' => 'role:admin,superadmin'], static function ($routes) {
    $routes->get('settings', 'Admin\Settings::index');
});

// Multiple filters
$routes->group('api', ['filter' => ['cors', 'tokens', 'throttle:60,60']], function ($routes) {
    $routes->resource('posts');
});

// On a single route
$routes->get('admin/dashboard', 'Admin\Dashboard::index', ['filter' => 'auth']);
```

## Common Filter Recipes

### CORS for SPA

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class CorsFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $response = service('response');
        $origin   = $request->getHeaderLine('Origin');

        $allowed = ['http://localhost:5173', 'https://app.example.com'];

        if (in_array($origin, $allowed, true)) {
            $response->setHeader('Access-Control-Allow-Origin', $origin);
        }
        $response->setHeader('Access-Control-Allow-Credentials', 'true');
        $response->setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,PATCH,DELETE,OPTIONS');
        $response->setHeader('Access-Control-Allow-Headers', 'Authorization,Content-Type,Accept');
        $response->setHeader('Access-Control-Max-Age', '7200');

        // Handle OPTIONS preflight
        if ($request->getMethod() === 'OPTIONS') {
            return $response->setStatusCode(204);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

> **Note:** CodeIgniter 4.5+ ships with a built-in `Cors` filter. Use the built-in version unless you need custom logic.

### Rate Limiting

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class RateLimitFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $throttler = service('throttler');
        $ip        = $request->getIPAddress();
        $capacity  = (int) ($arguments[0] ?? 60);
        $seconds   = (int) ($arguments[1] ?? 60);

        if (! $throttler->check($ip, $capacity, $seconds, 1)) {
            return service('response')
                ->setStatusCode(429)
                ->setHeader('Retry-After', (string) $throttler->getTokenTime())
                ->setJSON(['error' => 'Too many requests']);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Use it: `'filter' => 'throttle:100,60'` (100 requests per 60 seconds per IP).

### API Request/Response Logging

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiLogFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        log_message('info', sprintf(
            '[API] %s %s — IP: %s, UA: %s',
            $request->getMethod(),
            (string) $request->getUri(),
            $request->getIPAddress(),
            $request->getUserAgent()
        ));
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        log_message('info', sprintf(
            '[API] Response %d for %s',
            $response->getStatusCode(),
            (string) $request->getUri()
        ));
    }
}
```

### Force HTTPS in production

`app/Config/App.php`:

```php
public bool $forceGlobalSecureRequests = true;  // production only
```

Or apply per-route:

```php
$routes->group('checkout', ['filter' => 'forcehttps'], function ($routes) { ... });
```

### Maintenance Mode

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class MaintenanceFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (file_exists(WRITEPATH . 'maintenance.flag')) {
            $allowedIps = ['127.0.0.1', '203.0.113.10'];   // Allowlist
            if (! in_array($request->getIPAddress(), $allowedIps, true)) {
                return service('response')
                    ->setStatusCode(503)
                    ->setHeader('Retry-After', '3600')
                    ->setBody(view('errors/maintenance'));
            }
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Trigger: `touch writable/maintenance.flag`. Disable: `rm writable/maintenance.flag`.

## Best Practices

- **Keep filters lightweight** — they run on every request they target
- **Use the built-in `cors` filter** in CI 4.5+ rather than rolling your own
- Apply **`csrf` globally except for `api/*`** in `globals` array
- Apply **rate limiting** on auth endpoints to prevent brute force
- Use **filter arguments** for parameterized behavior (`role:admin`)
- **Log API access** with a custom filter (audit trail)
- **Order matters** — filters in `globals.before` run in defined order
- Place **performance-critical filters first** (auth before logging)
- Use **`except`** to skip globals on specific paths
