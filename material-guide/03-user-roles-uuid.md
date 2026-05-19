# 03 — UUID + `is_admin` role on users

> **Commit:** `ccef43f` — *feat(users): add UUID + is_admin role + admin factory state*

## Why this step?

Two foundational decisions we make once and use everywhere:

1. **Hybrid UUID strategy** — auto-increment `id` for joins (fast), `uuid` (UUIDv7) for public URLs (no leaking row counts to the world). Users will be route-bound by uuid.
2. **`is_admin` boolean** — the simplest possible role system. Good enough for a demo / classroom. (Production would use `spatie/laravel-permission` — we list this as a follow-up at the end of the course.)

We also add an `admin()` state on `UserFactory` so seeders and tests can spin up an admin in one line: `User::factory()->admin()->create()`.

## What you'll build

- A new migration `add_role_columns_to_users_table` adding `uuid` and `is_admin` columns
- `User` model with `HasUuids` trait, route-binding by `uuid`, an `isAdmin()` helper, and a `student()` HasOne relation (forward-declared for the next steps)
- `UserFactory` with `ms_MY` faker locale and an `admin()` state

## Prerequisites

- Completed [02 — Breeze authentication](./02-breeze-authentication.md).

## Steps

### 1. Generate the migration

```bash
php artisan make:migration add_role_columns_to_users_table --table=users
```

### 2. Fill in the migration

Open the generated file under `database/migrations/` and replace the body with:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->uuid('uuid')->after('id')->nullable()->unique();
            $table->boolean('is_admin')->default(false)->after('password');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['uuid', 'is_admin']);
        });
    }
};
```

The `uuid` column is **nullable** because existing users (from Breeze tests) don't have one yet — the `HasUuids` trait fills it on `creating` for new rows.

### 3. Update the `User` model

Replace `app/Models/User.php` with:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Support\Str;

class User extends Authenticatable
{
    use HasUuids;
    use Notifiable;

    protected $fillable = [
        'name', 'email', 'password', 'is_admin',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];

    public function uniqueIds(): array
    {
        return ['uuid'];          // HasUuids fills this column, not `id`
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';            // {user} route bindings resolve by uuid
    }

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
            'is_admin' => 'boolean',
        ];
    }

    public function initials(): string
    {
        return Str::of($this->name)
            ->explode(' ')
            ->take(2)
            ->map(fn ($word) => Str::substr($word, 0, 1))
            ->implode('');
    }

    public function isAdmin(): bool
    {
        return $this->is_admin === true;
    }

    /**
     * Forward-declared — Student model arrives in step 04.
     * Each User may have exactly one Student record (their own registration).
     */
    public function student(): HasOne
    {
        return $this->hasOne(Student::class);
    }
}
```

### 4. Update `UserFactory`

Open `database/factories/UserFactory.php` and replace `definition()` and add an `admin()` state:

```php
public function definition(): array
{
    return [
        'name' => fake('ms_MY')->name(),
        'email' => fake()->unique()->safeEmail(),
        'email_verified_at' => now(),
        'password' => static::$password ??= Hash::make('password'),
        'remember_token' => Str::random(10),
        'is_admin' => false,
    ];
}

public function admin(): static
{
    return $this->state(fn () => ['is_admin' => true]);
}
```

### 5. Reset the database and apply

```bash
php artisan migrate:fresh
```

## Expected output

```bash
php artisan tinker
```

```php
>>> $u = User::factory()->admin()->create();
>>> $u->isAdmin();
=> true
>>> $u->uuid;
=> "01935a..."        // UUIDv7 string
>>> $u->getRouteKey();
=> "01935a..."        // Same — confirms route binds by uuid
```

`php artisan test` should still be green — we haven't touched any auth logic, just added columns and helpers.

## Commit your work

```bash
git add .
git commit -m "feat(users): add UUID + is_admin role + admin factory state"
```

## Common pitfalls

- **`Class "App\Models\Student" not found`** when running tests — that's expected; Student arrives in step 04. The `student()` relation isn't called yet so the missing class doesn't bite. If your IDE complains, ignore it for now.
- **`uuid` column is empty for existing rows** — that's because they were inserted before the migration. Run `php artisan migrate:fresh` to wipe and start clean.
- **`HasUuids` fills `id` instead of `uuid`** — you forgot the `uniqueIds()` method. Without it, `HasUuids` defaults to the primary key column.

## What's next

Users now know who's an admin. Time to model the actual domain: [04 — Graduation + Student models](./04-domain-models.md).
