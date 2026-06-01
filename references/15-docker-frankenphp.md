# 15 - Docker: FrankenPHP

FrankenPHP is a modern PHP application server built on Caddy with **embedded PHP**, **Worker Mode**, **HTTP/2 and HTTP/3** native support, and **automatic HTTPS**. The fastest way to run PHP today.

## When to Use

- New high-performance projects (max throughput)
- API-heavy workloads (Worker Mode keeps app in memory)
- Real-time apps (HTTP/3, early hints, server-sent events)
- Edge/microservice deployments (single binary, single process)

## Pros & Cons

**Pros:**
- **Worker Mode**: app stays in memory between requests (10x+ faster)
- **No PHP-FPM** — embedded PHP runtime
- **Auto HTTPS** like Caddy
- **HTTP/2 + HTTP/3** native
- **Single binary** — simpler deployment
- **Mercure** built-in (server-sent events)

**Cons:**
- Newer (some library compat caveats — code must be "stateless safe")
- Worker mode requires careful state management (no global state leaks)
- Smaller community than Nginx/FPM
- CI4 doesn't officially document FrankenPHP yet (works fine in practice)

## Project Structure

```
project/
├── docker/
│   └── frankenphp/
│       ├── Caddyfile
│       └── frankenphp-worker.php
├── app/
├── public/
├── writable/
├── Dockerfile
├── docker-compose.yml
└── .env
```

## Dockerfile (Standard Mode)

```dockerfile
# syntax=docker/dockerfile:1.6
FROM dunglas/frankenphp:latest-php8.2

# Install required PHP extensions
RUN install-php-extensions \
        intl \
        mbstring \
        zip \
        gd \
        pdo_mysql \
        mysqli \
        opcache \
        xml \
        bcmath \
        redis

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /app

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-scripts

COPY . .

RUN chown -R www-data:www-data /app \
    && find writable -type d -exec chmod 775 {} \; \
    && find writable -type f -exec chmod 664 {} \;

# Caddyfile lives at /etc/caddy/Caddyfile by default
COPY docker/frankenphp/Caddyfile /etc/caddy/Caddyfile

EXPOSE 80 443 443/udp
```

## Caddyfile for FrankenPHP

### Standard Mode

```caddyfile
{
    # Frankenphp directives
    frankenphp
    order php_server before file_server
}

:80 {
    root * /app/public
    encode zstd gzip

    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options        "SAMEORIGIN"
        Referrer-Policy        "strict-origin-when-cross-origin"
        -Server
    }

    @static {
        path *.css *.js *.woff2 *.ttf *.png *.jpg *.jpeg *.svg *.ico *.webp
    }
    header @static Cache-Control "public, max-age=2592000, immutable"

    # PHP server (handles all .php and CI4 routing)
    php_server

    @hidden { path_regexp ^/\.(?!well-known) }
    respond @hidden 403
}
```

### Production with Auto HTTPS

```caddyfile
{
    frankenphp
    order php_server before file_server
    email admin@example.com
}

example.com, www.example.com {
    root * /app/public
    encode zstd gzip

    header {
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Content-Type-Options    "nosniff"
        X-Frame-Options           "SAMEORIGIN"
    }

    @static {
        path *.css *.js *.woff2 *.ttf *.png *.jpg *.jpeg *.svg *.ico *.webp
    }
    header @static Cache-Control "public, max-age=31536000, immutable"

    php_server
}
```

## Worker Mode (Maximum Performance)

Worker Mode boots CI4 once and keeps the framework in memory between requests. This eliminates ~80% of bootstrap overhead.

### Worker Script (`docker/frankenphp/worker.php`)

```php
<?php

// Boot CI4 once
ignore_user_abort(true);

require __DIR__ . '/../../public/index.php';

// `php_server` will pass the request directly via $_SERVER, etc.
// CI4's CodeIgniter::run() is called per request inside the worker loop.

$handler = static function () {
    // CI4 bootstrap (similar to public/index.php)
    require __DIR__ . '/../../app/Config/Boot/' . ENVIRONMENT . '.php';
    $app = \Config\Services::codeigniter();
    $app->setContext('web');
    $app->initialize();
    return $app;
};

$app = $handler();

// Worker loop
while (\frankenphp_handle_request(static function () use ($app) {
    $app->run();
})) {
    // Cleanup between requests if needed
    gc_collect_cycles();
}
```

### Caddyfile with Worker

```caddyfile
{
    frankenphp {
        worker /app/docker/frankenphp/worker.php 4   # 4 worker processes
    }
    order php_server before file_server
}

:80 {
    root * /app/public
    encode zstd gzip

    php_server {
        try_files {path} /index.php
    }
}
```

> **CAVEAT:** Worker mode requires CI4 to be safe across requests:
>
> - Don't use static state in your code
> - Reset session/auth state between requests (CI4 does this in `run()`)
> - Some third-party libraries with global state may misbehave
> - Test thoroughly before going to production

## docker-compose.yml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ci4_frankenphp
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3
    volumes:
      - ./app:/app/app:cached
      - ./public:/app/public:cached
      - ./writable:/app/writable
      - ./.env:/app/.env
      - caddy_data:/data
      - caddy_config:/config
    environment:
      CI_ENVIRONMENT: production
      SERVER_NAME: ":80"             # ":443" for HTTPS, "example.com" for prod
      database.default.hostname: db
      database.default.database: ci4_app
      database.default.username: ci4_user
      database.default.password: secret
    depends_on:
      db:
        condition: service_healthy
    networks: [ci4_net]

  db:
    image: mysql:8.0
    container_name: ci4_db
    environment:
      MYSQL_ROOT_PASSWORD: rootsecret
      MYSQL_DATABASE: ci4_app
      MYSQL_USER: ci4_user
      MYSQL_PASSWORD: secret
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-prootsecret"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [ci4_net]

  redis:
    image: redis:7-alpine
    container_name: ci4_redis
    networks: [ci4_net]

volumes:
  caddy_data:
  caddy_config:
  db_data:

networks:
  ci4_net:
    driver: bridge
```

## Without Docker (Standalone Binary)

FrankenPHP can also run as a single binary:

```bash
# Download
curl -L -o frankenphp 'https://github.com/php/frankenphp/releases/latest/download/frankenphp-linux-x86_64'
chmod +x frankenphp

# Run with auto-PHP detection
./frankenphp run --config Caddyfile

# Or just php-server mode
./frankenphp php-server -r /path/to/public
```

## Performance Comparison (Synthetic)

For a typical CI4 "Hello World" controller:

| Stack | Requests/sec |
|-------|------------:|
| Apache + mod_php | ~600 |
| Nginx + PHP-FPM | ~1,800 |
| Caddy + PHP-FPM | ~1,800 |
| FrankenPHP (standard) | ~2,500 |
| FrankenPHP (Worker Mode) | ~12,000+ |

> Numbers vary by hardware/app. The relative ranking is consistent.

## Tips for CI4 + Worker Mode

- **Reset Services container** between requests if you mutate state:
  ```php
  while (frankenphp_handle_request(...)) {
      \Config\Services::reset();
      // ...
  }
  ```
- **Avoid `static` properties** that hold request data
- **Don't `exit()` or `die()`** — return responses normally
- **Profile memory** with `memory_get_usage()` to detect leaks
- **Set `pm.max_requests = 1000`** equivalent: restart workers periodically:
  ```caddyfile
  frankenphp {
      worker /app/worker.php 4 1000   # 4 workers, restart every 1000 requests
  }
  ```

## Server-Sent Events (Mercure)

FrankenPHP includes Mercure, a hub for real-time updates:

```caddyfile
{
    mercure {
        publisher_jwt !ChangeThisMercureHubJWTSecretKey!
        subscriber_jwt !ChangeThisMercureHubJWTSecretKey!
    }
}

example.com {
    route /.well-known/mercure* {
        mercure
    }

    php_server
}
```

In CI4:

```php
// Publish an update
$jwt = service('jwt')->generateForMercure(['*']);
$ch  = curl_init('http://localhost/.well-known/mercure');
curl_setopt_array($ch, [
    CURLOPT_POSTFIELDS => http_build_query([
        'topic' => 'order/123',
        'data'  => json_encode(['status' => 'shipped']),
    ]),
    CURLOPT_HTTPHEADER => ["Authorization: Bearer {$jwt}"],
    CURLOPT_RETURNTRANSFER => true,
]);
curl_exec($ch);
```

## Best Practices

- **Start in standard mode**, switch to worker mode after profiling
- **Test thoroughly** — worker mode bugs only show up under load
- **Use `SERVER_NAME` env var** to control HTTPS in production
- **Mount `caddy_data` volume** to persist Let's Encrypt certs
- **Open UDP 443** for HTTP/3 to actually work
- **Use Redis/file cache** instead of in-memory PHP arrays (worker safety)
