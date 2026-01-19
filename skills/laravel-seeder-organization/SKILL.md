---
name: laravel-seeder-organization
description: Document how to group Laravel seeders between local and shared environments, ensure seeders guard against duplication, and keep reruns safe.
license: MIT
compatibility: opencode
metadata:
  audience: developers
---

## Overview
Seeders in this repo should respect an environment split: shared seeders that apply everywhere and local-only helpers that run when `app()->environment('local')` is true. `DatabaseSeeder` already models that flow, so a skill can walk through adding new seeders following the same guardrails.

## Directory Layout
- `database/seeders/` – entry point `DatabaseSeeder.php` plus environment-neutral seeders.
- `database/seeders/Dummy/` – local-only helpers (e.g., `LocalUserSeeder`, `HistoricalInvoiceSeeder`) that require development data.

## Implementation Steps
1. **Seed shared data**: write a seeder in `database/seeders` and register it in `DatabaseSeeder::run`. Use `firstOrCreate`, `updateOrCreate`, or `upsert` so repeated runs reconcile data without duplication.
2. **Keep local helpers separate**: place dev-only seeders in a `Dummy` subnamespace (or equivalent) and invoke them inside `if (app()->environment('local'))` so they never hit staging/production.
3. **Structure DatabaseSeeder clearly**: call shared seeders in one block, then wrap the local-only calls in the environment guard, avoiding duplicate registrations (e.g., do not list the same seeder twice).

## Examples
**DatabaseSeeder layout**
```php
namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            SharedUserSeeder::class,
            PolicyStatusTableSeeder::class,
        ]);

        if (app()->environment('local')) {
            $this->call([
                Dummy\ProductSeeder::class,
            ]);
        }
    }
}
```

**Shared seeder (idempotent users)**
```php
namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class SharedUserSeeder extends Seeder
{
    public function run(): void
    {
        collect([
            ['email' => 'dev@mugonat.com', 'name' => 'Dev Lead', 'role' => 'owner'],
            ['email' => 'ops@mugonat.com', 'name' => 'Ops Specialist', 'role' => 'ops'],
        ])->each(fn (array $payload) =>
            User::updateOrCreate(
                ['email' => $payload['email']],
                [
                    'name' => $payload['name'],
                    'password' => Hash::make('secure-password'),
                    'role' => $payload['role'],
                ]
            )
        );
    }
}
```

**Local-only seeder (products)**
```php
namespace Database\Seeders\Dummy;

use Illuminate\Database\Seeder;
use App\Models\Product;

class ProductSeeder extends Seeder
{
    public function run(): void
    {
        collect([
            ['sku' => 'loc-prod-001', 'name' => 'Demo Policy Product', 'price_cents' => 15000],
            ['sku' => 'loc-prod-002', 'name' => 'Practice Claim Product', 'price_cents' => 9000],
        ])->each(fn (array $payload) =>
            Product::updateOrCreate(['sku' => $payload['sku']], $payload)
        );
    }
}
```
