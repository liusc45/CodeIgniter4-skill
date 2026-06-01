# 18 - Production Deployment

Pre-flight checklist and procedures to take CodeIgniter 4 to production safely.

## Pre-Deployment Checklist

### Environment

- [ ] `CI_ENVIRONMENT = production` in `.env`
- [ ] `CI_DEBUG = false`
- [ ] `app.baseURL` set to actual production URL (with trailing slash)
- [ ] `app.indexPage = ''` (empty for clean URLs)
- [ ] `app.forceGlobalSecureRequests = true` (HTTPS only)
- [ ] `app.cookieSecure = true`
- [ ] `app.cookieHTTPOnly = true`
- [ ] `app.cookieSameSite = 'Lax'`
- [ ] `app.CSPEnabled = true` (if you've configured CSP)
- [ ] `encryption.key` is generated (`php spark key:generate`)
- [ ] Database credentials configured

### Database

- [ ] All migrations run (`php spark migrate --all`)
- [ ] Production seed data inserted (if any)
- [ ] DB user has minimal required privileges (no `DROP DATABASE` for app user)
- [ ] Backups configured (daily, off-site)
- [ ] Connection pooling configured if needed

### Performance

- [ ] `composer install --no-dev --optimize-autoloader --classmap-authoritative`
- [ ] OPcache enabled (`opcache.enable=1`)
- [ ] OPcache validation off (`opcache.validate_timestamps=0`)
- [ ] OPcache memory ≥ 256MB (`opcache.memory_consumption=256`)
- [ ] Realpath cache tuned (`realpath_cache_size=4M`)
- [ ] Cache backend configured (Redis/Memcached recommended)
- [ ] `php spark config:cache` (if using config caching)
- [ ] Static assets compiled and minified
- [ ] Asset cache busting (file hash in URL)

### Security

- [ ] HTTPS enforced (TLS 1.2+, HSTS preload)
- [ ] Security headers set (X-Frame-Options, CSP, etc.)
- [ ] CSRF enabled for non-API routes
- [ ] Rate limiting on auth endpoints
- [ ] `writable/` not web-accessible
- [ ] `app/`, `system/`, `vendor/` not web-accessible
- [ ] `.env` permissions: `600` (`chmod 600 .env`)
- [ ] Server software updated (PHP, MySQL, Nginx/Caddy)
- [ ] Firewall configured (only ports 80/443 public)
- [ ] SSH key-only access (no password)
- [ ] Fail2ban or similar configured

### Monitoring

- [ ] Error tracking (Sentry, Rollbar, Bugsnag)
- [ ] Application logs aggregated (CloudWatch, Datadog, ELK)
- [ ] Uptime monitoring (UptimeRobot, Pingdom)
- [ ] Performance monitoring (New Relic, Tideways, Blackfire)
- [ ] Log rotation configured (avoid disk full)

### Operations

- [ ] Health check endpoint (`/health` or `/up`)
- [ ] Maintenance mode mechanism in place
- [ ] Rollback plan documented
- [ ] Deployment automation (CI/CD pipeline)
- [ ] Database migration strategy (blue-green or zero-downtime)

## Production `.env` Template

```dotenv
#--------------------------------------------------------------------
# ENVIRONMENT
#--------------------------------------------------------------------
CI_ENVIRONMENT = production
CI_DEBUG = false

#--------------------------------------------------------------------
# APP
#--------------------------------------------------------------------
app.baseURL = 'https://example.com/'
app.indexPage = ''
app.forceGlobalSecureRequests = true
app.proxyIPs = '10.0.0.0/8'   # If behind LB/proxy

# Sessions
app.sessionDriver = 'CodeIgniter\Session\Handlers\RedisHandler'
app.sessionCookieName = 'ci_session'
app.sessionExpiration = 7200
app.sessionSavePath = 'tcp://redis:6379'
app.cookieSecure = true
app.cookieHTTPOnly = true
app.cookieSameSite = 'Lax'

# CSP
app.CSPEnabled = true

#--------------------------------------------------------------------
# DATABASE
#--------------------------------------------------------------------
database.default.hostname = db
database.default.database = ci4_app
database.default.username = ci4_user
database.default.password = strong_random_password_here
database.default.DBDriver = MySQLi
database.default.DBPrefix =
database.default.DBDebug = false   # IMPORTANT: false in production
database.default.charset = utf8mb4
database.default.DBCollat = utf8mb4_unicode_ci

#--------------------------------------------------------------------
# CACHE
#--------------------------------------------------------------------
cache.handler = 'redis'
cache.default = 'redis'
cache.redis.host = 'redis'
cache.redis.port = 6379

#--------------------------------------------------------------------
# ENCRYPTION
#--------------------------------------------------------------------
encryption.key = hex2bin:GENERATED_BY_SPARK_KEY_GENERATE_HERE

#--------------------------------------------------------------------
# LOGGER
#--------------------------------------------------------------------
logger.threshold = 4   # info and below

#--------------------------------------------------------------------
# EMAIL
#--------------------------------------------------------------------
email.protocol = smtp
email.SMTPHost = smtp.example.com
email.SMTPUser = noreply@example.com
email.SMTPPass = secret
email.SMTPPort = 587
email.SMTPCrypto = tls
email.fromEmail = noreply@example.com
email.fromName = "My App"
```

## OPcache Production Config

```ini
; /usr/local/etc/php/conf.d/opcache.ini

[opcache]
opcache.enable = 1
opcache.enable_cli = 0

; Memory
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16

; Files
opcache.max_accelerated_files = 20000
opcache.max_wasted_percentage = 5

; Validation - false = max speed, requires deploy to refresh code
opcache.validate_timestamps = 0
; If you must keep validation on, set high revalidate freq:
; opcache.validate_timestamps = 1
; opcache.revalidate_freq = 60

; Misc
opcache.fast_shutdown = 1
opcache.save_comments = 1     ; required for some attributes/annotations
opcache.enable_file_override = 0

; Preloading (PHP 7.4+) - boots CI4 once on startup
opcache.preload = /var/www/html/preload.php
opcache.preload_user = www-data

; Realpath cache
realpath_cache_size = 4M
realpath_cache_ttl = 600
```

## Preload Script (`preload.php`)

```php
<?php
// Preload CI4's core classes for fastest cold-start
$basePath = __DIR__;
opcache_compile_file($basePath . '/system/Boot.php');
opcache_compile_file($basePath . '/system/CodeIgniter.php');
opcache_compile_file($basePath . '/system/Controller.php');
opcache_compile_file($basePath . '/system/Model.php');

foreach (glob($basePath . '/system/HTTP/*.php') as $f) {
    opcache_compile_file($f);
}
foreach (glob($basePath . '/system/Database/*.php') as $f) {
    opcache_compile_file($f);
}
foreach (glob($basePath . '/system/Validation/*.php') as $f) {
    opcache_compile_file($f);
}
foreach (glob($basePath . '/system/View/*.php') as $f) {
    opcache_compile_file($f);
}
```

## Health Check Endpoint

```php
// app/Config/Routes.php
$routes->get('health', static function () {
    $checks = [
        'app'   => 'ok',
        'db'    => 'ok',
        'cache' => 'ok',
    ];

    try {
        \Config\Database::connect()->query('SELECT 1');
    } catch (\Throwable $e) {
        $checks['db'] = 'fail';
    }

    try {
        service('cache')->save('healthcheck', '1', 5);
    } catch (\Throwable $e) {
        $checks['cache'] = 'fail';
    }

    $allOk = ! in_array('fail', $checks, true);

    return service('response')
        ->setStatusCode($allOk ? 200 : 503)
        ->setJSON([
            'status'    => $allOk ? 'ok' : 'fail',
            'checks'    => $checks,
            'timestamp' => date('c'),
            'version'   => env('APP_VERSION', 'unknown'),
        ]);
});
```

## Maintenance Mode

```php
// app/Filters/MaintenanceFilter.php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class MaintenanceFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (! file_exists(WRITEPATH . 'maintenance.flag')) {
            return;
        }

        $allowedIps = explode(',', env('MAINTENANCE_ALLOWED_IPS', ''));
        if (in_array($request->getIPAddress(), $allowedIps, true)) {
            return;
        }

        return service('response')
            ->setStatusCode(503)
            ->setHeader('Retry-After', '3600')
            ->setBody(view('errors/html/maintenance'));
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Toggle: `touch writable/maintenance.flag` (on) / `rm writable/maintenance.flag` (off).

## Zero-Downtime Deployment

### Strategy: Symlinked releases

```
/var/www/myapp/
├── current → releases/2026-06-01-120000/   # symlink
├── releases/
│   ├── 2026-05-30-090000/
│   ├── 2026-05-31-100000/
│   └── 2026-06-01-120000/
└── shared/
    ├── .env
    ├── writable/
    └── public/uploads/
```

Deploy script (simplified):

```bash
#!/bin/bash
set -e

RELEASE=$(date +%Y-%m-%d-%H%M%S)
BASE=/var/www/myapp
RELEASES=$BASE/releases
NEW=$RELEASES/$RELEASE
SHARED=$BASE/shared

# 1. Clone & install
git clone --depth=1 -b main https://repo.example.com/myapp $NEW
cd $NEW
composer install --no-dev --optimize-autoloader --classmap-authoritative

# 2. Link shared
ln -nfs $SHARED/.env       $NEW/.env
rm -rf $NEW/writable
ln -nfs $SHARED/writable   $NEW/writable
ln -nfs $SHARED/uploads    $NEW/public/uploads

# 3. Migrations
php $NEW/spark migrate --all

# 4. Cache config
php $NEW/spark optimize

# 5. Atomically swap symlink
ln -nfs $NEW $BASE/current

# 6. Reload PHP-FPM (or whatever runtime)
sudo systemctl reload php8.2-fpm

# 7. Cleanup old releases (keep 5)
ls -1t $RELEASES | tail -n +6 | xargs -I{} rm -rf $RELEASES/{}
```

### Strategy: Docker blue-green

Two containers (`app-blue` and `app-green`) behind a load balancer. Deploy to inactive one, run migrations, switch traffic, drain old.

## CI/CD Pipeline (GitHub Actions Example)

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: intl, mbstring, mysqli, pdo_mysql, gd, zip, opcache

      - name: Install dependencies
        run: composer install --no-dev --optimize-autoloader --classmap-authoritative

      - name: Run tests
        run: vendor/bin/phpunit
        env:
          CI_ENVIRONMENT: testing

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push myapp:${{ github.sha }}

      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.HOST }}
          username: deploy
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker pull myapp:${{ github.sha }}
            docker tag myapp:${{ github.sha }} myapp:latest
            cd /var/www/myapp
            docker compose up -d
            docker compose exec -T app php spark migrate --all
```

## Troubleshooting Production

### App returns 500 with no useful info

```bash
# Check logs
tail -f writable/logs/log-$(date +%Y-%m-%d).log

# Check PHP error log
tail -f /var/log/php/error.log

# Check web server logs
tail -f /var/log/nginx/error.log
tail -f /var/log/apache2/error.log
```

### Performance degradation

- Check OPcache hit rate: `opcache_get_status()['opcache_statistics']['opcache_hit_rate']` (should be >99%)
- Check slow query log on MySQL
- Check `pm.max_children` on FPM (might be saturated)
- Check Redis memory: `redis-cli info memory`

### "writable not writable" after deploy

```bash
chown -R www-data:www-data writable/
find writable -type d -exec chmod 775 {} \;
find writable -type f -exec chmod 664 {} \;
```

## Backup Strategy

- **Database:** Daily automated backups, retained 30 days; weekly off-site
- **Uploaded files:** Real-time sync to S3/object storage
- **Configuration:** Version-controlled (with secrets in vault)
- **Test restores quarterly** — backups you haven't restored aren't backups

## Best Practices

- **Never** edit code on production server — always deploy from git
- **Never** run `migrate:refresh` or `db:seed` in production (data loss)
- **Always** test migrations on a staging copy of production data first
- **Always** have a rollback procedure tested before you need it
- **Monitor everything** — uptime, errors, performance, logs
- **Automate everything** — humans make mistakes
- **Document runbooks** — for common ops tasks (deploy, rollback, scale)
- **Limit production access** — only specific people, audit logs
- **Practice incident response** — game days, runbook drills
