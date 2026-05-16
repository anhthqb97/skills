---
name: laravel-performance
description: >-
  Laravel 13 performance optimization. Covers N+1 detection, eager loading,
  query optimization, Redis caching, cache tags, database indexing, chunking,
  lazy collections, and Octane. Use when optimizing slow queries, fixing N+1
  problems, implementing caching strategies, adding database indexes, or
  profiling Laravel applications.
  Invoke for N+1, eager load, with(), cache, Redis, index, chunk, lazy, Octane, query optimization.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: N+1, eager load, with(), cache, Redis, index, chunk, lazy, Octane, slow query, performance, optimize
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-queue
---

# Laravel Performance Specialist

Measure first. Fix the right thing. Never guess.

## Performance Hierarchy

```
1. Fix N+1 queries      ← biggest gains, always first
2. Add DB indexes       ← second biggest, often missed
3. Cache hot data       ← Redis for repeated reads
4. Chunk bulk ops       ← memory control for large datasets
5. Octane / async       ← infrastructure, last resort
```

## N+1 Detection & Fix

```php
// DETECT — add to AppServiceProvider in local env
if (app()->isLocal()) {
    \Illuminate\Database\Eloquent\Model::preventLazyLoading();
}

// BAD — N+1: fires 1 + N queries
$assets = Asset::all();
foreach ($assets as $asset) {
    echo $asset->location->name; // query per iteration
}

// GOOD — eager load: 2 queries total
$assets = Asset::with(['location', 'category'])->paginate(15);

// GOOD — conditional eager load in repository
public function listTable(array $filters): LengthAwarePaginator
{
    return $this->model->query()
        ->with(['location:id,name', 'category:id,name']) // select only needed columns
        ->when($filters['status'] ?? null, fn($q, $v) => $q->where('status', $v))
        ->paginate($filters['per_page'] ?? 15);
}
```

## Database Indexing

```php
// Migration — always index foreign keys and filter columns
Schema::create('asset_assets', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('code')->unique();           // unique index
    $table->string('status', 20)->index();      // filter column
    $table->foreignId('location_id')
          ->constrained('setting_locations')
          ->cascadeOnDelete();                  // FK auto-indexes
    $table->softDeletes();                      // index deleted_at
    $table->timestamps();

    // Composite index for common filter combos
    $table->index(['status', 'location_id']);
    $table->index(['deleted_at', 'status']);    // soft delete + filter
});
```

## Redis Caching

```php
// Cache a slow query result
public function getActiveLocations(): Collection
{
    return Cache::remember(
        key: 'setting.locations.active',
        ttl: now()->addHours(6),
        callback: fn() => Location::active()->orderBy('name')->get(['id', 'name'])
    );
}

// Cache with tags (allows group invalidation)
public function getAssetStats(): array
{
    return Cache::tags(['inventory', 'assets'])->remember(
        key: 'inventory.assets.stats',
        ttl: now()->addMinutes(30),
        callback: fn() => [
            'total'    => Asset::count(),
            'active'   => Asset::active()->count(),
            'disposed' => Asset::where('status', 'disposed')->count(),
        ]
    );
}

// Invalidate on write
public function create(array $data): Asset
{
    $asset = $this->model->create($data);
    Cache::tags(['inventory', 'assets'])->flush();
    return $asset;
}

// Extend TTL without re-fetching (Laravel 13)
Cache::touch('setting.locations.active', now()->addHours(6));
```

## Chunking Large Datasets

```php
// BAD — loads all into memory
Asset::all()->each(fn($asset) => $this->process($asset));

// GOOD — chunk: fixed memory, processes in batches
Asset::chunk(500, fn($assets) => $assets->each(fn($a) => $this->process($a)));

// BETTER — lazy collection: cursor-based, minimal memory
Asset::lazy()->each(fn($asset) => $this->process($asset));

// BEST for reports — chunkById: stable pagination
Asset::chunkById(500, function ($assets) {
    foreach ($assets as $asset) {
        $this->process($asset);
    }
});
```

## Select Only Needed Columns

```php
// BAD — fetches all columns including large text fields
Asset::with('location')->get();

// GOOD — explicit column selection
Asset::select(['id', 'name', 'code', 'status', 'location_id'])
    ->with(['location:id,name'])
    ->get();
```

## Query Optimization Patterns

```php
// Use whereIn over multiple queries
$locationIds = [1, 2, 3];
Asset::whereIn('location_id', $locationIds)->get(); // 1 query

// Use exists() over count() for boolean checks
if (Asset::where('code', $code)->exists()) { ... }  // faster than count() > 0

// Avoid subqueries — use joins for filtering
Asset::join('setting_locations', 'asset_assets.location_id', '=', 'setting_locations.id')
    ->where('setting_locations.region', 'north')
    ->select('asset_assets.*')
    ->get();

// withCount for aggregates
Asset::withCount(['maintenances as open_count' => fn($q) => $q->where('status', 'open')])
    ->get();
```

## Caching Strategy by Data Type

| Data Type | TTL | Strategy |
|-----------|-----|----------|
| Config / lookup tables | 6–24h | `Cache::remember` + tag invalidation on update |
| User-specific data | 5–15min | Key includes `user_id` |
| Aggregates / stats | 15–60min | Tag-based, flush on write |
| Session / temp | Short TTL | Redis session driver |
| Never cache | — | Financial totals, real-time inventory counts |

## Laravel Octane (Production Throughput)

```php
// Install: composer require laravel/octane
// config/octane.php
'server'  => env('OCTANE_SERVER', 'frankenphp'), // or 'swoole', 'roadrunner'
'workers' => env('OCTANE_WORKERS', 4),           // CPU cores
'max_requests' => 500,                            // recycle workers to prevent memory leaks

// Start: php artisan octane:start --workers=4 --task-workers=2
```

```php
// Octane-safe — flush state between requests
final class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Re-bind per-request singletons to prevent state leaks
        \Laravel\Octane\Facades\Octane::tick('flush-state', function () {
            app()->forgetInstance(RequestContext::class);
        })->seconds(0);  // every request
    }
}
```

## Read Replica Routing

```php
// config/database.php — route reads to replica, writes to primary
'mysql' => [
    'read'  => ['host' => env('DB_READ_HOST', '127.0.0.1')],
    'write' => ['host' => env('DB_HOST', '127.0.0.1')],
    'sticky' => true,   // read-your-own-writes after a write
],

// Force primary for time-critical reads (after a write)
$asset = Asset::on('mysql::write')->find($id);

// Use read replica explicitly
$assets = Asset::on('mysql::read')->paginate(15);
```

## Query Analysis (EXPLAIN)

```php
// In dev — analyze slow queries before deploying
// AppServiceProvider::boot() — log queries over 100ms
if (app()->isLocal()) {
    \Illuminate\Support\Facades\DB::listen(function ($query) {
        if ($query->time > 100) {
            \Illuminate\Support\Facades\Log::warning('slow.query', [
                'sql'      => $query->sql,
                'bindings' => $query->bindings,
                'time_ms'  => $query->time,
            ]);
        }
    });
}

// Get EXPLAIN output for a query
$explain = \Illuminate\Support\Facades\DB::select(
    'EXPLAIN ' . Asset::where('status', 'active')->toSql(),
    Asset::where('status', 'active')->getBindings()
);
// Check type = 'ALL' (full scan) → add index
// Check rows = large number → composite index needed
```

## Full-Text Search vs LIKE

```php
// BAD for large tables — LIKE with leading wildcard kills indexes
Asset::where('name', 'like', "%{$search}%")->get();  // full table scan

// GOOD — MySQL full-text index
// Migration:
$table->fullText(['name', 'code']);

// Query:
Asset::whereFullText(['name', 'code'], $search)->paginate(15);

// BEST for complex search — dedicated search engine
// Meilisearch via Laravel Scout:
Asset::search($search)->paginate(15);  // instant, typo-tolerant
```

## Constraints

### MUST DO
- `Model::preventLazyLoading()` in local env — catches N+1 at dev time
- Eager load all relationships in repository queries with `with()`
- Add `index()` on all columns used in `WHERE`, `ORDER BY`, or `JOIN`
- Add `unique()` on business-key columns (code, slug, email)
- Add composite indexes for common filter combinations
- Use `chunk()` or `lazy()` for any operation over 1000 records
- Select specific columns — never `SELECT *` on large tables
- Use cache tags for grouped invalidation

### MUST NOT DO
- Load relationships inside loops — always eager load
- `Asset::all()` in production — always paginate or chunk
- Cache mutable financial or transactional data
- Add indexes after the fact without testing query plans
- Use `count()` for boolean checks — use `exists()`
