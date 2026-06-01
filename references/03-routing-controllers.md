# 03 - Routing & Controllers

Complete guide to CodeIgniter 4 routing system and controllers.

## Routes Configuration

All routes live in `app/Config/Routes.php`.

```php
<?php

use CodeIgniter\Router\RouteCollection;

/**
 * @var RouteCollection $routes
 */

// HTTP verb routes (preferred over $routes->add for security)
$routes->get('/',           'Home::index');
$routes->get('about',       'PageController::about');
$routes->get('contact',     'PageController::contact');
$routes->post('contact',    'PageController::sendContact');

// Route parameters
$routes->get('users/(:num)',     'UserController::show/$1');
$routes->get('posts/(:segment)', 'PostController::show/$1');
$routes->get('search/(:any)',    'SearchController::query/$1');

// Named routes (use route_to() helper)
$routes->get('login', 'AuthController::login', ['as' => 'login']);
// In code: redirect()->route('login');
// In view: <?= route_to('login') ?>
```

## Route Placeholders

| Placeholder | Matches |
|-------------|---------|
| `(:any)` | Any character (greedy, matches `/`) |
| `(:segment)` | Any character except `/` |
| `(:num)` | Numbers only |
| `(:alpha)` | Alphabetic characters |
| `(:alphanum)` | Alphabetic + numbers |
| `(:hash)` | Synonym for `(:segment)` |

Custom placeholder:

```php
$routes->addPlaceholder('uuid', '[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}');
$routes->get('orders/(:uuid)', 'OrderController::show/$1');
```

## Route Groups

```php
// Prefix + namespace
$routes->group('admin', ['namespace' => 'App\Controllers\Admin'], static function ($routes) {
    $routes->get('dashboard', 'Dashboard::index');
    $routes->get('users',     'UserController::index');
    $routes->resource('posts');
});

// With filter
$routes->group('admin', [
    'namespace' => 'App\Controllers\Admin',
    'filter'    => 'session:admin,superadmin',
], static function ($routes) {
    $routes->get('dashboard', 'Dashboard::index');
});

// Subdomain routing
$routes->group('', ['hostname' => 'api.example.com'], static function ($routes) {
    $routes->resource('users');
});
```

## Resource Routes (REST)

```php
$routes->resource('photos');
```

Generates:

| HTTP | Path | Controller method |
|------|------|------------------|
| GET | `/photos` | `index()` |
| GET | `/photos/new` | `new()` |
| POST | `/photos` | `create()` |
| GET | `/photos/(:segment)` | `show($id)` |
| GET | `/photos/(:segment)/edit` | `edit($id)` |
| PUT/PATCH | `/photos/(:segment)` | `update($id)` |
| DELETE | `/photos/(:segment)` | `delete($id)` |

API resource (no `new`/`edit` HTML routes):

```php
$routes->resource('api/photos', ['controller' => 'Api\Photos']);
```

Limit methods:

```php
$routes->resource('photos', ['only' => ['index', 'show']]);
$routes->resource('photos', ['except' => ['delete']]);
```

## Verb Routes Cheat Sheet

```php
$routes->get('path', 'Controller::method');
$routes->post('path', 'Controller::method');
$routes->put('path', 'Controller::method');
$routes->patch('path', 'Controller::method');
$routes->delete('path', 'Controller::method');
$routes->options('path', 'Controller::method');
$routes->head('path', 'Controller::method');

// Multiple verbs
$routes->match(['get', 'post'], 'path', 'Controller::method');
```

## Controllers — BaseController

All controllers should extend `App\Controllers\BaseController`:

```php
<?php

declare(strict_types=1);

namespace App\Controllers;

use CodeIgniter\Controller;
use CodeIgniter\HTTP\CLIRequest;
use CodeIgniter\HTTP\IncomingRequest;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use Psr\Log\LoggerInterface;

abstract class BaseController extends Controller
{
    protected $request;

    /**
     * Helpers loaded for every controller extending this base.
     */
    protected $helpers = ['form', 'url', 'text'];

    /**
     * Be sure to declare properties for any property fetch you initialized.
     */
    protected $session;

    public function initController(
        RequestInterface $request,
        ResponseInterface $response,
        LoggerInterface $logger
    ) {
        parent::initController($request, $response, $logger);

        $this->session = service('session');
    }
}
```

## Web Controller (HTML Response)

```php
<?php

declare(strict_types=1);

namespace App\Controllers;

class PostController extends BaseController
{
    public function index(): string
    {
        $posts = model('PostModel')->paginate(15);

        return view('posts/index', [
            'posts' => $posts,
            'pager' => model('PostModel')->pager,
        ]);
    }

    public function show(int $id): string
    {
        $post = model('PostModel')->find($id);

        if ($post === null) {
            throw \CodeIgniter\Exceptions\PageNotFoundException::forPageNotFound();
        }

        return view('posts/show', ['post' => $post]);
    }

    public function create()
    {
        if (! $this->request->is('post')) {
            return view('posts/create');
        }

        if (! $this->validate('post')) {
            return view('posts/create', [
                'errors' => $this->validator->getErrors(),
            ]);
        }

        $id = model('PostModel')->insert($this->request->getPost());

        return redirect()
            ->to("/posts/{$id}")
            ->with('success', 'Post created.');
    }
}
```

## API Controller — ResponseTrait

```php
<?php

declare(strict_types=1);

namespace App\Controllers\Api;

use App\Controllers\BaseController;
use CodeIgniter\API\ResponseTrait;
use CodeIgniter\HTTP\ResponseInterface;

class Users extends BaseController
{
    use ResponseTrait;

    public function index(): ResponseInterface
    {
        return $this->respond(model('UserModel')->findAll());
    }

    public function show(int $id): ResponseInterface
    {
        $user = model('UserModel')->find($id);

        if ($user === null) {
            return $this->failNotFound("User {$id} not found");
        }

        return $this->respond($user);
    }

    public function create(): ResponseInterface
    {
        $data = $this->request->getJSON(true) ?? $this->request->getPost();

        $userModel = model('UserModel');

        if (! $userModel->insert($data)) {
            return $this->failValidationErrors($userModel->errors());
        }

        return $this->respondCreated([
            'id'      => $userModel->getInsertID(),
            'message' => 'User created',
        ]);
    }

    public function update(int $id): ResponseInterface
    {
        $data      = $this->request->getJSON(true) ?? $this->request->getRawInput();
        $userModel = model('UserModel');

        if ($userModel->find($id) === null) {
            return $this->failNotFound("User {$id} not found");
        }

        if (! $userModel->update($id, $data)) {
            return $this->failValidationErrors($userModel->errors());
        }

        return $this->respond(['message' => 'User updated']);
    }

    public function delete(int $id): ResponseInterface
    {
        $userModel = model('UserModel');

        if ($userModel->find($id) === null) {
            return $this->failNotFound("User {$id} not found");
        }

        $userModel->delete($id);

        return $this->respondDeleted(['message' => 'User deleted']);
    }
}
```

## ResponseTrait Methods

### Success

| Method | Default Status | Use Case |
|--------|---------------|----------|
| `respond($data, $status = 200)` | 200 OK | Generic success |
| `respondCreated($data)` | 201 Created | Resource created |
| `respondDeleted($data)` | 200 OK | Resource deleted |
| `respondNoContent($message = '')` | 204 No Content | No body needed |
| `respondUpdated($data)` | 200 OK | Resource updated |

### Failure

| Method | Status | Use Case |
|--------|--------|----------|
| `fail($errors, $status, $code, $message)` | Custom | Generic failure |
| `failUnauthorized($desc)` | 401 | Not authenticated |
| `failForbidden($desc)` | 403 | Not authorized |
| `failNotFound($desc)` | 404 | Resource missing |
| `failValidationErrors($errors)` | 400 | Validation failed |
| `failResourceExists($desc)` | 409 | Conflict |
| `failResourceGone($desc)` | 410 | Permanently removed |
| `failTooManyRequests($desc)` | 429 | Rate limited |
| `failServerError($desc)` | 500 | Server error |

## ResourceController (Auto-RESTful)

For full CRUD with conventions:

```php
<?php

namespace App\Controllers;

use CodeIgniter\RESTful\ResourceController;

class Photos extends ResourceController
{
    protected $modelName = 'App\Models\PhotoModel';
    protected $format    = 'json';

    public function index()  { return $this->respond($this->model->findAll()); }
    public function show($id = null)
    {
        $photo = $this->model->find($id);
        return $photo
            ? $this->respond($photo)
            : $this->failNotFound();
    }
    public function create()
    {
        $data = $this->request->getJSON(true);
        if (! $this->model->insert($data)) {
            return $this->failValidationErrors($this->model->errors());
        }
        return $this->respondCreated(['id' => $this->model->getInsertID()]);
    }
    public function update($id = null)  { /* ... */ }
    public function delete($id = null)  { /* ... */ }
}
```

## Reading the Request

```php
// Body — JSON
$data = $this->request->getJSON(true);     // true => associative array
$data = $this->request->getJSON();         // object

// Body — form data
$name = $this->request->getPost('name');
$all  = $this->request->getPost();

// Body — raw (PUT/PATCH)
$data = $this->request->getRawInput();

// Query string
$page = $this->request->getGet('page');

// Headers
$auth = $this->request->getHeaderLine('Authorization');

// Files
$file = $this->request->getFile('avatar');

// HTTP method
$method = $this->request->getMethod();     // 'GET' | 'POST' | ...
$is_post = $this->request->is('post');

// IP address
$ip = $this->request->getIPAddress();

// User agent
$ua = $this->request->getUserAgent();
```

## Best Practices

- Use **HTTP verb routes** (`$routes->get/post/...`), avoid `$routes->add()` (less secure)
- Use **`(:segment)` instead of `(:any)`** unless you really need slashes
- Use **route groups** for prefixes, namespaces, and filters
- Use **`ResponseTrait`** for all API controllers
- **Type-hint controller method parameters** (`int $id`, etc.) for automatic conversion
- Throw **`PageNotFoundException`** for missing resources in web controllers
- Return **`failValidationErrors()`** with model errors in APIs
- Use **`route_to('name')`** instead of hardcoded URLs in views/redirects
- Add **named routes** for any URL referenced in views or redirects
