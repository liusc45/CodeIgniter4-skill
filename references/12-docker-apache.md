# 12 - Docker: Apache + mod_php

Simplest Docker setup. Best for local development and legacy projects that rely on `.htaccess`.

## When to Use

- Local development (simplest setup)
- Legacy projects migrating from CI3 (preserves `.htaccess` rules)
- Hosts that only offer Apache (shared hosting, cPanel)
- Quick prototyping

## Pros & Cons

**Pros:**
- `.htaccess` works out of the box (no rewrite config needed)
- Simplest configuration (one container)
- Beginner-friendly

**Cons:**
- Slower than Nginx + PHP-FPM (no separation)
- One process per request (mod_php)
- Higher memory footprint

## Project Structure

```
project/
├── docker/
│   ├── apache/
│   │   ├── 000-default.conf
│   │   └── php.ini
│   └── entrypoint.sh
├── app/
├── public/
├── writable/
├── Dockerfile
├── docker-compose.yml
└── .env
```

## Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.6
FROM php:8.2-apache

# ----- System dependencies -----
RUN apt-get update && apt-get install -y --no-install-recommends \
        git \
        unzip \
        libicu-dev \
        libzip-dev \
        libonig-dev \
        libpng-dev \
        libjpeg-dev \
        libfreetype6-dev \
        libxml2-dev \
        default-mysql-client \
    && rm -rf /var/lib/apt/lists/*

# ----- PHP extensions required by CI4 -----
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        intl \
        mbstring \
        zip \
        gd \
        pdo_mysql \
        mysqli \
        opcache \
        xml \
        bcmath

# ----- Composer -----
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# ----- Apache modules -----
RUN a2enmod rewrite headers expires deflate

# ----- Apache config -----
COPY docker/apache/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY docker/apache/php.ini          /usr/local/etc/php/conf.d/zz-app.ini

# ----- App -----
WORKDIR /var/www/html

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-scripts

COPY . .

# ----- Permissions -----
RUN chown -R www-data:www-data /var/www/html \
    && find writable -type d -exec chmod 775 {} \; \
    && find writable -type f -exec chmod 664 {} \;

EXPOSE 80
CMD ["apache2-foreground"]
```

## Apache Config (`docker/apache/000-default.conf`)

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted

        # Security headers
        Header set X-Content-Type-Options "nosniff"
        Header set X-Frame-Options "SAMEORIGIN"
        Header set Referrer-Policy "strict-origin-when-cross-origin"
    </Directory>

    # Block access to sensitive paths
    <DirectoryMatch "^/var/www/html/(app|system|writable|tests|vendor)">
        Require all denied
    </DirectoryMatch>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## PHP Config (`docker/apache/php.ini`)

```ini
; ----- Time & memory -----
date.timezone = "UTC"
memory_limit = 256M
max_execution_time = 60
max_input_time = 60

; ----- Uploads -----
upload_max_filesize = 32M
post_max_size = 32M
max_file_uploads = 20

; ----- OPcache (production) -----
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 1   ; 0 in pure production for max speed
opcache.revalidate_freq = 2

; ----- Realpath cache -----
realpath_cache_size = 4M
realpath_cache_ttl = 600

; ----- Sessions -----
session.cookie_httponly = 1
session.cookie_secure = 1   ; only via HTTPS
session.cookie_samesite = "Lax"

; ----- Errors (production) -----
display_errors = Off
display_startup_errors = Off
log_errors = On
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
```

## docker-compose.yml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ci4_app
    ports:
      - "8080:80"
    volumes:
      - ./app:/var/www/html/app:cached
      - ./public:/var/www/html/public:cached
      - ./writable:/var/www/html/writable
      - ./tests:/var/www/html/tests:cached
      - ./.env:/var/www/html/.env
    environment:
      CI_ENVIRONMENT: development
      database.default.hostname: db
      database.default.database: ci4_app
      database.default.username: ci4_user
      database.default.password: secret
      database.default.DBDriver: MySQLi
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
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-prootsecret"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [ci4_net]

  phpmyadmin:
    image: phpmyadmin:5
    container_name: ci4_pma
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      PMA_USER: ci4_user
      PMA_PASSWORD: secret
    depends_on:
      - db
    networks: [ci4_net]

  redis:
    image: redis:7-alpine
    container_name: ci4_redis
    ports:
      - "6379:6379"
    networks: [ci4_net]

volumes:
  db_data:

networks:
  ci4_net:
    driver: bridge
```

## .dockerignore

```
.git
.github
.idea
node_modules
vendor
writable/cache/*
writable/logs/*
writable/session/*
writable/uploads/*
.env
.env.testing
*.md
docker-compose.override.yml
tests
phpunit.xml.dist
```

## Workflow

```bash
# Build & start
docker compose up -d --build

# Composer commands
docker compose exec app composer install
docker compose exec app composer require some/package

# Spark CLI
docker compose exec app php spark serve --host=0.0.0.0
docker compose exec app php spark migrate
docker compose exec app php spark db:seed UserSeeder
docker compose exec app php spark routes

# Logs
docker compose logs -f app
docker compose logs -f db

# Shell
docker compose exec app bash

# Stop
docker compose down

# Stop + delete volumes (DANGER: data loss)
docker compose down -v
```

## Production Notes

For production builds, use multi-stage to keep images small:

```dockerfile
# ----- Stage 1: build -----
FROM composer:2 AS builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-scripts
COPY . .
RUN composer dump-autoload --classmap-authoritative

# ----- Stage 2: runtime -----
FROM php:8.2-apache
RUN apt-get update && apt-get install -y --no-install-recommends \
        libicu-dev libzip-dev \
    && docker-php-ext-install intl mbstring zip pdo_mysql opcache \
    && a2enmod rewrite headers \
    && rm -rf /var/lib/apt/lists/*
COPY docker/apache/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY docker/apache/php.ini          /usr/local/etc/php/conf.d/zz-app.ini
WORKDIR /var/www/html
COPY --from=builder --chown=www-data:www-data /app /var/www/html
EXPOSE 80
CMD ["apache2-foreground"]
```
