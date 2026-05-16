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
  version: "2.0.0"
  domain: backend
  triggers: Laravel, Eloquent, PHP 8.3, Artisan, Blade, Livewire, Sanctum, Horizon, laravel-modules, UseCase, Repository, FormRequest, Migration, Queue, Job
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-testing, laravel-api, laravel-security, laravel-performance, laravel-queue, laravel-modules, laravel-livewire, php-modernization
---

# Laravel 13 Specialist

Expert Laravel 13 engineer. PHP 8.3 minimum. Attributes-first. No shortcuts.

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Architecture & Structure | CRITICAL | `arch-` |
| 2 | Eloquent & Database | CRITICAL | `eloquent-` |
| 3 | Controllers & Routing | HIGH | `ctrl-` |
| 4 | Validation & Requests | HIGH | `valid-` |
| 5 | Security | HIGH | `sec-` |
| 6 | Performance | MEDIUM | `perf-` |
| 7 | API Design | MEDIUM | `api-` |

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
| **DTO** | Typed data transfer between layers | Business logic, DB access |

## Laravel 13 Key Features

- **PHP Attributes** on models, jobs, commands — prefer over class properties
- **`#[Fillable]`, `#[Hidden]`, `#[Table]`** on models
- **`#[Connection]`, `#[Queue]`, `#[Tries]`, `#[Timeout]`** on jobs
- **`#[Signature]`, `#[Description]`** on console commands
- **JSON:API Resources** — first-party, use for all API responses
- **Laravel AI SDK** — `use Illuminate\AI\Facades\AI` for LLM integration
- **`Cache::touch()`** — extend TTL without re-storing
- **Queue routing by class** — `Queue::route(MyJob::class, 'redis')` (`arch-queue-routing`)
- **Vector/semantic search** — pgvector support via `whereNearestNeighbor()` (`eloquent-vector-search`)
- **Model pruning** — `MassPrunable` / `Prunable` for automatic cleanup (`eloquent-pruning`)

## Code Patterns

### 1. Model (PHP 8.3 + Attributes) — `eloquent-`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Models;

use Illuminate\Database\Eloquent\Attributes\{Fillable, Hidden, ObservedBy};
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\MassPrunable;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Builder;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;

#[Fillable(['name', 'code', 'status', 'location_id'])]
#[ObservedBy(AssetObserver::class)]
final class Asset extends Model
{
    use SoftDeletes, MassPrunable;

    protected $table = 'asset_assets';

    protected function casts(): array
    {
        return [
            'status'     => AssetStatusEnum::class,
            'disposed_at' => 'datetime',
        ];
    }

    // Relationships
    public function location(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Location::class);
    }

    // Scopes — reusable query logic (`eloquent-query-scopes`)
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', AssetStatusEnum::ACTIVE);
    }

    public function scopeByLocation(Builder $query, int $locationId): Builder
    {
        return $query->where('location_id', $locationId);
    }

    // Pruning — auto-cleanup disposed assets older than 1 year (`eloquent-pruning`)
    public function prunable(): Builder
    {
        return static::where('status', AssetStatusEnum::DISPOSED)
            ->where('disposed_at', '<=', now()->subYear());
    }
}
```

### 2. DTO (Data Transfer Object) — `arch-dto-pattern`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\DTOs;

final readonly class CreateAssetDTO
{
    public function __construct(
        public string $name,
        public string $code,
        public string $status,
        public int    $locationId,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            name:       $data['name'],
            code:       $data['code'],
            status:     $data['status'],
            locationId: $data['location_id'],
        );
    }
}
```

### 3. Action (UseCase) — `arch-action-classes`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\UseCases\Asset;

use Modules\Inventory\App\DTOs\CreateAssetDTO;
use Modules\Inventory\App\Events\AssetCreated;
use Modules\Inventory\App\Models\Asset;
use Modules\Inventory\App\Repositories\AssetRepository;

final class CreateAction
{
    public function __construct(
        private readonly AssetRepository $repository,
    ) {}

    public function __invoke(CreateAssetDTO $dto): Asset
    {
        $asset = $this->repository->create([
            'name'        => $dto->name,
            'code'        => $dto->code,
            'status'      => $dto->status,
            'location_id' => $dto->locationId,
        ]);

        AssetCreated::dispatch($asset);   // arch-event-driven

        return $asset;
    }
}
```

### 4. Controller (thin) — `ctrl-`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Controllers\Api;

use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\{DB, Log};
use Modules\Inventory\App\DTOs\CreateAssetDTO;
use Modules\Inventory\App\Http\Requests\Asset\CreateRequest;
use Modules\Inventory\App\Http\Resources\AssetResource;
use Modules\Inventory\App\Repositories\AssetRepository;
use Modules\Inventory\App\UseCases\Asset\CreateAction;

final class AssetController
{
    public function __construct(
        private readonly AssetRepository $repository,   // ctrl-dependency-injection
    ) {}

    public function store(CreateRequest $request, CreateAction $action): JsonResponse
    {
        $this->authorize('create', Asset::class);       // sec-mass-assignment + Policy

        try {
            $asset = DB::transaction(
                fn() => $action(CreateAssetDTO::fromArray($request->validated()))
            );

            return response()->json([
                'status'  => 'success',
                'message' => __('inventory::lang.created'),
                'data'    => new AssetResource($asset),
            ]);
        } catch (\Throwable $th) {
            Log::error($th);
            return response()->json([
                'status'  => 'error',
                'message' => __('inventory::lang.action_failed'),
            ], 500);
        }
    }
}
```

### 5. Repository — `eloquent-eager-loading`, `perf-`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Repositories;

use Illuminate\Pagination\LengthAwarePaginator;
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
            ->select(['id', 'name', 'code', 'status', 'location_id', 'created_at'])
            ->with(['location:id,name'])                        // perf-eager-loading
            ->when($filters['status'] ?? null, fn($q, $v) => $q->where('status', $v))
            ->when($filters['search'] ?? null, fn($q, $v) => $q->where('name', 'like', "%{$v}%"))
            ->when($filters['location_id'] ?? null, fn($q, $v) => $q->byLocation($v))
            ->paginate($filters['per_page'] ?? 15);
    }
}
```

### 6. FormRequest — `valid-`

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Requests\Asset;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;

final class CreateRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Asset::class); // valid- + sec-
    }

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

    // valid-after-hooks — cross-field validation
    public function withValidator(\Illuminate\Validation\Validator $validator): void
    {
        $validator->after(function ($v) {
            if ($this->input('status') === AssetStatusEnum::DISPOSED->value
                && !$this->input('disposed_at')) {
                $v->errors()->add('disposed_at', __('inventory::lang.disposed_at_required'));
            }
        });
    }
}
```

### 7. Enum (Backed + label/color) — `eloquent-casts`

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

### 8. Queue Job (PHP 8.3 Attributes) — `arch-queue-routing`

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
        // idempotent — safe to retry
    }

    public function failed(\Throwable $e): void
    {
        Log::error('SendAssetNotification failed', [
            'asset_id' => $this->asset->id,
            'error'    => $e->getMessage(),
        ]);
    }
}

// arch-queue-routing — centralized routing in AppServiceProvider
Queue::route(SendAssetNotification::class, 'redis');
```

### 9. Migration Best Practices — `eloquent-`

```php
<?php declare(strict_types=1);

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_assets', function (Blueprint $table) {
            $table->id();
            $table->foreignId('location_id')
                  ->constrained('setting_locations')
                  ->cascadeOnDelete();
            $table->string('name');
            $table->string('code')->unique();
            $table->string('status', 20)->index();
            $table->timestamp('disposed_at')->nullable();
            $table->softDeletes();
            $table->timestamps();

            // Composite indexes for common filter combos
            $table->index(['status', 'location_id']);
            $table->index(['deleted_at', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_assets');
    }
};
```

## API Response Envelope — `api-`

Always return:
```json
{ "status": "success|error", "message": "...", "data": {}, "count": N }
```

| Operation | Shape |
|-----------|-------|
| Write | `{ "status": "success", "message": "..." }` |
| Read | `{ "status": "success", "data": [...], "count": N }` |
| Error | `{ "status": "error", "message": __('lang.action_failed') }` |
| HTTP | `200` success, `422` validation, `401` unauth, `403` forbidden, `500` server |

## Constraints

### MUST DO — `arch-` / `eloquent-` / `sec-` / `valid-`
- `declare(strict_types=1)` on every file
- PHP 8.3 Attributes on models (`#[Fillable]`) and jobs (`#[Connection]`, `#[Queue]`, etc.)
- Typed parameters and return types on all public methods
- Use DTOs between controller and action — never pass raw arrays across layers
- Backed enums for all status/type fields with `label()` and `color()`
- Explicit `$table` on all models
- `#[Fillable]` explicit allowlist — `$guarded = []` is prohibited (`sec-mass-assignment`)
- Eager load all relationships — never trigger N+1 (`eloquent-eager-loading`)
- `DB::transaction()` on all write operations in controllers
- `try/catch(\Throwable)` + `Log::error()` on all write endpoints
- `$request->validated()` — never `$request->all()` on write paths (`valid-form-requests`)
- `$this->authorize()` at the top of every controller method (`sec-`)
- Bilingual lang files (`lang/en/lang.php` + `lang/vi/lang.php`)
- `__()` for all user-facing strings — no hardcoded text
- `SoftDeletes` on all business entities (`eloquent-soft-deletes`)
- `withValidator()` for cross-field validation logic (`valid-after-hooks`)
- Fire domain events from Actions — not from controllers (`arch-event-driven`)
- Use `Queue::route()` in `AppServiceProvider` for centralized job routing (`arch-queue-routing`)

### MUST NOT DO
- Query in controller: `Model::where(...)->get()` inside controller method
- Business logic in model methods or observers
- Cross-module direct model import — use events/services
- Raw SQL outside repositories
- `$guarded = []` on any model
- Hardcode config values or user-facing strings
- Skip validation on any user input
- Return exception messages or stack traces in API responses
- `new ActionClass()` — always inject via container
- Pass `$request->all()` to actions — always `$request->validated()` then DTO

## Validation Checkpoints

| Stage | Command | Expected |
|-------|---------|----------|
| Migration | `php artisan migrate:status` | All `Ran` |
| Routes | `php artisan route:list --path=api` | New routes visible |
| Queue | `php artisan queue:work --once` | Job processes cleanly |
| Code style | `./vendor/bin/pint --test` | PSR-12 passes |
| Pruning | `php artisan model:prune --pretend` | Correct records listed |

## Reference Guides

| Topic | File | Load When |
|-------|------|-----------|
| Eloquent deep-dive | `references/eloquent.md` | Complex queries, relationships, scopes |
| Module architecture | `references/architecture.md` | Creating new modules, cross-module patterns |
| Senior architecture patterns | `references/patterns.md` | DDD, Action, Repository, DTO, Value Objects, Strategy, Events |
