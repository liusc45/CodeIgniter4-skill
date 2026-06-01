# 07 - Shield Authentication

Official authentication & authorization for CodeIgniter 4.

## Installation

```bash
composer require codeigniter4/shield
php spark shield:setup
php spark migrate --all
```

`shield:setup` does the following:

- Publishes config files (`Auth.php`, `AuthGroups.php`, `AuthToken.php`, etc.)
- Updates `app/Config/Routes.php` with auth routes
- Updates `app/Config/Filters.php` with auth filter aliases
- Modifies `app/Controllers/BaseController.php` for helpers

## Filter Aliases (auto-registered)

| Alias | Class | Use Case |
|-------|-------|----------|
| `session` | `SessionAuth` | Session-based login (web apps) |
| `tokens` | `TokenAuth` | Personal Access Tokens (SPAs/mobile) |
| `hmac` | `HmacAuth` | HMAC SHA256 (server-to-server APIs) |
| `jwt` | `JWTAuth` | JWT tokens (stateless APIs) |
| `chain` | `ChainAuth` | Try multiple methods in order |
| `auth-rates` | `AuthRates` | Rate-limit auth endpoints |
| `group` | `GroupFilter` | Group-based access control |
| `permission` | `PermissionFilter` | Permission-based access control |
| `force-reset` | `ForcePasswordResetFilter` | Force password reset on next login |

## Routes Setup

```php
// app/Config/Routes.php

service('auth')->routes($routes);   // Adds default Shield routes
// /login, /logout, /register, /change-password, /forgot, etc.

// Web area (session auth)
$routes->group('admin', ['filter' => 'session'], static function ($routes) {
    $routes->get('dashboard', 'Admin\Dashboard::index');
});

// API area (token auth)
$routes->group('api', ['filter' => 'tokens'], static function ($routes) {
    $routes->resource('users');
});

// Group-based protection
$routes->group('superadmin', ['filter' => 'group:superadmin'], static function ($routes) {
    $routes->get('settings', 'SuperAdmin\Settings::index');
});

// Permission-based protection
$routes->group('posts', ['filter' => 'permission:posts.manage'], static function ($routes) {
    $routes->resource('posts');
});
```

## Session-based Login (Web)

```php
<?php

namespace App\Controllers;

class AuthController extends BaseController
{
    public function login()
    {
        if (! $this->request->is('post')) {
            return view('auth/login');
        }

        $credentials = [
            'email'    => $this->request->getPost('email'),
            'password' => $this->request->getPost('password'),
        ];

        $remember = (bool) $this->request->getPost('remember');

        $loginAttempt = auth()->remember($remember)->attempt($credentials);

        if (! $loginAttempt->isOK()) {
            return redirect()->back()->withInput()->with('error', $loginAttempt->reason());
        }

        return redirect()->to('/dashboard');
    }

    public function logout()
    {
        auth()->logout();
        return redirect()->to('/login')->with('success', 'Logged out');
    }
}
```

## Token-based Auth (API/SPA)

```php
<?php
// Generate token after login
public function login()
{
    $credentials = [
        'email'    => $this->request->getJsonVar('email'),
        'password' => $this->request->getJsonVar('password'),
    ];

    $loginAttempt = auth()->attempt($credentials);
    if (! $loginAttempt->isOK()) {
        return $this->failUnauthorized($loginAttempt->reason());
    }

    $user  = auth()->user();
    $token = $user->generateAccessToken('Mobile App');

    return $this->respond([
        'token'      => $token->raw_token,    // Show ONCE
        'token_name' => $token->name,
        'user'       => ['id' => $user->id, 'email' => $user->email],
    ]);
}
```

### Scoped tokens

```php
// Limit what the token can access
$token = $user->generateAccessToken('Read-only', ['posts.read', 'users.read']);
```

In a route, use the `tokens` filter with scope:

```php
$routes->group('api', ['filter' => 'tokens:posts.read'], function ($routes) {
    $routes->get('posts', 'PostController::index');
});
```

### Expiring tokens

```php
$expiresAt = \CodeIgniter\I18n\Time::now()->addDays(30);
$token = $user->generateAccessToken('Temp Access', ['*'], $expiresAt);
```

### Revoke tokens

```php
$user = auth()->user();
$user->revokeAccessToken('Mobile App');
$user->revokeAllAccessTokens();
```

## JWT Auth (Stateless)

```bash
composer require firebase/php-jwt
```

Configure JWT in `app/Config/AuthJWT.php`:

```php
public string $defaultGroup = 'default';

public array $keys = [
    'default' => [
        [
            'secret'    => 'YOUR_LONG_RANDOM_SECRET_HERE',
            'algorithm' => 'HS256',
        ],
    ],
];

public string $defaultClaims = [
    'iss' => 'https://your-app.example.com',
];

public int $timeToLive = 3600;   // 1 hour
```

Generate a JWT:

```php
$token = service('jwtmanager')->generateToken(auth()->user());
```

Use the `jwt` filter:

```php
$routes->group('api', ['filter' => 'jwt'], function ($routes) {
    $routes->resource('users');
});
```

## Groups & Permissions

`app/Config/AuthGroups.php`:

```php
public array $groups = [
    'superadmin' => [
        'title'       => 'Super Admin',
        'description' => 'Full site control.',
    ],
    'admin' => [
        'title'       => 'Administrator',
        'description' => 'Day-to-day site administration.',
    ],
    'user' => [
        'title'       => 'User',
        'description' => 'Regular user.',
    ],
    'beta' => [
        'title'       => 'Beta User',
        'description' => 'Has access to beta features.',
    ],
];

public array $permissions = [
    'admin.access'   => 'Can access the admin area',
    'admin.settings' => 'Can change site settings',
    'users.manage'   => 'Can manage users',
    'posts.create'   => 'Can create posts',
    'posts.edit'     => 'Can edit posts',
    'posts.delete'   => 'Can delete posts',
    'posts.publish'  => 'Can publish posts',
];

// Default permissions per group
public array $matrix = [
    'superadmin' => ['admin.*', 'users.*', 'posts.*'],
    'admin'      => ['admin.access', 'users.manage', 'posts.*'],
    'user'       => ['posts.create', 'posts.edit'],
];

public string $defaultGroup = 'user';
```

### Manage groups/permissions in code

```php
$user = auth()->user();

// Groups
$user->addGroup('admin');
$user->removeGroup('user');
$user->getGroups();             // ['admin']
$user->inGroup('admin');        // true
$user->syncGroups('admin', 'beta');

// Permissions (additive on top of groups)
$user->addPermission('posts.delete');
$user->removePermission('posts.delete');
$user->getPermissions();
$user->can('posts.delete');     // true if direct OR via group
$user->hasPermission('posts.delete');  // direct only
```

### Spark commands

```bash
php spark shield:user create
php spark shield:user list
php spark shield:user assign -g admin -u user@example.com
php spark shield:user remove -g admin -u user@example.com
php spark shield:user activate user@example.com
php spark shield:user ban user@example.com -m "Spamming"
php spark shield:user unban user@example.com
php spark shield:user password user@example.com   # Reset password
```

## Email Verification (2FA, magic links)

Configure in `app/Config/Auth.php`:

```php
public array $actions = [
    'register' => \CodeIgniter\Shield\Authentication\Actions\EmailActivator::class,
    'login'    => \CodeIgniter\Shield\Authentication\Actions\Email2FA::class,
];
```

Configure email service in `.env`:

```dotenv
email.protocol = smtp
email.SMTPHost = smtp.example.com
email.SMTPUser = noreply@example.com
email.SMTPPass = secret
email.SMTPPort = 587
email.SMTPCrypto = tls
email.fromEmail = noreply@example.com
email.fromName = My App
```

## Helpers

```php
// Always available with shield helper loaded
auth()->loggedIn();         // bool
auth()->user();             // current user object or null
auth()->id();               // current user id or null
auth()->getProvider();      // user provider

user_id();                  // shortcut for auth()->id()

// In controllers
$this->user;                // available in BaseController if extended properly
```

## In Views

```php
<?php if (auth()->loggedIn()): ?>
    <p>Welcome, <?= esc(auth()->user()->username) ?></p>

    <?php if (auth()->user()->inGroup('admin')): ?>
        <a href="/admin">Admin panel</a>
    <?php endif; ?>

    <?php if (auth()->user()->can('posts.create')): ?>
        <a href="/posts/create">New post</a>
    <?php endif; ?>

    <?= form_open('logout') ?>
        <?= csrf_field() ?>
        <button>Logout</button>
    <?= form_close() ?>
<?php else: ?>
    <a href="<?= route_to('login') ?>">Login</a>
<?php endif; ?>
```

## Best Practices

- Use **session auth** for web apps, **tokens** for SPAs/mobile, **JWT** for stateless APIs
- Use **scoped tokens** for least-privilege API access
- Set **expiration** on long-lived tokens
- Always **rate-limit auth endpoints** with `auth-rates` filter
- Use **groups for roles** (admin, editor, user)
- Use **permissions for granular actions** (`posts.delete`, etc.)
- Build user-facing UI with `auth()->user()->can('permission')`
- Enable **email activation** for production registrations
- Use **2FA** (`Email2FA` action) for sensitive accounts
- Never expose Shield's internal `users` table fields in API responses
