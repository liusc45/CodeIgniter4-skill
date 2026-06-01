# 11 - Testing with PHPUnit

CodeIgniter 4 ships with a PHPUnit-based testing framework specifically tuned for CI4 apps.

## Setup

`phpunit.xml.dist` (already provided in starter):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.5/phpunit.xsd"
         bootstrap="vendor/codeigniter4/framework/system/Test/bootstrap.php"
         colors="true"
         columns="max"
         displayDetailsOnTestsThatTriggerWarnings="true">
    <coverage includeUncoveredFiles="true" pathCoverage="false" ignoreDeprecatedCodeUnits="true">
        <report>
            <clover outputFile="build/logs/clover.xml"/>
            <html outputDirectory="build/logs/html"/>
            <text outputFile="php://stdout" showUncoveredFiles="false"/>
        </report>
    </coverage>
    <testsuites>
        <testsuite name="App">
            <directory>./tests</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory suffix=".php">./app</directory>
        </include>
        <exclude>
            <directory>./app/Views</directory>
            <file>./app/Common.php</file>
        </exclude>
    </source>
</phpunit>
```

## Generate a Test

```bash
php spark make:test UserTest
# Creates: tests/UserTest.php
```

## Test Types

### 1. Unit Test (no DB, no HTTP)

```php
<?php

namespace Tests\Unit;

use CodeIgniter\Test\CIUnitTestCase;
use App\Libraries\PriceCalculator;

final class PriceCalculatorTest extends CIUnitTestCase
{
    public function testCalculatesTaxCorrectly(): void
    {
        $calc = new PriceCalculator(taxRate: 0.21);
        $this->assertEquals(121.00, $calc->withTax(100.00));
    }

    public function testHandlesZero(): void
    {
        $calc = new PriceCalculator(taxRate: 0.21);
        $this->assertEquals(0.00, $calc->withTax(0.00));
    }
}
```

### 2. Database Test (with `DatabaseTestTrait`)

```php
<?php

namespace Tests\Models;

use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;

final class UserModelTest extends CIUnitTestCase
{
    use DatabaseTestTrait;

    /** Run migrations before each test */
    protected $migrate     = true;
    protected $migrateOnce = false;
    /** Seeder to run */
    protected $seed        = 'UserSeeder';
    protected $seedOnce    = false;
    /** Use in-memory SQLite or your test DB */
    protected $namespace   = 'App';

    public function testFindByEmail(): void
    {
        $model = model('UserModel');
        $user  = $model->where('email', 'admin@example.com')->first();
        $this->assertNotNull($user);
        $this->assertEquals('Admin', $user['name']);
    }

    public function testInsertCreatesUser(): void
    {
        $model = model('UserModel');
        $id = $model->insert([
            'name'     => 'New',
            'email'    => 'new@example.com',
            'password' => 'password',
        ]);

        $this->assertIsInt($id);
        $this->seeInDatabase('users', ['email' => 'new@example.com']);
    }

    public function testValidationFailsForInvalidEmail(): void
    {
        $model  = model('UserModel');
        $result = $model->insert([
            'name'     => 'Bad',
            'email'    => 'not-an-email',
            'password' => 'password',
        ]);

        $this->assertFalse($result);
        $this->assertArrayHasKey('email', $model->errors());
    }
}
```

Helper assertions from `DatabaseTestTrait`:

```php
$this->seeInDatabase('users', ['email' => 'foo@bar.com']);
$this->dontSeeInDatabase('users', ['email' => 'deleted@bar.com']);
$this->seeNumRecords(5, 'users', ['is_active' => 1]);
```

### 3. Feature Test (HTTP layer)

Tests routes + controllers + middleware end-to-end:

```php
<?php

namespace Tests\Controllers;

use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;
use CodeIgniter\Test\FeatureTestTrait;

final class UserControllerTest extends CIUnitTestCase
{
    use FeatureTestTrait;
    use DatabaseTestTrait;

    protected $migrate = true;
    protected $seed    = 'UserSeeder';

    public function testIndexReturnsUserList(): void
    {
        $result = $this->get('/api/v1/users');

        $result->assertOK();
        $result->assertStatus(200);
        $result->assertJSONFragment(['email' => 'admin@example.com']);
    }

    public function testShowReturnsUser(): void
    {
        $result = $this->get('/api/v1/users/1');

        $result->assertOK();
        $result->assertJSONFragment(['id' => 1]);
    }

    public function testShowReturns404ForMissingUser(): void
    {
        $result = $this->get('/api/v1/users/9999');
        $result->assertStatus(404);
    }

    public function testCreateUser(): void
    {
        $result = $this->post('/api/v1/users', [
            'name'     => 'Charlie',
            'email'    => 'charlie@example.com',
            'password' => 'password123',
        ]);

        $result->assertStatus(201);
        $this->seeInDatabase('users', ['email' => 'charlie@example.com']);
    }

    public function testCreateUserFailsValidation(): void
    {
        $result = $this->post('/api/v1/users', [
            'email' => 'invalid',
        ]);

        $result->assertStatus(400);
    }

    public function testProtectedRouteRequiresAuth(): void
    {
        $result = $this->get('/admin/dashboard');
        $result->assertRedirect();
        $result->assertRedirectTo('/login');
    }

    public function testAuthenticatedAccess(): void
    {
        $session = ['isLoggedIn' => true, 'role' => 'admin'];

        $result = $this->withSession($session)->get('/admin/dashboard');
        $result->assertOK();
    }
}
```

## FeatureTestTrait Methods

```php
$this->get('/path');                        // GET
$this->post('/path', ['key' => 'value']);   // POST with body
$this->put('/path', $data);
$this->patch('/path', $data);
$this->delete('/path');
$this->call('GET', '/path');                // Generic

// With JSON body
$this->withBody(json_encode($data))
     ->withHeaders(['Content-Type' => 'application/json'])
     ->post('/api/users');

// With session
$this->withSession(['user_id' => 1])->get('/profile');

// With routes (override at runtime)
$this->withRoutes([
    ['get', 'test', 'TestController::index'],
])->get('/test');

// Custom headers
$this->withHeaders([
    'Authorization' => 'Bearer ' . $token,
])->get('/api/me');
```

## Result Assertions

```php
$result->assertOK();                    // 200-299
$result->assertStatus(201);
$result->assertNotOK();
$result->assertRedirect();
$result->assertRedirectTo('/login');
$result->assertSessionHas('user_id');
$result->assertSessionMissing('errors');
$result->assertSee('Welcome');           // String in body
$result->assertDontSee('Error');
$result->assertSeeElement('h1');         // CSS selector
$result->assertHeader('X-Custom-Header');

// JSON
$result->assertJSONFragment(['email' => 'foo@bar.com']);
$result->assertJSONExact($expectedArray);

// Body access
$body = $result->getBody();
$json = $result->getJSON();
$decoded = json_decode($body, true);
```

## Mocking Services

```php
public function testEmailIsSent(): void
{
    $mockEmail = $this->getMockBuilder(\CodeIgniter\Email\Email::class)
        ->disableOriginalConstructor()
        ->getMock();

    $mockEmail->expects($this->once())
              ->method('send')
              ->willReturn(true);

    \Config\Services::injectMock('email', $mockEmail);

    // Trigger code that sends email
    $service = new \App\Libraries\WelcomeMailer();
    $service->send('user@example.com');
}
```

## Mocking Cache

```php
public function testReportIsCached(): void
{
    $mockCache = new \CodeIgniter\Cache\Handlers\DummyHandler(config('Cache'));
    \Config\Services::injectMock('cache', $mockCache);

    $report = service('reportGenerator')->generate(1);

    $this->assertNotNull($mockCache->get('report.user.1'));
}
```

## Faking Time

```php
use CodeIgniter\I18n\Time;

public function testEventScheduledLastWeek(): void
{
    Time::setTestNow('2026-06-01 12:00:00');

    $event = new Event();
    $this->assertTrue($event->isLastWeek());

    Time::setTestNow();   // reset
}
```

## Configuration for Tests

`tests/_support/Database/Seeds/CITestSeeder.php` for test-only data.

`.env.testing` (auto-loaded when `CI_ENVIRONMENT=testing`):

```dotenv
CI_ENVIRONMENT = testing

database.tests.hostname = 127.0.0.1
database.tests.database = ci4_test
database.tests.username = root
database.tests.password = secret
database.tests.DBDriver = MySQLi

# Or use SQLite in-memory
database.tests.DBDriver  = SQLite3
database.tests.database  = :memory:
```

In `phpunit.xml.dist`:

```xml
<php>
    <env name="CI_ENVIRONMENT" value="testing"/>
    <env name="DBGroup" value="tests"/>
</php>
```

## Running Tests

```bash
# All tests
vendor/bin/phpunit

# Single file
vendor/bin/phpunit tests/Models/UserModelTest.php

# Single method
vendor/bin/phpunit --filter testInsertCreatesUser

# With coverage
vendor/bin/phpunit --coverage-html build/coverage

# Stop on first failure
vendor/bin/phpunit --stop-on-failure

# Via spark
php spark test
```

## CI Integration (GitHub Actions)

`.github/workflows/test.yml`:

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: ci4_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping"

    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: intl, mbstring, mysqli, pdo_mysql

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run tests
        run: vendor/bin/phpunit --coverage-text
        env:
          CI_ENVIRONMENT: testing
          database.tests.hostname: 127.0.0.1
          database.tests.username: root
          database.tests.password: secret
```

## Best Practices

- **Use `final class`** for test classes (prevents accidental inheritance)
- **One assertion concept per test** (test method = one behavior)
- Name tests descriptively: `testFailsValidationForMissingEmail`
- **Use `DatabaseTestTrait`** with migrations for DB tests (clean state per test)
- **Use SQLite `:memory:`** for fast test runs (or a dedicated test DB)
- **Mock external services** (email, payment gateways, third-party APIs)
- **Test the happy path AND failure modes**
- **Aim for >80% coverage** of `app/Models`, `app/Controllers`, `app/Libraries`
- **Don't test framework internals** — test YOUR code
- **Run tests in CI** on every PR
