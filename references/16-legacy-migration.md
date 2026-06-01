# 16 - Legacy CI3 → CI4 Migration

Strategies for migrating an existing CodeIgniter 3 application to CodeIgniter 4.

## Reality Check

**CI4 is a complete rewrite — not a drop-in upgrade.** Your CI3 code WILL NOT run on CI4 without rewriting. Plan for a real migration project.

Two strategies:
1. **Full rewrite (greenfield)** — Best for small/mid-size apps
2. **Strangler Fig (gradual)** — Best for large apps in production

## Strategy 1: Full Rewrite

Bigger upfront effort, cleaner result.

### Steps

1. **Inventory the CI3 app**
   - List all controllers, models, libraries, helpers, views
   - Document URL routes
   - Document database schema
   - Identify third-party dependencies

2. **Set up a fresh CI4 project**
   ```bash
   composer create-project codeigniter4/appstarter myapp-v2
   ```

3. **Migrate the schema**
   ```bash
   php spark make:migration CreateUsersTable
   # Translate CI3 schema to CI4 migrations
   php spark migrate
   ```

4. **Rewrite controllers** (see "Code Translation Map" below)

5. **Rewrite models** using `CodeIgniter\Model`

6. **Rewrite views** with `esc()`, layouts, sections

7. **Migrate data** with a one-time script:
   ```bash
   php spark make:command MigrateOldData
   ```

8. **Run side-by-side** with both apps for testing

9. **Cut over** when feature parity is reached

## Strategy 2: Strangler Fig (Gradual)

Run both versions side-by-side, route migrated paths to CI4.

### Architecture

```
Internet
    │
    ▼
┌──────────────┐
│   Caddy /    │
│   Nginx      │
└──────┬───────┘
       │
       ├─── /v2/* ─────► CI4 (new code)
       │
       └─── /* ────────► CI3 (legacy)
```

### Caddy example

```caddyfile
example.com {
    # New CI4 paths
    @v2 path /api/v2/* /admin-v2/* /new-feature/*
    handle @v2 {
        reverse_proxy ci4-app:80
    }

    # Everything else goes to legacy CI3
    handle {
        reverse_proxy ci3-app:80
    }
}
```

### Migration Workflow

1. Pick a small, isolated feature (e.g., the `/api/products` endpoint)
2. Reimplement in CI4
3. Update reverse proxy to route that path to CI4
4. Test in production with feature flags
5. Once stable, repeat with next feature
6. When CI3 has 0 active routes left, remove it

## Code Translation Map

### Loading Resources

```php
// CI3
$this->load->model('user_model');
$this->load->library('email');
$this->load->helper('url');
$this->load->view('users/profile', $data);

// CI4
$users = model('UserModel');
$email = service('email');
helper('url');                            // or in $helpers controller property
return view('users/profile', $data);
```

### Database

```php
// CI3
$this->db->select('id, name')
         ->from('users')
         ->where('status', 'active')
         ->get()->result_array();

// CI4
$db = \Config\Database::connect();
$db->table('users')
   ->select('id, name')
   ->where('status', 'active')
   ->get()
   ->getResultArray();
```

### Models

```php
// CI3
class User_model extends CI_Model {
    public function get_all() {
        return $this->db->get('users')->result_array();
    }
}

// CI4
class UserModel extends \CodeIgniter\Model {
    protected $table         = 'users';
    protected $allowedFields = ['name', 'email'];
    // findAll(), find(), insert(), update(), delete() come for free
}
```

### Controllers

```php
// CI3
class Welcome extends CI_Controller {
    public function index() {
        $data['title'] = 'Hello';
        $this->load->view('welcome', $data);
    }
}

// CI4
namespace App\Controllers;

class Welcome extends BaseController {
    public function index(): string {
        return view('welcome', ['title' => 'Hello']);
    }
}
```

### Routes

```php
// CI3 - application/config/routes.php
$route['default_controller'] = 'welcome';
$route['users/(:num)'] = 'users/show/$1';

// CI4 - app/Config/Routes.php
$routes->get('/', 'Welcome::index');
$routes->get('users/(:num)', 'Users::show/$1');
```

### Configuration

```php
// CI3
$this->config->load('app');
$baseUrl = $this->config->item('base_url');

// CI4
$baseUrl = config('App')->baseURL;
// Or via .env: app.baseURL = '...'
```

### Sessions

```php
// CI3
$this->session->set_userdata('user_id', 1);
$id = $this->session->userdata('user_id');
$this->session->unset_userdata('user_id');

// CI4
session()->set('user_id', 1);
$id = session('user_id');
session()->remove('user_id');
```

### Form Validation

```php
// CI3
$this->load->library('form_validation');
$this->form_validation->set_rules('email', 'Email', 'required|valid_email');
if ($this->form_validation->run()) { ... }

// CI4
$rules = ['email' => 'required|valid_email'];
if ($this->validate($rules)) { ... }
```

### Input Class

```php
// CI3
$name = $this->input->post('name');
$ip   = $this->input->ip_address();

// CI4
$name = $this->request->getPost('name');
$ip   = $this->request->getIPAddress();
```

### Hooks → Events

```php
// CI3 - application/config/hooks.php
$hook['post_controller_constructor'] = [
    'class'    => 'MyHook',
    'function' => 'run',
    'filename' => 'MyHook.php',
    'filepath' => 'hooks',
];

// CI4 - app/Config/Events.php
\CodeIgniter\Events\Events::on('post_controller_constructor', static function () {
    // ...
});
```

### Encryption

```php
// CI3
$this->load->library('encryption');
$encoded = $this->encryption->encrypt($data);
$decoded = $this->encryption->decrypt($encoded);

// CI4 - if you need to read CI3 encrypted data:
$config = new \Config\Encryption();
$config->driver = 'OpenSSL';
$config->key    = hex2bin('YOUR_CI3_ENCRYPTION_KEY');
$config->cipher = 'AES-128-CBC';
$config->rawData = false;
$config->encryptKeyInfo = 'encryption';
$config->authKeyInfo    = 'authentication';

$encrypter = service('encrypter', $config);
$decoded   = $encrypter->decrypt($encoded);
```

## Auth Migration: CI3 → Shield

If you used Ion Auth, Tank Auth, or a custom auth in CI3:

1. Install Shield in CI4
2. Run Shield migrations
3. Migrate users with a custom seeder:

```php
// app/Database/Seeds/MigrateUsersFromCI3Seeder.php
namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class MigrateUsersFromCI3Seeder extends Seeder
{
    public function run(): void
    {
        // Connect to old CI3 database
        $oldDb = \Config\Database::connect('legacy');

        $oldUsers = $oldDb->table('users')->get()->getResultArray();

        $userProvider = auth()->getProvider();

        foreach ($oldUsers as $row) {
            $user = new \CodeIgniter\Shield\Entities\User([
                'username' => $row['username'],
                'email'    => $row['email'],
            ]);
            $userProvider->save($user);

            // CI3 typically used bcrypt — Shield supports it
            $user = $userProvider->findById($userProvider->getInsertID());
            $user->password_hash = $row['password'];   // Already-hashed BCrypt password
            $userProvider->save($user);

            // Assign group
            $user->addGroup($row['role'] ?? 'user');
        }
    }
}
```

Users login normally — Shield re-hashes on next successful login.

## URL Compatibility

To keep old CI3 URLs working, use route mapping:

```php
// app/Config/Routes.php

// Old CI3 URL → new CI4 controller
$routes->get('user/profile/(:num)',     'UserController::profile/$1');   // CI3 style
$routes->get('users/(:num)/profile',    'UserController::profile/$1');   // CI4 style

// Or 301 redirects for SEO
$routes->get('user/profile/(:num)', static function ($id) {
    return redirect()->to("/users/{$id}/profile", 301);
});
```

## Database Coexistence

Run CI3 and CI4 against the same DB temporarily — both will work if the schema doesn't change.

For new tables, only add them via CI4 migrations.

## Migration Checklist

- [ ] Inventory all CI3 controllers, models, libraries
- [ ] Document all URLs and route them in CI4
- [ ] Document database schema
- [ ] Set up CI4 starter project
- [ ] Migrate schema with CI4 migrations
- [ ] Rewrite models extending `CodeIgniter\Model`
- [ ] Rewrite controllers extending `BaseController`
- [ ] Rewrite views with `esc()` and layouts
- [ ] Replace CI3 hooks with CI4 events
- [ ] Migrate auth to Shield
- [ ] Set up reverse proxy for gradual cutover (optional)
- [ ] Write tests for the new code
- [ ] Performance test before cutover
- [ ] Have a rollback plan

## What to Drop

Things from CI3 you should not bring forward:

- **MY_Controller / MY_Model with global state** — use Services and DI instead
- **Hooks** — use Events
- **`$this->load->...`** — use service container, model(), helper()
- **String-typed routes (`$route['x']`)** — use HTTP verb routes
- **Custom Form Validation library extensions** — use CI4 custom rule classes
- **CI3 encryption library config** — only keep enough to decrypt existing data once
- **CI_Session in DB without proper schema** — Shield handles this properly

## Best Practices for Migration

- **Don't migrate and refactor at the same time** — first port, then improve
- **Use version control religiously** — small commits, feature branches
- **Write tests first** for the new CI4 code
- **Run side-by-side** for at least one billing/usage cycle
- **Monitor errors** during cutover with Sentry/Rollbar
- **Have a rollback plan** at the reverse-proxy layer (route back to CI3)
- **Communicate with the team** — schema changes need migration files
