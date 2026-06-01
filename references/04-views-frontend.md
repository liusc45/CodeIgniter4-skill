# 04 - Views & Frontend Integration

How CI4 renders HTML/CSS/JavaScript, view layouts, view cells, and integrates with frontend assets.

## Basic View Rendering

```php
// In a controller
return view('home');                          // app/Views/home.php
return view('users/profile', ['user' => $u]); // app/Views/users/profile.php

// Multiple views (header + body + footer)
return view('templates/header', $data)
     . view('users/profile', $data)
     . view('templates/footer', $data);

// View as string (no echo)
$html = view('emails/welcome', ['name' => 'Jane']);
mail('jane@example.com', 'Welcome', $html);

// Set HTTP status
return $this->response->setStatusCode(404)
                      ->setBody(view('errors/404'));
```

## Inside a View File

```php
<!-- app/Views/users/profile.php -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title><?= esc($user['name']) ?> — Profile</title>
    <link rel="stylesheet" href="<?= base_url('assets/css/app.css') ?>">
</head>
<body>
    <h1>Hello, <?= esc($user['name']) ?>!</h1>

    <?php if ($user['is_admin']): ?>
        <p>You have admin privileges.</p>
    <?php endif; ?>

    <ul>
        <?php foreach ($user['roles'] as $role): ?>
            <li><?= esc($role) ?></li>
        <?php endforeach; ?>
    </ul>

    <script src="<?= base_url('assets/js/app.js') ?>"></script>
</body>
</html>
```

**Critical security rule:** ALWAYS use `esc()` to output any data. Without it you have an XSS vulnerability.

```php
<?= esc($user['bio']) ?>                  // HTML context (default)
<?= esc($url, 'url') ?>                   // URL context
<?= esc($value, 'attr') ?>                // HTML attribute context
<?= esc($json, 'js') ?>                   // JavaScript context
<?= esc($css, 'css') ?>                   // CSS context
```

## View Layouts (Template Inheritance)

The recommended way to share structure across pages.

### 1. Define a layout

```php
<!-- app/Views/layouts/default.php -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= $this->renderSection('title') ?> — My App</title>

    <link rel="stylesheet" href="<?= base_url('assets/css/app.css') ?>">
    <?= $this->renderSection('styles') ?>
</head>
<body>
    <header>
        <nav>
            <a href="<?= route_to('home') ?>">Home</a>
            <a href="<?= route_to('about') ?>">About</a>
        </nav>
    </header>

    <main>
        <?= $this->renderSection('content') ?>
    </main>

    <footer>
        <p>&copy; <?= date('Y') ?> My App</p>
    </footer>

    <script src="<?= base_url('assets/js/app.js') ?>"></script>
    <?= $this->renderSection('scripts') ?>
</body>
</html>
```

### 2. Extend the layout in a page view

```php
<!-- app/Views/users/profile.php -->
<?= $this->extend('layouts/default') ?>

<?= $this->section('title') ?>
    <?= esc($user['name']) ?>'s Profile
<?= $this->endSection() ?>

<?= $this->section('styles') ?>
    <link rel="stylesheet" href="<?= base_url('assets/css/profile.css') ?>">
<?= $this->endSection() ?>

<?= $this->section('content') ?>
    <h1><?= esc($user['name']) ?></h1>
    <p><?= esc($user['bio']) ?></p>

    <?= $this->include('users/_avatar') ?>
<?= $this->endSection() ?>

<?= $this->section('scripts') ?>
    <script>
        const userId = <?= esc($user['id'], 'js') ?>;
    </script>
    <script src="<?= base_url('assets/js/profile.js') ?>"></script>
<?= $this->endSection() ?>
```

### 3. View partials with `include()`

```php
<!-- app/Views/users/_avatar.php (a partial, no extends) -->
<div class="avatar">
    <img src="<?= esc($user['avatar_url'] ?? '/assets/img/default.png') ?>"
         alt="<?= esc($user['name']) ?>">
</div>
```

Partials are reusable view files that don't extend any layout.

## View Cells (Reusable Components)

View Cells are PHP classes that produce HTML, similar to Vue/React components but server-rendered.

### Generate a cell

```bash
php spark make:cell ProductCardCell
```

### Cell class

```php
<?php
// app/Cells/ProductCardCell.php

namespace App\Cells;

use CodeIgniter\View\Cells\Cell;

class ProductCardCell extends Cell
{
    public int    $id;
    public string $name;
    public float  $price;
    public string $imageUrl = '/assets/img/placeholder.png';

    public function discount(): string
    {
        return $this->price > 100 ? '10%' : '0%';
    }

    public function formattedPrice(): string
    {
        return '$' . number_format($this->price, 2);
    }
}
```

### Cell view

```php
<!-- app/Cells/product_card_cell.php (auto-discovered) -->
<article class="product-card">
    <img src="<?= esc($imageUrl) ?>" alt="<?= esc($name) ?>">
    <h3><?= esc($name) ?></h3>
    <p class="price"><?= esc($formattedPrice()) ?></p>
    <?php if ($discount() !== '0%'): ?>
        <span class="badge"><?= esc($discount()) ?> OFF</span>
    <?php endif; ?>
</article>
```

### Use the cell in a view

```php
<?= view_cell(\App\Cells\ProductCardCell::class, [
    'id'       => $product['id'],
    'name'     => $product['name'],
    'price'    => $product['price'],
    'imageUrl' => $product['image_url'],
]) ?>

<!-- With cache (3600s) -->
<?= view_cell('App\Cells\ProductCardCell', $product, 3600, 'product_' . $product['id']) ?>
```

## Asset Management

### Standard asset structure

```
public/
└── assets/
    ├── css/
    │   ├── app.css
    │   └── vendor/
    ├── js/
    │   ├── app.js
    │   └── vendor/
    ├── img/
    └── fonts/
```

### Helpers for URLs

```php
<?= base_url() ?>                       // http://example.com/
<?= base_url('assets/css/app.css') ?>   // http://example.com/assets/css/app.css
<?= site_url('users/profile') ?>        // http://example.com/users/profile
<?= current_url() ?>                    // current page URL
<?= asset_url('app.css') ?>             // requires custom helper
```

### Cache busting

```php
<!-- Manually with file mtime -->
<link rel="stylesheet" href="<?= base_url('assets/css/app.css') ?>?v=<?= filemtime(FCPATH . 'assets/css/app.css') ?>">

<!-- Using a versioned manifest (Vite-style) -->
<?php $manifest = json_decode(file_get_contents(FCPATH . 'build/manifest.json'), true); ?>
<link rel="stylesheet" href="<?= base_url('build/' . $manifest['app.css']['file']) ?>">
```

## Frontend Bundlers

CI4 doesn't ship a bundler — you choose. Common setups:

### Vite (recommended for new projects)

`package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

`vite.config.js`:

```js
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    outDir: 'public/build',
    manifest: true,
    rollupOptions: {
      input: ['resources/js/app.js', 'resources/css/app.css'],
    },
  },
});
```

In dev: `npm run dev` (assets served from `http://localhost:5173/`).
In production: `npm run build` (writes to `public/build/`).

### Tailwind CSS

```bash
npm install -D tailwindcss @tailwindcss/cli
npx tailwindcss init
```

`tailwind.config.js`:

```js
module.exports = {
  content: [
    './app/Views/**/*.php',
    './app/Cells/**/*.php',
  ],
  theme: { extend: {} },
  plugins: [],
};
```

Build:

```bash
npx tailwindcss -i ./resources/css/app.css -o ./public/assets/css/app.css --watch
```

## Hybrid Mode: Server-Rendered + Sprinkled JavaScript

Perfect for legacy projects modernizing UX without going full SPA.

```php
<!-- app/Views/dashboard.php -->
<?= $this->extend('layouts/default') ?>
<?= $this->section('content') ?>

<div id="app">
    <h1>Dashboard</h1>
    <button id="refresh-btn">Refresh stats</button>
    <div id="stats">
        <!-- Server-rendered initial state -->
        <p>Total users: <?= esc($totalUsers) ?></p>
    </div>
</div>

<script>
document.getElementById('refresh-btn').addEventListener('click', async () => {
    const res = await fetch('<?= site_url('api/stats') ?>', {
        headers: { 'Accept': 'application/json' }
    });
    const data = await res.json();
    document.getElementById('stats').innerHTML = `<p>Total users: ${data.total}</p>`;
});
</script>

<?= $this->endSection() ?>
```

This pattern works great with **htmx**, **Alpine.js**, or **Stimulus** for incremental modernization.

## CSP (Content Security Policy)

Enable in `app/Config/App.php`:

```php
public bool $CSPEnabled = true;
```

Then in views, the framework automatically generates nonces:

```php
<script <?= csp_script_nonce() ?>>
    // Your inline script (will be allowed by CSP)
</script>

<style <?= csp_style_nonce() ?>>
    /* Inline styles allowed */
</style>
```

Configure CSP rules in `app/Config/ContentSecurityPolicy.php`.

## Best Practices

- **ALWAYS use `esc()`** — output escaping prevents XSS
- **Use layouts** for shared structure (`extend()`, `section()`)
- **Use partials (`include()`)** for repeated chunks
- **Use View Cells** for reusable, parameterized HTML components
- **Place compiled assets** in `public/assets/` or `public/build/`
- **Source files** (uncompiled JS/CSS/SCSS/TS) go in `resources/`
- **Cache busting** is required for CSS/JS in production
- **Enable CSP** for an extra XSS layer
- **For SPA frontends**, see `references/05-rest-api.md` instead
