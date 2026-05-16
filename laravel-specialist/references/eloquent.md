# Eloquent Deep-Dive — Laravel 13

## Relationships

```php
// Always declare return types
public function department(): BelongsTo
{
    return $this->belongsTo(Department::class, 'department_id');
}

public function assets(): HasMany
{
    return $this->hasMany(Asset::class, 'location_id');
}

public function roles(): BelongsToMany
{
    return $this->belongsToMany(Role::class, 'user_roles', 'user_id', 'role_id')
                ->withTimestamps();
}
```

## Eager Loading (mandatory — never N+1)

```php
// In Repository::listTable()
$this->model->query()
    ->with(['location', 'department', 'category'])
    ->withCount('workOrders')
    ->paginate(15);

// Conditional loading
->with(['media' => fn($q) => $q->where('type', 'image')])
```

## Query Scopes

```php
// Local scope — called as ->active()
public function scopeActive(Builder $query): Builder
{
    return $query->where('status', AssetStatusEnum::ACTIVE);
}

// Parameterized scope
public function scopeOfType(Builder $query, string $type): Builder
{
    return $query->where('type', $type);
}
```

## Casts (Laravel 13)

```php
protected function casts(): array
{
    return [
        'status'       => AssetStatusEnum::class,       // backed enum
        'metadata'     => 'array',
        'purchased_at' => 'immutable_date',
        'settings'     => AsValueObject::class . ':array', // cast to value object
    ];
}
```

## Attribute Accessors/Mutators (Laravel 9+ style)

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function fullName(): Attribute
{
    return Attribute::make(
        get: fn() => trim("{$this->first_name} {$this->last_name}"),
    );
}
```

## Model Events — Prefer Observers

```php
// Register via Attribute (Laravel 13)
#[ObservedBy(AssetObserver::class)]
final class Asset extends Model {}

// Observer
final class AssetObserver
{
    public function created(Asset $asset): void
    {
        // audit log, etc.
    }
}
```

## Soft Deletes

```php
use SoftDeletes;

// Restore
$asset->restore();

// Force delete (admin only)
$asset->forceDelete();

// Query including trashed
Asset::withTrashed()->where('code', $code)->first();
```

## Complex Filters (in Repository)

```php
public function listTable(array $filters): LengthAwarePaginator
{
    return $this->model->query()
        ->with(['location', 'category'])
        ->when($filters['status'] ?? null,      fn($q, $v) => $q->where('status', $v))
        ->when($filters['location_id'] ?? null, fn($q, $v) => $q->where('location_id', $v))
        ->when($filters['search'] ?? null,      fn($q, $v) => $q->where(function ($q) use ($v) {
            $q->where('name', 'like', "%{$v}%")
              ->orWhere('code', 'like', "%{$v}%");
        }))
        ->when($filters['date_from'] ?? null, fn($q, $v) => $q->whereDate('created_at', '>=', $v))
        ->orderBy($filters['sort'] ?? 'created_at', $filters['direction'] ?? 'desc')
        ->paginate($filters['per_page'] ?? 15);
}
```

## Tree Queries (self-referential)

```php
public function getActiveTree(): Collection
{
    return $this->model->query()
        ->with('children')
        ->whereNull('parent_id')
        ->where('status', LocationStatusEnum::ACTIVE)
        ->orderBy('sort_order')
        ->get();
}
```
