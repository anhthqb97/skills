---
name: laravel-security
description: >-
  Laravel 13 security hardening. Covers Policies, Gates, Sanctum token scopes,
  mass assignment protection, SQL injection prevention, XSS, CSRF, rate limiting,
  input sanitization, and OWASP Top 10 for Laravel APIs. Use when implementing
  authorization, protecting routes, writing Policies, configuring Sanctum scopes,
  hardening models, or auditing security vulnerabilities.
  Invoke for Policy, Gate, authorize, Sanctum, scope, middleware, CSRF, XSS, SQL injection, security.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: Policy, Gate, authorize, Sanctum, scope, middleware, CSRF, XSS, SQL injection, security, permission, role, auth
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-api
---

# Laravel Security Specialist

Deny by default. Validate everything. Trust nothing from the client.

## Authorization Layers

```
Route Middleware → Gate/Policy → Controller → Action
```

| Layer | Tool | Purpose |
|-------|------|---------|
| Route | `auth:sanctum` middleware | Must be authenticated |
| Route | `ability:read-assets` | Token scope check |
| Controller | `$this->authorize('create', Asset::class)` | Policy check |
| FormRequest | `authorize(): bool` | Fine-grained per-request |

## Code Patterns

### Policy

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Policies;

use App\Models\User;
use Modules\Inventory\App\Models\Asset;

final class AssetPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->hasPermissionTo('inventory.assets.view');
    }

    public function view(User $user, Asset $asset): bool
    {
        return $user->hasPermissionTo('inventory.assets.view');
    }

    public function create(User $user): bool
    {
        return $user->hasPermissionTo('inventory.assets.create');
    }

    public function update(User $user, Asset $asset): bool
    {
        return $user->hasPermissionTo('inventory.assets.update');
    }

    public function delete(User $user, Asset $asset): bool
    {
        return $user->hasPermissionTo('inventory.assets.delete')
            && $asset->deleted_at === null;
    }
}
```

### Register Policy

```php
// app/Providers/AuthServiceProvider.php
protected $policies = [
    \Modules\Inventory\App\Models\Asset::class => \Modules\Inventory\App\Policies\AssetPolicy::class,
];
```

### Controller with Policy Enforcement

```php
public function index(ListRequest $request): AssetCollection
{
    $this->authorize('viewAny', Asset::class);
    return new AssetCollection($this->repository->listTable($request->validated()));
}

public function store(CreateRequest $request, CreateAction $action): JsonResponse
{
    $this->authorize('create', Asset::class);
    // ...
}

public function destroy(int $id): JsonResponse
{
    $asset = $this->repository->findOrFail($id);
    $this->authorize('delete', $asset);
    // ...
}
```

### Sanctum Token Scopes

```php
// Issue token with abilities
$token = $user->createToken('api-token', [
    'inventory:read',
    'inventory:write',
])->plainTextToken;

// Route protection by ability
Route::middleware(['auth:sanctum', 'ability:inventory:read'])->group(function () {
    Route::get('inventory/assets', [AssetController::class, 'index']);
});

Route::middleware(['auth:sanctum', 'ability:inventory:write'])->group(function () {
    Route::post('inventory/assets', [AssetController::class, 'store']);
});
```

### FormRequest Authorization

```php
public function authorize(): bool
{
    // For update: ensure user owns or has permission on specific resource
    if ($this->route('asset')) {
        return $this->user()->can('update', $this->route('asset'));
    }
    return $this->user()->can('create', Asset::class);
}
```

### Mass Assignment Protection

```php
// GOOD — explicit allowlist via PHP Attribute
#[Fillable(['name', 'code', 'status', 'location_id'])]
final class Asset extends Model {}

// GOOD — only validated data passed to action
$action($request->validated());
```

### SQL Injection Prevention

```php
// GOOD — parameterized bindings
$this->model->whereRaw('name LIKE ?', ["%{$search}%"])->get();
$this->model->where('status', $status)->get();

// BAD — never interpolate user input
$this->model->whereRaw("name LIKE '%{$search}%'")->get(); // NEVER
```

### Sensitive Data — Hidden Fields

```php
#[Hidden(['password', 'remember_token', 'two_factor_secret'])]
final class User extends Authenticatable {}
```

### Rate Limiting on Sensitive Routes

```php
// Login endpoint
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip())
        ->response(fn() => response()->json([
            'status'  => 'error',
            'message' => 'Too many attempts. Try again later.',
        ], 429));
});

Route::middleware('throttle:login')->post('/login', [AuthController::class, 'login']);
```

### Prevent User Enumeration (Timing-safe Auth)

```php
public function login(LoginRequest $request): JsonResponse
{
    if (!Auth::attempt($request->only('email', 'password'))) {
        // Generic message — never reveal if email exists
        return response()->json([
            'status'  => 'error',
            'message' => __('auth.failed'),
        ], 401);
    }
    // ...
}
```

## Security Checklist

| Risk | Laravel Mitigation | Status |
|------|--------------------|--------|
| SQL Injection | Eloquent ORM / parameterized queries | Auto if using Eloquent |
| XSS | Blade `{{ }}` auto-escapes | Auto in Blade |
| CSRF | `VerifyCsrfToken` middleware | Auto on web routes |
| Mass Assignment | `#[Fillable]` on all models | Manual — always required |
| Broken Auth | Sanctum + token abilities | Manual |
| Sensitive Data | `#[Hidden]`, never log passwords | Manual |
| Rate Limiting | `RateLimiter::for()` | Manual |
| IDOR | Policy `view($user, $model)` | Manual — always required |
| Stack Trace Leak | `try/catch` + generic error response | Manual — always required |
| Insecure Direct Object Reference | `findOrFail()` + Policy check | Manual |

## Constraints

### MUST DO
- Register a Policy for every business entity
- Call `$this->authorize()` at the top of every controller method
- Use `ability:` middleware for Sanctum token scopes on sensitive routes
- Use `#[Fillable]` — explicit allowlist only
- Use `$request->validated()` on all write paths
- Return generic `__('auth.failed')` message — never reveal which field failed
- Rate-limit login, password-reset, and OTP endpoints
- `#[Hidden]` on all sensitive model fields

### MUST NOT DO
- `$guarded = []` on any model
- Expose exception messages or stack traces in API responses
- Use `$request->all()` on write paths
- Return different error messages for "user not found" vs "wrong password"
- Log user passwords or tokens
- Skip policy checks for any authenticated endpoint
