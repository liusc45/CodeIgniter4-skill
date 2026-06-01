# 13 - Docker: Nginx + PHP-FPM

The standard production stack for CodeIgniter 4. Documented officially in CI4's running guide.

## When to Use

- Production deployments (most popular choice)
- High-traffic apps (best performance with OPcache)
- Microservices / API-first architectures
- When you want clear separation: web server vs PHP runtime

## Pros & Cons

**Pros:**
- High performance (Nginx static, FPM dynamic)
- Officially documented by CI4
- Mature, well-known stack
- Easy to scale FPM workers independently

**Cons:**
- Two containers to manage (nginx + php-fpm)
- More config files (nginx.conf + fpm pool)
- No `.htaccess` — rules in `nginx.conf` only

## Project Structure

```
project/
├── docker/
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       ├── Dockerfile
│       ├── php.ini
│       └── www.conf       # FPM pool config
├── app/
├── public/
├── writable/
├── docker-compose.yml
└── .env
```

## PHP-FPM Dockerfile (`docker/php/Dockerfile`)

```dockerfile
# syntax=docker/dockerfile:1.6
FROM php:8.2-fpm

# System deps
RUN apt-get update && apt-get install -y --no-install-recommends \
        git unzip \
        libicu-dev libzip-dev libonig-dev \
        libpng-dev libjpeg-dev libfreetype6-dev \
        libxml2-dev default-mysql-client \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        intl mbstring zip gd \
        pdo_mysql mysqli \
        opcache xml bcmath

# Redis extension (optional but common)
RUN pecl install redis && docker-php-ext-enable redis

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Configs
COPY php.ini  /usr/local/etc/php/conf.d/zz-app.ini
COPY www.conf /usr/local/etc/php-fpm.d/zz-www.conf

WORKDIR /var/www/html

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-scripts

COPY . .

RUN chown -R www-data:www-data /var/www/html \
    && find writable -type d -exec chmod 775 {} \; \
    && find writable -type f -exec chmod 664 {} \;

EXPOSE 9000
CMD ["php-fpm"]
```

## PHP-FPM Pool Config (`docker/php/www.conf`)

```ini
[www]
user = www-data
group = www-data

listen = 0.0.0.0:9000
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; ----- Process manager -----
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 10
pm.max_requests = 500

; ----- Status -----
pm.status_path = /fpm-status
ping.path = /fpm-ping

; ----- Logging -----
catch_workers_output = yes
decorate_workers_output = no
access.log = /proc/self/fd/2
slowlog = /proc/self/fd/2
request_slowlog_timeout = 5s

; ----- Limits -----
request_terminate_timeout = 60s

; ----- Env vars from container -----
clear_env = no
```

Tune `pm.max_children` based on RAM:
> `pm.max_children = (Available RAM - other services) / Avg PHP memory per request`
>
> Example: 512MB available, ~30MB per request → ~16 workers

## Nginx Config (`docker/nginx/default.conf`)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name localhost;

    root  /var/www/html/public;
    index index.php index.html;

    client_max_body_size 32M;

    # Security headers
    add_header X-Content-Type-Options "nosniff"          always;
    add_header X-Frame-Options        "SAMEORIGIN"        always;
    add_header Referrer-Policy        "strict-origin-when-cross-origin" always;
    add_header X-XSS-Protection       "1; mode=block"     always;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/javascript application/json application/javascript application/xml+rss image/svg+xml;

    # ----- Static assets with long cache -----
    location ~* \.(?:css|js|woff2?|ttf|eot|otf|svg|png|jpg|jpeg|gif|ico|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # ----- Main rewrite (CI4 routes) -----
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # ----- PHP handling -----
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;                    # Service name from compose
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO       $fastcgi_path_info;
        fastcgi_read_timeout 60s;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # ----- 404 → CI4's handler -----
    error_page 404 /index.php;

    # ----- Block hidden files -----
    location ~ /\.(?!well-known).* {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

## docker-compose.yml

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    container_name: ci4_nginx
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html/public:ro
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
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
      - ./tests:/var/www/html/tests:cached
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
  db_data:

networks:
  ci4_net:
    driver: bridge
```

## Workflow

```bash
# Build & up
docker compose up -d --build

# Spark CLI inside the PHP container
docker compose exec app php spark migrate
docker compose exec app php spark routes
docker compose exec app composer require codeigniter4/shield

# Tail logs
docker compose logs -f nginx
docker compose logs -f app

# Reload nginx after config change
docker compose exec nginx nginx -s reload
```

## Performance Tuning

### OPcache (php.ini for production)

```ini
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0   ; 0 = max speed (rebuild image to update)
opcache.save_comments = 1         ; required by some annotations/attributes
opcache.fast_shutdown = 1
opcache.preload = /var/www/html/preload.php   ; PHP 7.4+ preloading
opcache.preload_user = www-data
```

### FPM scaling

For high traffic:

```ini
pm = static                ; pre-fork all workers (max stability)
pm.max_children = 100      ; size based on RAM and avg request memory
pm.max_requests = 1000     ; restart workers periodically (memory leaks)
```

### Nginx cache (optional micro-cache)

```nginx
fastcgi_cache_path /tmp/nginx_cache levels=1:2 keys_zone=ci4_cache:50m max_size=500m inactive=10m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";

location ~ \.php$ {
    fastcgi_cache ci4_cache;
    fastcgi_cache_valid 200 5m;
    fastcgi_cache_methods GET HEAD;
    fastcgi_cache_bypass $arg_nocache;
    add_header X-Cache-Status $upstream_cache_status;
    # ... rest of fastcgi config
}
```

## SSL/TLS in Production

Use a reverse proxy (Traefik, Caddy, or AWS ALB) terminating TLS, OR add Certbot:

```yaml
nginx:
  image: nginx:1.27-alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./certs:/etc/ssl/certs:ro
    - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
```

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    # ... rest of config
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```
