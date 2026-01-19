---
name: laravel-model-migrations
description: Explain how this repo (and others) reason about defining migrations directly on models, keeping the commands aligned after changes.
license: MIT
compatibility:
  - opencode
  - claude
  - antigravity
  - copilot
metadata:
  audience: developers
---

## Overview
This repo relies on `laravel-auto/migrations` (v1.0.4, see `composer.json`) so we can keep most tables in sync by declaring their schema on the model itself. The package lives alongside the standard Laravel migration workflow: `php artisan migrate` still runs first, then `php artisan migrate:auto` compares the models’ `migration()` definitions against the database and applies the delta.

## Installation & Setup
1. Run `composer install` (already included in this repo and enforced via post-install scripts) to install `laravel-auto/migrations` alongside the other dependencies.
2. Confirm the package version in `composer.json` (`"laravel-auto/migrations": "^1.0.4"`), and ensure `laravel-auto/migrations` is registered via auto-discovery.
3. Copy `.env.example` to `.env`, configure MySQL credentials, and run `php artisan key:generate` before running migrations; the `migrate:auto` command expects a database connection.

## Usage in this repo
- Define each model’s schema in a static `migration(Blueprint $table)` method. For example, create tables, timestamps, and unique indexes directly inside the model class instead of writing a separate migration file.
- Add an optional `public $migrationOrder` property when a table must be created before another (pivot tables, child models, etc.). Models without the property default to order `0`.
- When you need a model and factory that already include the migration scaffolding, run `php artisan make:amodel MyModel` (use `--force` to replace existing files, `--no-static-migration` to skip the auto migration stub, or `--no-soft-delete`/`--no-factory` to adjust generated artifacts).
- Run `php artisan migrate:auto` after you’ve updated or added models to let the package compare the declared schema to the database. Use `-f`/`--fresh` to drop all tables first, `-s` to seed afterward, and `--force` when running in production.

## Example model definition
```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Eloquent\Model;

class Invoice extends Model
{
    public static function migration(Blueprint $table): void
    {
        $table->id();
        $table->foreignId('customer_id')->constrained()->cascadeOnDelete();
        $table->decimal('total_amount', 15, 2);
        $table->enum('status', ['draft','paid','cancelled'])->default('draft');
        $table->timestamps(3);
    }
}
```

Run the command after editing models:
```
php artisan migrate
php artisan migrate:auto
```

## Workflow tips
- Treat `migrate:auto` as a follow-up to the regular migration run: the normal migration files in `database/migrations/` execute first, then the model-based schemas run.
- If you ever need to customize the stub used by `php artisan make:amodel`, publish the package stubs with `php artisan vendor:publish --tag=laravel-automatic-migrations` and update `config/auto-migrate.php` to point `stub_path` at `resources/stubs/vendor/laravel-automatic-migrations`.
- Keep schema changes in the models synchronized with the project’s `laravel-auto` settings so the command can safely diff the schema.

## References
- [Package README](https://github.com/chareka-legacy/laravel-auto-migrations/blob/main/readme.md)
- `composer.json` (`laravel-auto/migrations` entry)
