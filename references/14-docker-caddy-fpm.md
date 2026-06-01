# 14 - Docker: Caddy + PHP-FPM

Caddy v2 is a modern web server with **automatic HTTPS**, simpler config, and HTTP/2 + HTTP/3 out of the box. Excellent choice for new CI4 projects.

## When to Use

- Production with **automatic HTTPS** (Let's Encrypt) — zero-config TLS
- Teams that prefer **simple, declarative config** (vs Nginx)
- Need **HTTP/2 and HTTP/3** by default
- Internal services with **automatic local certs**

## Pros & Cons

**Pros:**
- **Auto HTTPS** with ZeroSSL/Let's Encrypt (no Certbot needed)
- **Single Caddyfile** much simpler than nginx.conf
- HTTP/2 and HTTP/3 (QUIC) by default
- On-demand TLS for multi-tenant apps
- Sane defaults (security headers, gzip)

**Cons:**
- Smaller community than Nginx (still active and growing)
- Newer, less Stack Overflow content
- Slight perf gap vs tuned Nginx (minor)

## Project Structure

```
project/
├── docker/
│   ├── caddy/
│   │   └── Caddyfile
│   └── php/
│       ├── Dockerfile
│       ├── php.ini
│       └── www.conf
├── app/
├── public/
├── writable/
├── docker-compose.yml
└── .env
```

## Caddyfile (`docker/caddy/Caddyfile`)

### Development (HTTP)

```caddyfile
{
    # Global options
    auto_https off
    admin off
}

:80 {
    root * /var/www/html/public
    encode zstd gzip

    # Security headers
    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options        "SAMEORIGIN"
        Referrer-Policy        "strict-origin-when-cross-origin"
        -Server
    }

    # Static asset caching
    @static {
        path *.css *.js *.woff2 *.ttf *.png *.jpg *.jpeg *.gif *.svg *.ico *.webp
    }
    header @static Cache-Control "public, max-age=2592000, immutable"

    # PHP-FPM
    php_fastcgi app:9000 {
        try_files {path} /index.php?{query}
    }

    # CI4 routing fallback
    try_files {path} /index.php?{query}

    file_server

    # Block hidden files
    @hidden {
        path_regexp ^/\.(?!well-known)
    }
    respond @hidden 403

    # Logs
    log {
        output stdout
        format console
        level INFO
    }
}
```

### Production (Auto HTTPS)

```caddyfile
{
    email admin@example.com
    # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory  ; uncomment for testing
}

example.com, www.example.com {
    redir https://example.com{uri} permanent

    root * /var/www/html/public
    encode zstd gzip

    header {
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Content-Type-Options    "nosniff"
        X-Frame-Options           "SAMEORIGIN"
        Referrer-Policy           "strict-origin-when-cross-origin"
        Permissions-Policy        "camera=(), microphone=(), geolocation=()"
        -Server
    }

    @static {
        path *.css *.js *.woff2 *.ttf *.png *.jpg *.jpeg *.gif *.svg *.ico *.webp
    }
    header @static Cache-Control "public, max-age=31536000, immutable"

    php_fastcgi app:9000 {
        try_files {path} /index.php?{query}

        # Health check
        env CI_ENVIRONMENT production
    }

    try_files {path} /index.php?{query}
    file_server

    @hidden { path_regexp ^/\.(?!well-known) }
    respond @hidden 403

    log {
        output file /var/log/caddy/access.log {
            roll_size 50mb
            roll_keep 10
        }
        format json
        level INFO
    }
}

# API subdomain example
api.example.com {
    root * /var/www/html/public

    php_fastcgi app:9000 {
        try_files {path} /index.php?{query}
    }

    try_files {path} /index.php?{query}
    file_server

    log {
        output file /var/log/caddy/api.log
        format json
    }
}
```

## docker-compose.yml

```yaml
services:
  caddy:
    image: caddy:2-alpine
    container_name: ci4_caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3 (QUIC)
    volumes:
      - ./docker/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./public:/var/www/html/public:ro
      - caddy_data:/data
      - caddy_config:/config
      - caddy_logs:/var/log/caddy
    depends_on:
      - app
    networks: [ci4_net]

  app:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    container_name: ci4_app
    volumes:
      - ./app:/var/www/html/app:cached
      - ./public:/var/www/html/public:cached
      - ./writable:/var/www/html/writable
      - ./.env:/var/www/html/.env
    environment:
      CI_ENVIRONMENT: production
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
  caddy_logs:
  db_data:

networks:
  ci4_net:
    driver: bridge
```

## PHP-FPM Dockerfile

(Same as `references/13-docker-nginx-fpm.md` — see that file.)

## Caddy Cheat Sheet

```bash
# Validate config
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile

# Reload config without downtime
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile

# View Caddy admin API (port 2019, internal only)
docker compose exec caddy curl http://localhost:2019/config/

# View logs
docker compose logs -f caddy
```

## Multi-Domain / Multi-App

Easy with Caddy — just add more site blocks:

```caddyfile
admin.example.com {
    root * /var/www/admin/public
    php_fastcgi admin-app:9000
    try_files {path} /index.php?{query}
    file_server
}

shop.example.com {
    root * /var/www/shop/public
    php_fastcgi shop-app:9000
    try_files {path} /index.php?{query}
    file_server
}
```

## On-Demand TLS (Multi-Tenant)

For SaaS apps with custom domains per tenant:

```caddyfile
{
    on_demand_tls {
        ask https://example.com/api/check-domain
        interval 2m
        burst 5
    }
}

:443 {
    tls {
        on_demand
    }
    root * /var/www/html/public
    php_fastcgi app:9000
    try_files {path} /index.php?{query}
    file_server
}
```

The `ask` endpoint should return 200 if the domain is allowed, 4xx otherwise. Implement in CI4:

```php
// app/Controllers/Api/DomainCheck.php
public function check(): ResponseInterface
{
    $domain = $this->request->getGet('domain');
    $tenant = model('TenantModel')->where('domain', $domain)->first();

    return $tenant
        ? $this->respond(['ok' => true], 200)
        : $this->failNotFound();
}
```

## Workflow

```bash
# Build & start
docker compose up -d --build

# Get a local cert (Caddy generates self-signed for localhost)
# Visit https://localhost — accept self-signed cert in browser

# Migrations
docker compose exec app php spark migrate

# Reload Caddy after Caddyfile changes (no downtime)
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile

# Logs
docker compose logs -f caddy
```

## Notes

- **First start with auto HTTPS:** Caddy contacts Let's Encrypt, this takes ~30s
- **Behind a load balancer:** Use `auto_https off` and let the LB handle TLS
- **Local dev with HTTPS:** Caddy auto-generates a self-signed cert for `.localhost`
- **DNS challenges:** For wildcard certs, configure `dns` provider plugin
