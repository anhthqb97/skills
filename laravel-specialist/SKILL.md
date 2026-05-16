---
name: laravel-specialist
description: >-
  Expert Laravel 13 development with PHP 8.3+. Covers module-based architecture
  (nwidart/laravel-modules), Action/UseCase pattern, Repository pattern, Eloquent ORM,
  PHP Attributes, JSON:API resources, queue jobs, Sanctum auth, and bilingual i18n.
  Use when building Laravel 13 features, creating Eloquent models, designing API endpoints,
  implementing queue jobs, writing FormRequest validation, or structuring modular applications.
  Invoke for Laravel, Eloquent, Artisan, Blade, Livewire, Sanctum, Horizon, PHP 8.3, modules.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: Laravel, Eloquent, PHP 8.3, Artisan, Blade, Livewire, Sanctum, Horizon, laravel-modules, UseCase, Repository, FormRequest, Migration, Queue, Job
  role: specialist
  scope: implementation
  output-format: code
  related-skills: php-modernization, devops-engineer, test-master
---

# Laravel 13 Specialist

Expert Laravel 13 engineer. PHP 8.3 minimum. Attributes-first. No shortcuts.

## Core Architecture

```
Request → Middleware → Controller → Action (UseCase) → Repository → Model → DB
```

| Layer | Owns | Never |
|-------|------|-------|
| **Controller** | Validate (FormRequest), delegate to Action, wrap writes in `DB::transaction()`, return response | Business logic, direct DB queries |
| **Action** | Orchestrate business logic, call repositories, fire events | Touch `Request`, return HTTP responses |
| **Repository** | All Eloquent queries for one entity | Business rules, call other repositories |
| **Model** | Schema, relationships, scopes, casts | Business logic, service calls |
| **Service** | Cross-entity/cross-module capabilities | Own a specific entity's CRUD |

## Laravel 13 Key Features

- **PHP Attributes** on models, jobs, commands — prefer over class properties
- **`#[Fillable]`, `#[Hidden]`, `#[Table]`** on models
- **`#[Connection]`, `#[Queue]`, `#[Tries]`, `#[Timeout]`** on jobs
- **`#[Signature]`, `#[Description]`** on console commands
- **JSON:API Resources** — first-party, use for API responses
- **Laravel AI SDK** — `use Illuminate\AI\Facades\AI` for LLM integration
- **`Cache::touch()`** — extend TTL without re-storing
- **Queue routing by class** — `Queue::route(MyJob::class, 'redis')`
- **Vector/semantic search** — pgvector support via `whereNearestNeighbor()`

## Code Patterns

### Model (PHP 8.3 + Attributes)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Models;

use Illuminate\Database\Eloquent\Attributes\{Fillable, Hidden, ObservedBy, ScopedBy};
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;

#[Fillable(['name', 'code', 'status', 'location_id'])]
#[ObservedBy(AssetObserver::class)]
final class Asset extends Model
{
    use SoftDeletes;

    protected $table = 'asset_assets';

    protected function casts(): array
    {
        return ['status' => AssetStatusEnum::class];
    }

    public function location(): BelongsTo
    {
        return $this->belongsTo(Location::class);
    }

    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', AssetStatusEnum::ACTIVE);
    }
}
```

### Action (UseCase)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\UseCases\Asset;

use Modules\Inventory\App\Repositories\AssetRepository;

final class CreateAction
{
    public function __construct(
        private readonly AssetRepository $repository,
    ) {}

    public function __invoke(array $data): void
    {
        $this->repository->create($data);
    }
}
```

### Controller (thin)

```php
<?php declare(strict_types=1);

public function store(CreateRequest $request, CreateAction $action): JsonResponse
{
    try {
        DB::transaction(fn() => $action($request->validated()));
        return response()->json(['status' => 'success', 'message' => __('module::lang.created')]);
    } catch (\Throwable $th) {
        Log::error($th);
        return response()->json(['status' => 'error', 'message' => __('module::lang.action_failed')], 500);
    }
}
```

### Repository

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Repositories;

use Xalpha\Core\App\Repositories\BaseRepository;

final class AssetRepository extends BaseRepository
{
    public function __construct()
    {
        parent::__construct(new Asset());
    }

    public function listTable(array $filters): LengthAwarePaginator
    {
        return $this->model->query()
            ->with(['location'])
            ->when($filters['status'] ?? null, fn($q, $v) => $q->where('status', $v))
            ->when($filters['search'] ?? null, fn($q, $v) => $q->where('name', 'like', "%{$v}%"))
            ->paginate($filters['per_page'] ?? 15);
    }
}
```

### Enum (Backed + BaseEnum)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Enums\Models\Asset;

use Xalpha\Core\App\Enums\BaseEnum;

enum AssetStatusEnum: string
{
    use BaseEnum;

    case ACTIVE   = 'active';
    case INACTIVE = 'inactive';
    case DISPOSED = 'disposed';

    public function label(): string
    {
        return match($this) {
            self::ACTIVE   => __('inventory::lang.status.active'),
            self::INACTIVE => __('inventory::lang.status.inactive'),
            self::DISPOSED => __('inventory::lang.status.disposed'),
        };
    }

    public function color(): string
    {
        return match($this) {
            self::ACTIVE   => 'success',
            self::INACTIVE => 'secondary',
            self::DISPOSED => 'danger',
        };
    }
}
```

### FormRequest

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Requests\Asset;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class CreateRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'name'        => ['required', 'string', 'max:255'],
            'code'        => ['required', 'string', 'unique:asset_assets,code'],
            'status'      => ['required', Rule::enum(AssetStatusEnum::class)],
            'location_id' => ['required', 'exists:setting_locations,id'],
        ];
    }

    public function attributes(): array
    {
        return [
            'name'        => __('inventory::lang.fields.name'),
            'code'        => __('inventory::lang.fields.code'),
            'status'      => __('inventory::lang.fields.status'),
            'location_id' => __('inventory::lang.fields.location'),
        ];
    }
}
```

### Queue Job (PHP 8.3 Attributes)

```php
<?php declare(strict_types=1);

use Illuminate\Queue\Attributes\{Connection, Queue, Tries, Timeout};

#[Connection('redis')]
#[Queue('notifications')]
#[Tries(3)]
#[Timeout(60)]
final class SendAssetNotification implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly Asset $asset,
        private readonly string $event,
    ) {}

    public function handle(): void
    {
        // send notification
    }

    public function failed(\Throwable $e): void
    {
        Log::error('SendAssetNotification failed', [
            'asset' => $this->asset->id,
            'error' => $e->getMessage(),
        ]);
    }
}
```

## API Response Envelope

Always return:
```json
{ "status": "success|error", "message": "...", "data": {}, "count": N }
```

- Write ops: `{ "status": "success", "message": "..." }` — no `data`
- Read ops: `{ "status": "success", "data": [...] }`
- Errors: `{ "status": "error", "message": __('lang.action_failed') }` — never expose stack traces
- HTTP codes: `200` for all success (reads + writes), `422` validation, `403` auth, `500` server error

## Constraints

### MUST DO
- `declare(strict_types=1)` on every file
- PHP 8.3 Attributes on models and jobs
- Typed method parameters and return types on all public methods
- Backed enums for all status/type fields with `label()` and `color()`
- Explicit `$table` on all models (never rely on auto-naming)
- Explicit `$fillable` — `$guarded = []` is prohibited
- Eager load relationships — never trigger N+1
- `DB::transaction()` wrapping all write operations in controllers
- `try/catch(\Throwable)` + `Log::error()` on all write endpoints
- `$request->validated()` — never `$request->all()` on write paths
- Bilingual lang files (`lang/en/lang.php` + `lang/vi/lang.php`)
- `__()` for all user-facing strings — no hardcoded text
- Soft deletes (`SoftDeletes`) on all business entities

### MUST NOT DO
- Query in controller: `Model::where(...)->get()` inside controller method
- Business logic in model methods used in action flow
- Cross-module direct model import (use services/events/repositories)
- Raw SQL outside repositories
- `$guarded = []` on any model
- Hardcode config values
- Skip validation on user input
- Return exception messages or stack traces in API responses
- Use `new ActionClass()` — always inject via container

## Validation Checkpoints

| Stage | Command | Expected |
|-------|---------|----------|
| Migration | `php artisan migrate:status` | All `Ran` |
| Routes | `php artisan route:list --path=api` | New routes visible |
| Queue | `php artisan queue:work --once` | Job processes cleanly |
| Code style | `./vendor/bin/pint --test` | PSR-12 passes |

## Reference Guides

| Topic | File | Load When |
|-------|------|-----------|
| Eloquent deep-dive | `references/eloquent.md` | Complex queries, relationships, scopes |
| Module architecture | `references/architecture.md` | Creating new modules, cross-module patterns |
