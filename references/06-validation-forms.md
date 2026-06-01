# 06 - Validation & Forms

Comprehensive guide to CodeIgniter 4 validation system.

## Where to Define Rules

Three places — pick based on context:

1. **In the Model** (`$validationRules`) — auto-applied on insert/update/save
2. **In a Config file** (`app/Config/Validation.php`) — reusable rule groups
3. **Inline in the Controller** — one-off cases

## Validation in Model (Recommended)

```php
class UserModel extends Model
{
    protected $validationRules = [
        'name'     => 'required|min_length[3]|max_length[255]',
        'email'    => 'required|valid_email|is_unique[users.email,id,{id}]',
        'password' => 'required|min_length[8]',
        'age'      => 'permit_empty|integer|greater_than[0]|less_than[150]',
    ];

    protected $validationMessages = [
        'email' => [
            'is_unique' => 'This email is already registered.',
        ],
        'password' => [
            'min_length' => 'Password must be at least 8 characters.',
        ],
    ];
}

// Usage
$model = model('UserModel');
if (! $model->insert($data)) {
    $errors = $model->errors();   // ['email' => '...', 'password' => '...']
}
```

## Validation Rule Groups (Config)

`app/Config/Validation.php`:

```php
<?php

namespace Config;

use CodeIgniter\Config\BaseConfig;
use CodeIgniter\Validation\StrictRules\CreditCardRules;
use CodeIgniter\Validation\StrictRules\FileRules;
use CodeIgniter\Validation\StrictRules\FormatRules;
use CodeIgniter\Validation\StrictRules\Rules;

class Validation extends BaseConfig
{
    public array $ruleSets = [
        Rules::class,
        FormatRules::class,
        FileRules::class,
        CreditCardRules::class,
    ];

    public array $templates = [
        'list'   => 'CodeIgniter\Validation\Views\list',
        'single' => 'CodeIgniter\Validation\Views\single',
    ];

    // Custom rule group for signup
    public array $signup = [
        'name'         => 'required|min_length[3]|max_length[255]',
        'email'        => 'required|valid_email|is_unique[users.email]',
        'password'     => 'required|min_length[8]',
        'pass_confirm' => 'required_with[password]|matches[password]',
    ];

    public array $signup_errors = [
        'name' => [
            'required'   => 'Please provide your name.',
            'min_length' => 'Name must be at least 3 characters.',
        ],
        'email' => [
            'is_unique' => 'This email is already registered.',
        ],
    ];

    // Re-usable in different controllers
    public array $login = [
        'email'    => 'required|valid_email',
        'password' => 'required',
    ];
}
```

Use the group:

```php
// In a controller
if (! $this->validate('signup')) {
    return view('auth/signup', [
        'errors' => $this->validator->getErrors(),
    ]);
}

// Or in a model
class UserModel extends Model
{
    protected $validationRules = 'signup';   // points to the group above
}
```

## Inline Validation in Controllers

```php
public function store()
{
    $rules = [
        'title'   => 'required|min_length[5]|max_length[255]',
        'content' => 'required|min_length[20]',
        'tags'    => 'permit_empty|max_length[500]',
    ];

    $messages = [
        'title' => [
            'required' => 'Title is mandatory.',
        ],
    ];

    if (! $this->validate($rules, $messages)) {
        return view('posts/create', [
            'errors' => $this->validator->getErrors(),
        ]);
    }

    // Get only the validated fields
    $data = $this->validator->getValidated();
    model('PostModel')->insert($data);

    return redirect()->to('/posts')->with('success', 'Post created');
}
```

## Validating Specific Data

```php
// validateData($data, $rules, $messages)
$data = ['email' => 'foo@bar.com'];

if (! $this->validateData($data, ['email' => 'required|valid_email'])) {
    return $this->fail($this->validator->getErrors());
}
```

## Built-in Rules Reference

### Required & presence

| Rule | Description |
|------|-------------|
| `required` | Field must be present and non-empty |
| `permit_empty` | Allows empty value, but if present must pass other rules |
| `required_with[other]` | Required when another field is present |
| `required_without[other]` | Required when another field is absent |

### Length

| Rule | Description |
|------|-------------|
| `min_length[n]` | At least n characters |
| `max_length[n]` | At most n characters |
| `exact_length[n]` | Exactly n characters |

### Format

| Rule | Description |
|------|-------------|
| `alpha` | Letters only (A-Z, a-z) |
| `alpha_space` | Letters + spaces |
| `alpha_dash` | Letters + numbers + dash + underscore |
| `alpha_numeric` | Letters + numbers |
| `alpha_numeric_space` | Letters + numbers + space |
| `alpha_numeric_punct` | Letters + numbers + common punctuation |
| `numeric` | Numeric (allows decimals/negatives) |
| `integer` | Integers only |
| `decimal` | Decimal numbers |
| `is_natural` | Whole numbers (≥0) |
| `is_natural_no_zero` | Whole numbers (>0) |

### Comparison

| Rule | Description |
|------|-------------|
| `matches[field]` | Same value as another field (e.g. password confirmation) |
| `differs[field]` | Different from another field |
| `greater_than[n]` | Value greater than n |
| `greater_than_equal_to[n]` | Value ≥ n |
| `less_than[n]` | Value less than n |
| `less_than_equal_to[n]` | Value ≤ n |
| `in_list[a,b,c]` | Value must be in list |
| `not_in_list[a,b,c]` | Value must not be in list |

### Patterns

| Rule | Description |
|------|-------------|
| `regex_match[/pattern/]` | Match a regex |
| `valid_email` | Single valid email |
| `valid_emails` | Comma-separated valid emails |
| `valid_url` | Valid URL |
| `valid_url_strict[https]` | URL must use given scheme |
| `valid_ip` | Valid IPv4/IPv6 |
| `valid_date` | Valid date |
| `valid_date[Y-m-d]` | Valid date with format |
| `valid_json` | Valid JSON string |
| `valid_cc_number[type]` | Credit card |

### Database

| Rule | Description |
|------|-------------|
| `is_unique[table.field]` | Value must not exist in table.field |
| `is_unique[table.field,id_field,id]` | Unique except for current id |
| `is_not_unique[table.field]` | Value must exist (foreign key check) |

### File Uploads

| Rule | Description |
|------|-------------|
| `uploaded[field]` | A file was uploaded |
| `is_image[field]` | Uploaded file is an image |
| `mime_in[field,type1,type2]` | MIME type in list |
| `ext_in[field,jpg,png]` | Extension in list |
| `max_size[field,KB]` | Max size in KB |
| `max_dims[field,width,height]` | Max image dimensions |
| `min_dims[field,width,height]` | Min image dimensions |
| `is_image[field]` | Must be an image |

## File Upload Validation Example

```php
public function upload()
{
    $rules = [
        'avatar' => [
            'label' => 'Profile Photo',
            'rules' => [
                'uploaded[avatar]',
                'is_image[avatar]',
                'mime_in[avatar,image/jpg,image/jpeg,image/png,image/webp]',
                'max_size[avatar,2048]',                  // 2 MB
                'max_dims[avatar,2000,2000]',
            ],
        ],
    ];

    if (! $this->validateData([], $rules)) {
        return view('users/edit', [
            'errors' => $this->validator->getErrors(),
        ]);
    }

    $file = $this->request->getFile('avatar');

    if ($file->isValid() && ! $file->hasMoved()) {
        $newName = $file->getRandomName();
        $file->move(WRITEPATH . 'uploads/avatars', $newName);

        // Save path to DB...
        return redirect()->to('/profile')->with('success', 'Avatar updated');
    }

    return redirect()->back()->with('error', 'Upload failed');
}
```

## Custom Validation Rules

### Method 1: Closure rule (inline)

```php
$rules = [
    'username' => [
        'rules' => [
            'required',
            static function ($value, $data, &$error) {
                if (str_contains($value, 'admin')) {
                    $error = 'Username cannot contain "admin".';
                    return false;
                }
                return true;
            },
        ],
    ],
];
```

### Method 2: Custom rule class

```php
<?php
// app/Validation/CustomRules.php

namespace App\Validation;

class CustomRules
{
    public function strong_password(string $str, ?string &$error = null): bool
    {
        if (strlen($str) < 12) {
            $error = 'Password must be at least 12 characters.';
            return false;
        }
        if (! preg_match('/[A-Z]/', $str)) {
            $error = 'Password must contain an uppercase letter.';
            return false;
        }
        if (! preg_match('/[0-9]/', $str)) {
            $error = 'Password must contain a number.';
            return false;
        }
        if (! preg_match('/[^A-Za-z0-9]/', $str)) {
            $error = 'Password must contain a special character.';
            return false;
        }
        return true;
    }

    public function business_email(string $str): bool
    {
        $banned = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'];
        $domain = substr(strrchr($str, '@'), 1);
        return ! in_array(strtolower($domain), $banned, true);
    }
}
```

Register in `app/Config/Validation.php`:

```php
public array $ruleSets = [
    \CodeIgniter\Validation\StrictRules\Rules::class,
    \CodeIgniter\Validation\StrictRules\FormatRules::class,
    \CodeIgniter\Validation\StrictRules\FileRules::class,
    \CodeIgniter\Validation\StrictRules\CreditCardRules::class,
    \App\Validation\CustomRules::class,
];
```

Use it:

```php
$rules = [
    'password' => 'required|strong_password',
    'email'    => 'required|valid_email|business_email',
];
```

## Showing Errors in Views

```php
<!-- Single field error -->
<input type="email" name="email" value="<?= old('email') ?>">
<?php if (session('errors.email')): ?>
    <span class="error"><?= esc(session('errors.email')) ?></span>
<?php endif; ?>

<!-- Or with the error helper -->
<?= service('validation')->getError('email') ?>

<!-- All errors as a list -->
<?= service('validation')->listErrors() ?>

<!-- Single error display -->
<?= service('validation')->showError('email') ?>
```

The `old()` helper repopulates form fields after validation failure.

## Forms with Form Helper

```php
<?php helper('form'); ?>

<?= form_open('users/store') ?>
    <?= csrf_field() ?>

    <label for="name">Name</label>
    <?= form_input(['name' => 'name', 'id' => 'name', 'value' => old('name')]) ?>

    <label for="email">Email</label>
    <?= form_input(['name' => 'email', 'type' => 'email', 'value' => old('email')]) ?>

    <label for="role">Role</label>
    <?= form_dropdown('role', [
        'user'  => 'User',
        'admin' => 'Admin',
    ], old('role')) ?>

    <?= form_submit('submit', 'Save') ?>
<?= form_close() ?>
```

## Best Practices

- **Define validation in models** for entity-level rules (used everywhere)
- **Use rule groups in `Validation.php`** for shared rules across controllers
- Use **`is_unique[table.field,id_field,{id}]`** when updating to ignore the current row
- **`getValidated()`** returns only fields that passed validation (safer than getPost())
- Always **escape errors** in views with `esc()`
- Use **`old()`** to preserve user input on validation failure
- Add **CSRF tokens** to all non-API forms with `csrf_field()`
- For file uploads, **validate before moving** the file
- For custom rules, use the **error reference parameter** for clearer messages
