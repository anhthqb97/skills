---
name: laravel-api
description: >-
  Laravel 13 REST API design patterns. Covers API versioning, JSON:API resources,
  rate limiting, API authentication with Sanctum, request/response envelopes,
  pagination, filtering, sorting, and API documentation. Use when designing API
  endpoints, creating API resources, implementing rate limiting, versioning APIs,
  or structuring API responses. Invoke for API, REST, resources, versioning,
  rate limit, Sanctum, pagination, filtering, sorting.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: API, REST, resource, versioning, rate limit, Sanctum, pagination, filtering, sorting, JsonResource, ApiResource
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-security
---

# Laravel API Specialist

REST API-first. Versioned. Typed responses. No leaking internals.

## API Structure

```
routes/
└── api.php           ← Route groups by version and module

app/Http/
├── Resources/        ← JsonResource transformers
└── Controllers/Api/  ← Thin API controllers

Modules/*/
└── App/
    ├── Http/
    │   ├── Controllers/Api/
    │   ├── Requests/
    │   └── Resources/
    └── routes/
        └── api.php
```

## Response Envelope

Always return this structure:

```json
{ "status": "success|error", "message": "...", "data": {}, "count": N }
```

| Operation | Shape |
|-----------|-------|
| Read list | `{ "status": "success", "data": [...], "count": N }` |
| Read single | `{ "status": "success", "data": {} }` |
| Write (create/update/delete) | `{ "status": "success", "message": "..." }` |
| Validation error | `{ "message": "...", "errors": {} }` — Laravel default, 422 |
| Server error | `{ "status": "error", "message": __('lang.action_failed') }` — never expose stack traces |

## HTTP Status Codes

| Code | When |
|------|------|
| 200 | All successful reads AND writes |
| 422 | Validation failure |
| 401 | Unauthenticated |
| 403 | Unauthorized (authenticated but no permission) |
| 404 | Resource not found |
| 500 | Unhandled server error |

## Code Patterns

### API Resource

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

final class AssetResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'       => $this->id,
            'name'     => $this->name,
            'code'     => $this->code,
            'status'   => [
                'value' => $this->status->value,
                'label' => $this->status->label(),
                'color' => $this->status->color(),
            ],
            'location' => new LocationResource($this->whenLoaded('location')),
            'created_at' => $this->created_at?->toISOString(),
            'updated_at' => $this->updated_at?->toISOString(),
        ];
    }
}
```

### API Resource Collection with Pagination

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

final class AssetCollection extends ResourceCollection
{
    public $collects = AssetResource::class;

    public function toArray(Request $request): array
    {
        return [
            'status' => 'success',
            'data'   => $this->collection,
            'count'  => $this->total(),
            'meta'   => [
                'current_page' => $this->currentPage(),
                'last_page'    => $this->lastPage(),
                'per_page'     => $this->perPage(),
            ],
        ];
    }
}
```

### Controller (Index + Show + Store)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Controllers\Api;

use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\{DB, Log};
use Modules\Inventory\App\Http\Requests\Asset\{CreateRequest, ListRequest};
use Modules\Inventory\App\Http\Resources\{AssetCollection, AssetResource};
use Modules\Inventory\App\Repositories\AssetRepository;
use Modules\Inventory\App\UseCases\Asset\CreateAction;

final class AssetController
{
    public function __construct(
        private readonly AssetRepository $repository,
    ) {}

    public function index(ListRequest $request): AssetCollection
    {
        $assets = $this->repository->listTable($request->validated());
        return new AssetCollection($assets);
    }

    public function show(int $id): JsonResponse
    {
        $asset = $this->repository->findOrFail($id);
        return response()->json([
            'status' => 'success',
            'data'   => new AssetResource($asset),
        ]);
    }

    public function store(CreateRequest $request, CreateAction $action): JsonResponse
    {
        try {
            DB::transaction(fn() => $action($request->validated()));
            return response()->json(['status' => 'success', 'message' => __('inventory::lang.created')]);
        } catch (\Throwable $th) {
            Log::error($th);
            return response()->json(['status' => 'error', 'message' => __('inventory::lang.action_failed')], 500);
        }
    }
}
```

### API Versioning (Route Groups)

```php
// routes/api.php
Route::prefix('v1')->name('api.v1.')->group(function () {
    Route::middleware('auth:sanctum')->group(function () {
        Route::apiResource('inventory/assets', AssetController::class);
    });
});
```

### Rate Limiting

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('sensitive', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
});
```

### List Request (Filtering + Sorting + Pagination)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Requests\Asset;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class ListRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'search'   => ['sometimes', 'string', 'max:255'],
            'status'   => ['sometimes', Rule::enum(AssetStatusEnum::class)],
            'sort_by'  => ['sometimes', 'string', Rule::in(['name', 'code', 'created_at'])],
            'sort_dir' => ['sometimes', 'string', Rule::in(['asc', 'desc'])],
            'per_page' => ['sometimes', 'integer', 'min:1', 'max:100'],
            'page'     => ['sometimes', 'integer', 'min:1'],
        ];
    }
}
```

## Constraints

### MUST DO
- Version all APIs under `/v1/`, `/v2/` prefix
- Use `JsonResource` / `ResourceCollection` — never return raw models
- Always include `status` key in every response
- Use `whenLoaded()` for relationships in resources — prevent N+1
- Validate all query params via `ListRequest` FormRequest
- Apply rate limiting to all API routes
- `auth:sanctum` middleware on all protected routes
- Return `200` for writes (not 201/204) — consistent with envelope pattern
- Never expose `id` auto-increment — use `uuid` or keep internal

### MUST NOT DO
- Return raw Eloquent model in response
- Skip `whenLoaded()` — always guard relationship loading
- Expose stack traces or exception messages
- Use `$request->all()` — always `$request->validated()`
- Return different envelope structure per endpoint
