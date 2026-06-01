# 05 - REST API for SPAs (React / Angular / Vue)

How to build a CodeIgniter 4 backend consumed by SPA frontends.

## Architecture Overview

```
┌───────────────────────┐         ┌──────────────────────┐
│   SPA Frontend        │         │   CI4 Backend        │
│   (React/Vue/Angular) │ ◄─────► │   (API only)         │
│   localhost:5173      │  CORS   │   localhost:8080     │
└───────────────────────┘  JSON   └──────────────────────┘
        │                                   │
        │                                   ▼
        │                          ┌──────────────────┐
        │                          │  MySQL/Postgres  │
        │                          └──────────────────┘
        ▼
   Static hosting
   (Vercel/Netlify)
```

Two deployment models:

1. **Separated:** Frontend on `app.example.com`, API on `api.example.com` (CORS required)
2. **Co-hosted:** Frontend built into `public/`, API at `/api/*` (same origin, no CORS)

## API Routes Setup

```php
// app/Config/Routes.php

$routes->group('api/v1', [
    'namespace' => 'App\Controllers\Api\V1',
    'filter'    => 'cors',
], static function ($routes) {
    // Public endpoints
    $routes->post('auth/login',    'AuthController::login');
    $routes->post('auth/register', 'AuthController::register');

    // Protected (token auth via Shield)
    $routes->group('', ['filter' => 'tokens'], static function ($routes) {
        $routes->get('me', 'AuthController::me');

        $routes->resource('users', ['controller' => 'UserController']);
        $routes->resource('posts', ['controller' => 'PostController']);
        $routes->resource('comments', ['controller' => 'CommentController']);
    });

    // OPTIONS routes for CORS preflight
    $routes->options('(:any)', static function () {
        return service('response')->setStatusCode(204);
    });
});
```

## Enabling CORS

CodeIgniter 4.5+ ships with a built-in CORS filter.

### `app/Config/Cors.php`

```php
<?php

namespace Config;

use CodeIgniter\Config\BaseConfig;

class Cors extends BaseConfig
{
    public array $default = [
        'allowedOrigins'         => [],
        'allowedOriginsPatterns' => [],
        'supportsCredentials'    => false,
        'allowedHeaders'         => [],
        'exposedHeaders'         => [],
        'allowedMethods'         => [],
        'maxAge'                 => 7200,
    ];

    // Custom config for your SPA
    public array $api = [
        'allowedOrigins' => [
            'http://localhost:5173',           // Vite dev
            'http://localhost:3000',           // Next.js / CRA dev
            'http://localhost:4200',           // Angular dev
            'https://app.example.com',         // Production
        ],
        'allowedOriginsPatterns' => [
            '#^https://.*\.example\.com$#',     // Wildcard subdomain
        ],
        'supportsCredentials' => true,
        'allowedHeaders' => [
            'Authorization',
            'Content-Type',
            'X-Requested-With',
            'X-CSRF-TOKEN',
            'Accept',
        ],
        'exposedHeaders' => [
            'X-Pagination-Total',
            'X-Pagination-Pages',
        ],
        'allowedMethods' => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
        'maxAge'         => 7200,
    ];
}
```

Apply the named config in routes:

```php
$routes->group('api', ['filter' => 'cors:api'], static function ($routes) {
    // ...
});
```

## Authentication Strategies for SPAs

### Strategy 1: Shield Personal Access Tokens (recommended for mobile/SPA)

Install Shield (see `references/07-shield-auth.md`):

```bash
composer require codeigniter4/shield
php spark shield:setup
php spark migrate --all
```

Login endpoint returns a token:

```php
<?php
// app/Controllers/Api/V1/AuthController.php

namespace App\Controllers\Api\V1;

use App\Controllers\BaseController;
use CodeIgniter\API\ResponseTrait;
use CodeIgniter\Shield\Models\UserModel;

class AuthController extends BaseController
{
    use ResponseTrait;

    public function login()
    {
        $rules = [
            'email'    => 'required|valid_email',
            'password' => 'required|min_length[8]',
        ];

        if (! $this->validate($rules)) {
            return $this->failValidationErrors($this->validator->getErrors());
        }

        $credentials = [
            'email'    => $this->request->getJsonVar('email'),
            'password' => $this->request->getJsonVar('password'),
        ];

        $loginAttempt = auth()->attempt($credentials);

        if (! $loginAttempt->isOK()) {
            return $this->failUnauthorized($loginAttempt->reason());
        }

        $user  = auth()->user();
        $token = $user->generateAccessToken('SPA Token');

        return $this->respond([
            'user' => [
                'id'    => $user->id,
                'email' => $user->email,
            ],
            'token' => $token->raw_token,
        ]);
    }

    public function me()
    {
        $user = auth()->user();
        return $this->respond([
            'id'    => $user->id,
            'email' => $user->email,
        ]);
    }

    public function logout()
    {
        $user = auth()->user();
        $user->revokeAllAccessTokens();

        return $this->respondNoContent();
    }
}
```

Frontend usage:

```js
// React/Angular/Vue
const res = await fetch('http://localhost:8080/api/v1/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password }),
});
const { token } = await res.json();
localStorage.setItem('token', token);

// Authenticated requests
const usersRes = await fetch('http://localhost:8080/api/v1/users', {
  headers: { 'Authorization': `Bearer ${localStorage.getItem('token')}` },
});
```

### Strategy 2: Shield JWT (stateless auth)

```bash
composer require firebase/php-jwt
```

Configure JWT in Shield's config and use the `jwt` filter:

```php
$routes->group('api', ['filter' => 'jwt'], static function ($routes) {
    $routes->resource('users');
});
```

### Strategy 3: Session + CSRF (same-origin SPA only)

If your SPA is served from the same domain:

```php
// Apply CSRF + session
$routes->group('api', ['filter' => ['csrf', 'session']], static function ($routes) {
    $routes->resource('users');
});
```

The frontend reads the CSRF token from a meta tag or cookie and sends it in the `X-CSRF-TOKEN` header.

## Standardized API Response Format

A custom base API controller for consistent responses:

```php
<?php
// app/Controllers/Api/BaseApiController.php

namespace App\Controllers\Api;

use App\Controllers\BaseController;
use CodeIgniter\API\ResponseTrait;
use CodeIgniter\HTTP\ResponseInterface;

class BaseApiController extends BaseController
{
    use ResponseTrait;

    /**
     * Wrap successful responses in a consistent envelope.
     */
    protected function ok(mixed $data = null, int $status = 200, array $meta = []): ResponseInterface
    {
        return $this->respond([
            'success' => true,
            'data'    => $data,
            'meta'    => $meta,
        ], $status);
    }

    /**
     * Wrap error responses in a consistent envelope.
     */
    protected function error(string $message, int $status = 400, array $errors = []): ResponseInterface
    {
        return $this->respond([
            'success' => false,
            'message' => $message,
            'errors'  => $errors,
        ], $status);
    }

    /**
     * Paginated response with pagination metadata in headers.
     */
    protected function paginated(array $items, int $total, int $page, int $perPage): ResponseInterface
    {
        $totalPages = (int) ceil($total / $perPage);

        return $this->ok($items, 200, [
            'pagination' => [
                'total'       => $total,
                'page'        => $page,
                'per_page'    => $perPage,
                'total_pages' => $totalPages,
            ],
        ])
        ->setHeader('X-Pagination-Total', (string) $total)
        ->setHeader('X-Pagination-Pages', (string) $totalPages);
    }
}
```

Use it:

```php
class UserController extends BaseApiController
{
    public function index(): ResponseInterface
    {
        $page    = (int) ($this->request->getGet('page') ?? 1);
        $perPage = (int) ($this->request->getGet('per_page') ?? 20);

        $users = model('UserModel')->paginate($perPage, 'default', $page);
        $total = model('UserModel')->countAllResults();

        return $this->paginated($users, $total, $page, $perPage);
    }
}
```

Response shape:

```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ],
  "meta": {
    "pagination": {
      "total": 142,
      "page": 1,
      "per_page": 20,
      "total_pages": 8
    }
  }
}
```

## API Versioning

```php
// /api/v1/users
$routes->group('api/v1', ['namespace' => 'App\Controllers\Api\V1'], function ($routes) {
    $routes->resource('users');
});

// /api/v2/users (breaking changes go here)
$routes->group('api/v2', ['namespace' => 'App\Controllers\Api\V2'], function ($routes) {
    $routes->resource('users');
});
```

## Rate Limiting

Use Shield's `auth-rates` filter or build a custom one:

```php
<?php
// app/Filters/RateLimitFilter.php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class RateLimitFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $throttler = service('throttler');

        $ip       = $request->getIPAddress();
        $capacity = (int) ($arguments[0] ?? 60); // requests
        $seconds  = (int) ($arguments[1] ?? 60); // per N seconds

        if (! $throttler->check($ip, $capacity, $seconds, 1)) {
            return service('response')
                ->setStatusCode(429)
                ->setJSON([
                    'error'        => 'Too many requests',
                    'retry_after' => $throttler->getTokenTime(),
                ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Register in `app/Config/Filters.php`:

```php
public array $aliases = [
    // ...
    'throttle' => \App\Filters\RateLimitFilter::class,
];
```

Apply: `$routes->group('api', ['filter' => 'throttle:100,60'], ...)` — 100 req/min.

## Frontend Examples

### React (with fetch)

```jsx
// api.js
const API_URL = import.meta.env.VITE_API_URL;

export async function apiFetch(path, options = {}) {
  const token = localStorage.getItem('token');
  const res = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      ...(token ? { 'Authorization': `Bearer ${token}` } : {}),
      ...(options.headers || {}),
    },
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

// Usage
const { data: users } = await apiFetch('/users');
```

### Angular (HttpClient + Interceptor)

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
  }
  return next(req);
};

// users.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}
  list() {
    return this.http.get<{ data: User[] }>('/api/v1/users');
  }
}
```

### Vue (composable)

```ts
// useApi.ts
export function useApi() {
  const baseUrl = import.meta.env.VITE_API_URL;

  async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
    const token = localStorage.getItem('token');
    const res = await fetch(`${baseUrl}${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...options.headers,
      },
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  }
  return { request };
}
```

## Best Practices for SPA Backends

- Use **`api/v1/`** prefix from day one (versioning is cheap insurance)
- Always **enable CORS** with explicit allowed origins (no `*` in production)
- Use **Personal Access Tokens** (Shield) for SPAs — simple and secure
- **Never** put credentials in query strings — use `Authorization` header
- Return **consistent error shapes** (`{ success, message, errors }`)
- Use **HTTP status codes correctly** (201, 204, 400, 401, 403, 404, 422)
- Add **rate limiting** on auth endpoints
- Document with **OpenAPI/Swagger** (PHP package: `zircote/swagger-php`)
- Set **`maxAge`** in CORS to reduce preflight overhead
- **Don't return passwords** — even hashed ones — in any response
