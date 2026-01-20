---
name: audit-logging
description: Capture how this repo implements audit logging with owen-it/laravel-auditing, tapp/filament-auditing, and the custom Log model/traits so model changes and user actions are consistently recorded.
license: MIT
compatibility: opencode
metadata:
  audience: developers
---

## Overview
- Structured audits use `owen-it/laravel-auditing` (driver `database`, table `audits`) plus `tapp/filament-auditing` for UI. Audited events: `created`, `updated`, `deleted`, `restored`; timestamps are not logged; strict mode off; queue off; console auditing disabled.
- Every audited model also tracks `created_by_id`/`updated_by_id` via `App\Models\Traits\AuditsUser`, resolving the authenticated user on create/update and exposing `createdBy`/`updatedBy` relations.
- Ad-hoc activity logging uses `App\Models\Traits\HasLogs` (per-model helper) and `App\Models\Log` for action-level entries with IP, user agent, platform, browser, user type, and user ID (if authenticated). `App\Models\Traits\HasLogging` (Spatie `LogsActivity`) is included on most models to log all attributes by default.
- Avoid logging sensitive blobs/secrets by setting model-level `$auditExclude`/`$auditInclude` on OwenIt models and overriding `getActivitylogOptions()` when Spatie log scope needs to change.

## Implementing model audits (OwenIt)
1. Add to model: `use OwenIt\Auditing\Contracts\Auditable;` and `use \OwenIt\Auditing\Auditable;` then `implements Auditable` on the class. Keep `use AuditsUser` so actors are captured.
2. Ensure schema includes `created_by_id`/`updated_by_id` (see existing models using auto-migration stubs) and the shared `audits` table migration exists (`database/migrations/2025_07_01_163730_create_audits_table.php`).
3. If certain fields must be skipped (PII, large payloads), define `$auditExclude` or `$auditInclude` on the model. Arrays are not logged unless `allowed_array_values` is flipped on.
4. The package resolves IP, user agent, and URL via the default resolvers; leave them as-is unless you need a custom resolver.
5. Keep Filament audit views consistent: rely on `tapp/filament-auditing` defaults (`config/filament-auditing.php` uses lazy loading, sorted by `created_at desc`).

### Example model (audits + manual action logs)
```php
use App\Models\Traits\AuditsUser;
use App\Models\Traits\HasLogs;
use OwenIt\Auditing\Auditable;
use OwenIt\Auditing\Contracts\Auditable as AuditableContract;

class Invoice extends Model implements AuditableContract
{
    use AuditsUser, HasLogs;
    use Auditable; // OwenIt

    protected $guarded = [];
    protected $auditExclude = ['temporary_notes'];

    public function markPaid(): void
    {
        $this->update(['status' => 'paid']);
        $this->log('Invoice marked as paid');
    }
}
```

## Implementing action logs (custom Log model)
1. Add `use App\Models\Traits\HasLogs;` to models that need manual action entries. Call `$model->log('action text')` to persist a log row with user context (IP, agent, platform, browser, human/bot) and the acting user when authenticated.
2. `App\Models\Log` stores entries via a morph (`loggable_type/id`) and prunes old rows using `prunable()` (months defined by `config('documents.prune_after')`).
3. Use `HasLogging` (Spatie `LogsActivity`) when you want automatic attribute-level activity logs; override `getActivitylogOptions()` if you need narrower/expanded scope than `logAll()`.

## Displaying in Filament
- To show actors on tables, reuse `App\Filament\Components\Tables\AuditColumn::createdBy()` / `updatedBy()` for consistent labels/placeholders and email descriptions.
- For audit trails inside Filament resources, rely on the built-in audits relation manager provided by `tapp/filament-auditing` (honors the config sorting/lazy settings).

## Verification checklist
- Create/update/delete/restore a record and confirm an `audits` row exists with actor and context; ensure sensitive fields are excluded when required.
- Trigger `$model->log('...')` and confirm a `logs` row captures IP/user agent/platform/browser and the user (or null when unauthenticated).
- In Filament, verify audit columns show “Created By”/“Updated By” with placeholders (`System`/`Not updated yet`) and emails in the description.
