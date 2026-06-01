# 09 - Migrations & Seeds

Database schema versioning and data seeding.

## Generating a Migration

```bash
php spark make:migration CreateUsersTable
# Creates: app/Database/Migrations/2026-06-01-120000_CreateUsersTable.php
```

CI4 timestamps migrations automatically. They run in chronological order.

## Migration Anatomy

```php
<?php

declare(strict_types=1);

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateUsersTable extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id' => [
                'type'           => 'BIGINT',
                'unsigned'       => true,
                'auto_increment' => true,
            ],
            'name' => [
                'type'       => 'VARCHAR',
                'constraint' => 255,
            ],
            'email' => [
                'type'       => 'VARCHAR',
                'constraint' => 255,
            ],
            'password' => [
                'type'       => 'VARCHAR',
                'constraint' => 255,
            ],
            'role_id' => [
                'type'     => 'INT',
                'unsigned' => true,
                'null'     => true,
            ],
            'is_active' => [
                'type'    => 'TINYINT',
                'default' => 1,
            ],
            'metadata' => [
                'type' => 'JSON',
                'null' => true,
            ],
            'created_at datetime null',
            'updated_at datetime null',
            'deleted_at datetime null',
        ]);

        $this->forge->addPrimaryKey('id');
        $this->forge->addUniqueKey('email');
        $this->forge->addKey('role_id');                  // Index
        $this->forge->addKey(['is_active', 'deleted_at']); // Composite

        $this->forge->addForeignKey('role_id', 'roles', 'id', 'CASCADE', 'SET NULL');

        $this->forge->createTable('users', true, [        // true = IF NOT EXISTS
            'ENGINE'  => 'InnoDB',
            'CHARSET' => 'utf8mb4',
            'COLLATE' => 'utf8mb4_unicode_ci',
        ]);
    }

    public function down(): void
    {
        $this->forge->dropTable('users');
    }
}
```

## Field Types Reference

```php
'type'           // INT, BIGINT, VARCHAR, TEXT, ENUM, JSON, DATETIME, DECIMAL, etc.
'constraint'     // Length or array of values for ENUM
'unsigned'       // true | false
'auto_increment' // true | false
'null'           // true | false (default false)
'default'        // Default value
'unique'         // true to add a UNIQUE constraint inline
'comment'        // Column comment
'after'          // Place after another column (ALTER only)
'first'          // Place first column (ALTER only)
'collation'      // Per-column collation
```

Common patterns:

```php
// Money
'amount' => [
    'type'       => 'DECIMAL',
    'constraint' => '10,2',
    'default'    => 0.00,
],

// Enum
'status' => [
    'type'       => 'ENUM',
    'constraint' => ['draft', 'published', 'archived'],
    'default'    => 'draft',
],

// Foreign key column
'user_id' => [
    'type'     => 'BIGINT',
    'unsigned' => true,
    'null'     => false,
],

// Soft delete column
'deleted_at' => [
    'type' => 'DATETIME',
    'null' => true,
],

// JSON column (MySQL 5.7+, PostgreSQL)
'preferences' => [
    'type' => 'JSON',
    'null' => true,
],
```

## Modifying Tables (ALTER)

```php
public function up(): void
{
    // Add column
    $this->forge->addColumn('users', [
        'phone' => [
            'type'       => 'VARCHAR',
            'constraint' => 20,
            'null'       => true,
            'after'      => 'email',
        ],
    ]);

    // Modify column
    $this->forge->modifyColumn('users', [
        'name' => [
            'name'       => 'name',           // keep same name
            'type'       => 'VARCHAR',
            'constraint' => 500,              // change length
            'null'       => false,
        ],
    ]);

    // Rename column (4.5+)
    // Old way: pass new name as 'name' inside modifyColumn
    $this->forge->modifyColumn('users', [
        'phone' => [
            'name'       => 'phone_number',   // new name
            'type'       => 'VARCHAR',
            'constraint' => 20,
            'null'       => true,
        ],
    ]);

    // Drop column
    $this->forge->dropColumn('users', 'metadata');

    // Add key
    $this->forge->addKey('email', false, true, 'idx_users_email');
    $this->forge->processIndexes('users');
}

public function down(): void
{
    $this->forge->dropColumn('users', 'phone');
    $this->forge->dropColumn('users', 'phone_number');
}
```

## Foreign Keys

```php
$this->forge->addForeignKey(
    'user_id',          // Local field
    'users',            // Foreign table
    'id',               // Foreign field
    'CASCADE',          // ON DELETE: CASCADE | SET NULL | RESTRICT | NO ACTION
    'CASCADE'           // ON UPDATE
);

$this->forge->createTable('posts');

// Drop foreign key (must drop before dropping column)
$this->forge->dropForeignKey('posts', 'posts_user_id_foreign');
```

## Running Migrations

```bash
php spark migrate                    # Run all pending
php spark migrate --all              # Including from packages (Shield, etc.)
php spark migrate -g default         # Specific DB group
php spark migrate -n App             # Specific namespace
php spark migrate:rollback           # Rollback last batch
php spark migrate:rollback -b 3      # Rollback to batch number 3
php spark migrate:refresh            # Drop everything + re-run all
php spark migrate:refresh --seed UserSeeder
php spark migrate:status             # Show what's run / pending
```

## Seeders

Generate:

```bash
php spark make:seeder UserSeeder
```

Example:

```php
<?php

declare(strict_types=1);

namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        $data = [
            [
                'name'      => 'Admin',
                'email'     => 'admin@example.com',
                'password'  => password_hash('admin123', PASSWORD_DEFAULT),
                'role_id'   => 1,
                'is_active' => 1,
            ],
            [
                'name'      => 'Demo User',
                'email'     => 'demo@example.com',
                'password'  => password_hash('demo123', PASSWORD_DEFAULT),
                'role_id'   => 2,
                'is_active' => 1,
            ],
        ];

        // Bulk insert
        $this->db->table('users')->insertBatch($data);

        // Or via model (triggers callbacks/validation)
        // model('UserModel')->insertBatch($data);
    }
}
```

Run:

```bash
php spark db:seed UserSeeder
```

## DatabaseSeeder Pattern

A master seeder that calls others:

```php
<?php

namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call('RoleSeeder');
        $this->call('UserSeeder');
        $this->call('CategorySeeder');
        $this->call('PostSeeder');
        $this->call('TagSeeder');
    }
}
```

Run all at once: `php spark db:seed DatabaseSeeder`

## Faker for Realistic Test Data

Install Faker:

```bash
composer require --dev fakerphp/faker
```

```php
<?php

namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;
use Faker\Factory;

class FakeUserSeeder extends Seeder
{
    public function run(): void
    {
        $faker = Factory::create();
        $users = [];

        for ($i = 0; $i < 100; $i++) {
            $users[] = [
                'name'       => $faker->name(),
                'email'      => $faker->unique()->safeEmail(),
                'password'   => password_hash('password', PASSWORD_DEFAULT),
                'role_id'    => $faker->numberBetween(1, 3),
                'is_active'  => 1,
                'created_at' => $faker->dateTimeBetween('-1 year')->format('Y-m-d H:i:s'),
            ];
        }

        $this->db->table('users')->insertBatch($users);
    }
}
```

## Migrations from Packages (e.g. Shield)

```bash
# Run Shield migrations along with yours
php spark migrate --all
```

## Best Practices

- **Always provide a working `down()`** so you can roll back
- **Use timestamps** in migration names (auto by `make:migration`)
- **Use `BIGINT UNSIGNED`** for primary keys (future-proof)
- **Add indexes** to FK columns AND any column used in `WHERE`/`ORDER BY`
- **Use `utf8mb4_unicode_ci`** collation (handles emoji, all languages)
- **One migration per change** — don't bundle unrelated changes
- **Don't edit committed migrations** — create a new one to fix issues
- **Seed reference data** with seeders (roles, settings, statuses)
- **Seed development fixtures** with Faker (lots of fake users/posts)
- **Production:** never run `migrate:refresh` (it drops all data!)
- **Production:** run `migrate` only — and have backups before
