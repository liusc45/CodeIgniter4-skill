# 02 - Models & Database

Complete guide to CodeIgniter 4 Models, Query Builder, Entities, and database operations.

## Model Anatomy

A complete CI4 model with all common features:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use CodeIgniter\Model;

class UserModel extends Model
{
    // ----- Table configuration -----
    protected $DBGroup          = 'default';
    protected $table            = 'users';
    protected $primaryKey       = 'id';
    protected $useAutoIncrement = true;

    // ----- Return type -----
    protected $returnType = 'array';                  // 'array' | 'object' | 'App\Entities\User'
    protected $useSoftDeletes = true;

    // ----- Mass assignment protection -----
    protected $allowedFields = [
        'name',
        'email',
        'password',
        'status',
        'role_id',
    ];

    // ----- Type casts (CI 4.5+) -----
    protected array $casts = [
        'id'         => 'int',
        'role_id'    => '?int',
        'is_active'  => 'int-bool',
        'metadata'   => 'json-array',
        'birthdate'  => '?datetime',
    ];

    // ----- Timestamps -----
    protected $useTimestamps = true;
    protected $dateFormat    = 'datetime';
    protected $createdField  = 'created_at';
    protected $updatedField  = 'updated_at';
    protected $deletedField  = 'deleted_at';

    // ----- Validation -----
    protected $validationRules = [
        'name'     => 'required|min_length[3]|max_length[255]',
        'email'    => 'required|valid_email|is_unique[users.email,id,{id}]',
        'password' => 'required|min_length[8]',
        'status'   => 'in_list[active,inactive,banned]',
    ];

    protected $validationMessages = [
        'email' => [
            'is_unique' => 'This email is already registered.',
        ],
    ];

    protected $skipValidation     = false;
    protected $cleanValidationRules = true;

    // ----- Callbacks -----
    protected $allowCallbacks = true;
    protected $beforeInsert   = ['hashPassword'];
    protected $beforeUpdate   = ['hashPassword'];

    protected function hashPassword(array $data): array
    {
        if (! isset($data['data']['password'])) {
            return $data;
        }

        $data['data']['password'] = password_hash(
            $data['data']['password'],
            PASSWORD_DEFAULT
        );

        return $data;
    }
}
```

## Accessing Models

```php
// Helper (preferred — uses shared instance)
$users = model('UserModel');
$users = model(UserModel::class);

// Direct instantiation
$users = new \App\Models\UserModel();

// Inside a controller
$this->users = model(UserModel::class);
```

## CRUD Operations

### Read

```php
$users = model('UserModel');

// Single record
$user = $users->find(1);                       // by primary key
$user = $users->where('email', $email)->first();

// Multiple records
$all       = $users->findAll();
$paged     = $users->findAll(20, 40);          // limit 20 offset 40
$emails    = $users->findColumn('email');      // single column array
$active    = $users->where('status', 'active')->findAll();

// Query builder chains
$results = $users
    ->select('id, name, email')
    ->where('status', 'active')
    ->whereIn('role_id', [1, 2])
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->findAll();

// Pagination (with built-in pager)
$results = $users->paginate(20);
$pager   = $users->pager;
```

### Create

```php
$id = $users->insert([
    'name'     => 'Jane Doe',
    'email'    => 'jane@example.com',
    'password' => 'secret123',
]);

// Returns false on validation failure; inspect errors:
if (! $id) {
    return $users->errors();
}

// Batch insert
$users->insertBatch([
    ['name' => 'Alice', 'email' => 'a@x.com', 'password' => 'pw'],
    ['name' => 'Bob',   'email' => 'b@x.com', 'password' => 'pw'],
]);
```

### Update

```php
$users->update(1, ['name' => 'New Name']);

// Multiple
$users->where('status', 'inactive')
      ->set('status', 'active')
      ->update();

// Save (insert if no PK, update if PK present)
$users->save([
    'id'   => 1,
    'name' => 'Updated',
]);
```

### Delete

```php
$users->delete(1);              // soft delete (if enabled)
$users->delete(1, true);        // hard delete (force)
$users->purgeDeleted();         // permanently remove soft-deleted rows

// Restore soft-deleted record
$users->where('id', 1)->set('deleted_at', null)->update();

// Query helpers for soft deletes
$active   = $users->findAll();              // excludes deleted
$all      = $users->withDeleted()->findAll();
$trashed  = $users->onlyDeleted()->findAll();
```

## Query Builder (Direct, without Models)

```php
$db = \Config\Database::connect();

// SELECT
$query = $db->table('users')
    ->select('id, name, email')
    ->where('status', 'active')
    ->orderBy('name')
    ->get();
$results = $query->getResultArray();

// JOIN
$query = $db->table('orders')
    ->select('orders.*, users.name as user_name')
    ->join('users', 'users.id = orders.user_id', 'left')
    ->where('orders.status', 'paid')
    ->get();

// INSERT / UPDATE / DELETE
$db->table('users')->insert(['name' => 'New']);
$db->table('users')->where('id', 1)->update(['name' => 'X']);
$db->table('users')->where('id', 1)->delete();

// Raw query (use parameter binding!)
$query = $db->query('SELECT * FROM users WHERE email = ?', [$email]);
```

## Entities (Domain Objects)

Entities are objects representing one row of data with business logic. Use them when you want behavior, not just data.

```php
<?php

declare(strict_types=1);

namespace App\Entities;

use CodeIgniter\Entity\Entity;
use CodeIgniter\I18n\Time;

class User extends Entity
{
    protected $datamap = [
        'fullName' => 'full_name',  // property => column
    ];

    protected $casts = [
        'id'        => 'int',
        'is_active' => 'boolean',
        'options'   => 'json-array',
    ];

    protected $dates = ['created_at', 'updated_at', 'deleted_at'];

    // Custom setter (auto-hashes password)
    public function setPassword(string $pass): static
    {
        $this->attributes['password'] = password_hash($pass, PASSWORD_BCRYPT);
        return $this;
    }

    // Custom getter
    public function getCreatedAt(string $format = 'Y-m-d H:i:s'): string
    {
        return Time::parse($this->attributes['created_at'])->format($format);
    }

    // Business methods
    public function isAdmin(): bool
    {
        return $this->role_id === 1;
    }

    public function fullDisplayName(): string
    {
        return sprintf('%s <%s>', $this->name, $this->email);
    }
}
```

Use the entity in your model:

```php
class UserModel extends Model
{
    protected $returnType = \App\Entities\User::class;
    // ...
}

// Now find() returns User entity
$user = model('UserModel')->find(1);
$user->setPassword('newpass');           // auto-hashed
echo $user->fullDisplayName();
echo $user->getCreatedAt('d/m/Y');
$saved = model('UserModel')->save($user);
```

## Cast Types Reference

| Cast | PHP Type | Notes |
|------|----------|-------|
| `int` / `integer` | int | |
| `float` / `double` | float | |
| `string` | string | |
| `bool` / `boolean` | bool | |
| `int-bool` | bool stored as 0/1 | DB stores tinyint |
| `array` | array | PHP `serialize()` |
| `csv` | array | comma-separated |
| `json` | object | `json_decode` to object |
| `json-array` | array | `json_decode` to array |
| `datetime` | `Time` | CI's I18n\Time |
| `timestamp` | int | Unix timestamp |
| `uri` | `URI` | parsed URI |

Prefix with `?` for nullable: `?int`, `?datetime`, `?json-array`.

## Database Transactions

```php
$db = \Config\Database::connect();

// Auto rollback on error
$db->transStart();
$db->table('orders')->insert($order);
$db->table('order_items')->insertBatch($items);
$db->transComplete();

if ($db->transStatus() === false) {
    log_message('error', 'Order transaction failed');
    return false;
}

// Manual control
$db->transBegin();
try {
    // ... queries ...
    $db->transCommit();
} catch (\Throwable $e) {
    $db->transRollback();
    throw $e;
}
```

## Multiple Database Connections

`app/Config/Database.php`:

```php
public array $analytics = [
    'DSN'      => '',
    'hostname' => 'analytics.db.internal',
    'username' => 'reader',
    'password' => 'secret',
    'database' => 'analytics',
    'DBDriver' => 'Postgre',
    'DBPrefix' => '',
    'pConnect' => false,
    'DBDebug'  => true,
    'charset'  => 'utf8',
    'port'     => 5432,
];
```

Use it:

```php
$db = \Config\Database::connect('analytics');

// Or in a model:
class EventModel extends Model
{
    protected $DBGroup = 'analytics';
}
```

## Best Practices

- **Always set `$allowedFields`** — without it, no inserts/updates work via the Model
- **Define validation in the Model** — keeps rules close to the data
- **Use Entities for behavior** — keep models focused on data access
- **Soft deletes are opt-in per model** — set `$useSoftDeletes = true` and add `deleted_at` column
- **Use `paginate()` for lists** — built-in pager generates ready-to-render links
- **Use transactions for multi-step writes** — prevents partial state on failure
- **Prefer `model('UserModel')` over `new`** — uses shared instance, less memory
- **Index foreign keys and `WHERE`/`ORDER BY` columns** — `$forge->addKey()` in migrations
