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
