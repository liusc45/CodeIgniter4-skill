# 01 - Installation & Setup

Complete guide for installing CodeIgniter 4 and configuring the environment.

## Requirements

- **PHP 8.1+** (PHP 8.2+ recommended for CI 4.5+)
- **Composer 2.x**
- PHP extensions: `intl`, `mbstring`, `json`, `mysqlnd` (or your DB driver), `curl`, `xml`
- Optional: `gd` or `imagick` (image manipulation), `redis`, `memcached`

Verify your environment:

```bash
php -v
php -m | grep -iE "intl|mbstring|curl|mysqlnd|gd"
composer --version
```

## Installation Methods

### Method 1: Composer (recommended)

```bash
composer create-project codeigniter4/appstarter project-name
cd project-name
```

This creates a fully working starter app with `composer.json`, `app/`, `public/`, `writable/`, `system/`, and `tests/`.

### Method 2: Manual download

Download the release zip from https://codeigniter.com/download and unzip it. Then:

```bash
cd project-name
composer install
```

## Initial Configuration

### 1. Create the `.env` file

```bash
cp env .env
```

Edit `.env`:

```dotenv
#--------------------------------------------------------------------
# ENVIRONMENT
#--------------------------------------------------------------------
CI_ENVIRONMENT = development

#--------------------------------------------------------------------
# APP
#--------------------------------------------------------------------
app.baseURL = 'http://localhost:8080/'
app.indexPage = ''
app.forceGlobalSecureRequests = false
app.sessionDriver = 'CodeIgniter\Session\Handlers\FileHandler'
app.sessionCookieName = 'ci_session'
app.sessionExpiration = 7200
app.CSPEnabled = false

#--------------------------------------------------------------------
# DATABASE
#--------------------------------------------------------------------
database.default.hostname = 127.0.0.1
database.default.database = ci4_app
database.default.username = root
database.default.password = secret
database.default.DBDriver = MySQLi
database.default.DBPrefix =
database.default.port = 3306

#--------------------------------------------------------------------
# ENCRYPTION
#--------------------------------------------------------------------
encryption.key = hex2bin:GENERATE_ME_WITH_SPARK_KEY_GENERATE

#--------------------------------------------------------------------
# LOGGER
#--------------------------------------------------------------------
logger.threshold = 4
```

### 2. Generate encryption key

```bash
php spark key:generate
```

This writes a secure key to `.env` automatically.

### 3. Set writable permissions

```bash
chmod -R 775 writable/
```

On production with separate web user:

```bash
chown -R www-data:www-data writable/
find writable -type d -exec chmod 775 {} \;
find writable -type f -exec chmod 664 {} \;
```

### 4. Run the development server

```bash
php spark serve
# Custom host/port:
php spark serve --host=0.0.0.0 --port=8080
```

Open `http://localhost:8080`. You should see the CodeIgniter welcome page.

## Project Directory Structure

```
project-root/
├── app/                          # Your application code
│   ├── Config/                   # Configuration classes
│   │   ├── App.php               # App-level config (baseURL, timezone)
│   │   ├── Autoload.php          # Autoload helpers/files/namespaces
│   │   ├── Cache.php             # Cache driver config
│   │   ├── Database.php          # Database connections
│   │   ├── Filters.php           # Filter aliases & globals
│   │   ├── Routes.php            # Route definitions
│   │   ├── Services.php          # Service container overrides
│   │   └── Validation.php        # Validation rule groups
│   ├── Controllers/
│   │   ├── BaseController.php    # Base controller all others extend
│   │   └── Home.php              # Default home controller
│   ├── Database/
│   │   ├── Migrations/           # Schema migrations
│   │   └── Seeds/                # Database seeders
│   ├── Entities/                 # Domain entities (optional)
│   ├── Filters/                  # Custom filters (middleware)
│   ├── Helpers/                  # Custom helper functions
│   ├── Language/                 # i18n translations
│   ├── Libraries/                # Custom libraries
│   ├── Models/                   # Models (extend CodeIgniter\Model)
│   ├── ThirdParty/               # Vendor extensions
│   └── Views/                    # PHP view templates
├── public/                       # Document root (web server points here)
│   ├── index.php                 # Front controller
│   ├── .htaccess                 # Apache rewrite rules
│   ├── favicon.ico
│   └── assets/                   # Compiled CSS/JS/images
├── system/                       # Framework core (DO NOT EDIT)
├── tests/                        # PHPUnit tests
├── writable/                     # Logs, cache, sessions, uploads (writable)
│   ├── cache/
│   ├── logs/
│   ├── session/
│   └── uploads/
├── vendor/                       # Composer dependencies
├── .env                          # Environment configuration (DO NOT COMMIT)
├── env                           # Template (commit this)
├── composer.json
├── phpunit.xml.dist
└── spark                         # CLI entry point
```

## Spark CLI — Daily Commands

```bash
# Help
php spark                                   # List all commands
php spark help <command>                    # Help for a command

# Server
php spark serve                             # Start dev server
php spark serve --host=0.0.0.0 --port=8081

# Routes
php spark routes                            # List all routes

# Generators (make:* commands)
php spark make:controller User --restful api-resource --bare
php spark make:model UserModel --return entity --suffix
php spark make:entity User
php spark make:migration CreateUsersTable
php spark make:seeder UserSeeder
php spark make:filter AuthFilter
php spark make:command MyCommand
php spark make:test UserTest
php spark make:cell ProductCell
php spark make:validation Custom

# Database
php spark migrate                           # Run pending migrations
php spark migrate --all                     # Including from packages (Shield, etc.)
php spark migrate:rollback
php spark migrate:rollback -b 5             # Rollback to batch 5
php spark migrate:refresh                   # Drop all + run all
php spark migrate:status
php spark db:seed UserSeeder
php spark db:create my_database
php spark db:table users                    # Inspect a table
php spark db:show                           # Show DB info

# Cache
php spark cache:clear                       # Clear all cache items
php spark cache:info                        # Show cache info

# Configuration optimization (production)
php spark config:cache                      # Cache configs (faster boot)
php spark config:cache --clear
php spark optimize                          # All-in-one optimize

# Environment
php spark env                               # Show current env
php spark env production                    # Switch to production
php spark env development

# Diagnostics
php spark phpini:check                      # Recommended php.ini settings
php spark logs:clear                        # Clear log files
```

## Environment Modes

CodeIgniter supports three built-in environments via `CI_ENVIRONMENT`:

| Mode | Use Case | Behavior |
|------|----------|----------|
| `development` | Local dev | Full error display, debug toolbar enabled |
| `testing` | Automated tests | Errors shown, optimized for PHPUnit |
| `production` | Live deployment | Errors hidden, optimized, no debug |

Set in `.env`:

```dotenv
CI_ENVIRONMENT = development
```

Or per-server (Nginx example):

```nginx
fastcgi_param CI_ENVIRONMENT "production";
```

## Common First-Time Issues

### "Writable directory not writable"

```bash
chmod -R 775 writable/
# Or for stricter setups:
sudo chown -R www-data:www-data writable/
```

### "Class 'Config\Services' not found"

```bash
composer dump-autoload
```

### Routes return 404 on Apache

Ensure `mod_rewrite` is enabled and `AllowOverride All` is set on the `public/` directory.

### "Could not connect to database"

- Verify `database.default.hostname` (use `127.0.0.1`, not `localhost`, on some systems)
- Verify port (3306 for MySQL, 5432 for PostgreSQL)
- Check user privileges and firewall rules
