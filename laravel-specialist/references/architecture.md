# Module Architecture — nwidart/laravel-modules + Laravel 13

## Module Directory Structure (Fixed)

```
Modules/[Name]/
├── app/
│   ├── Enums/Models/[Entity]/      ← StatusEnum, TypeEnum per entity
│   ├── Enums/Routes/               ← ApiEnum.php, WebEnum.php
│   ├── Http/Controllers/Api/       ← JSON API controllers
│   ├── Http/Controllers/Web/       ← Blade controllers
│   ├── Http/Requests/[Entity]/     ← CreateRequest, UpdateRequest
│   ├── Models/                     ← Eloquent models
│   ├── Repositories/               ← data access layer
│   ├── Services/                   ← cross-cutting domain services
│   ├── UseCases/[Entity]/          ← one Action class per operation
│   └── Providers/                  ← ServiceProvider, RouteServiceProvider
├── config/config.php
├── database/migrations/
├── database/seeders/
├── lang/en/lang.php
├── lang/vi/lang.php
├── resources/views/[entity]/
├── routes/api.php + routes/api/[entity].php
├── routes/web.php + routes/web/[entity].php
├── composer.json
└── module.json
```

## Table Naming Convention

| Module | Prefix | Example |
|--------|--------|---------|
| Settings | `setting_` | `setting_locations` |
| Asset | `asset_` | `asset_assets` |
| WorkOrder | `work_order_` | `work_order_orders` |
| Vendor | `vendor_` | `vendor_vendors` |
| Procurement | `procurement_` | `procurement_requests` |

## Cross-Module Communication

```php
// ✅ Correct — via service injection
final class CreateWorkOrderAction
{
    public function __construct(
        private readonly AssetServiceInterface $assetService,
    ) {}
}

// ✅ Correct — via events
event(new AssetAssigned($asset, $workOrder));

// ❌ Wrong — direct import across modules
use Modules\Asset\App\Models\Asset; // inside WorkOrder module
```

## Route Name Constants (Enum)

```php
// Enums/Routes/ApiEnum.php
enum ApiEnum: string
{
    case INDEX  = 'api.assets.index';
    case STORE  = 'api.assets.store';
    case UPDATE = 'api.assets.update';
    case DELETE = 'api.assets.destroy';
}

// Usage
route(ApiEnum::INDEX->value)
```

## Standard Action Verbs

| Verb | Example | Notes |
|------|---------|-------|
| `Create` | `CreateAction` | Creates single record |
| `Update` | `UpdateAction` | Updates single record |
| `UpdateStatus` | `UpdateStatusAction` | Status transition only |
| `DeleteMany` | `DeleteManyAction` | Bulk soft delete |
| `ListTable` | `ListTableAction` | Paginated datatable |
| `GetTree` | `GetTreeAction` | Hierarchical data |
| `GetByType` | `GetByTypeAction` | Filtered list |
| `ExportCsv` | `ExportCsvAction` | File export |

## Migration Naming

```
YYYY_MM_DD_NNNNNN_create_[prefix]_[entities]_table.php

# Example
2026_04_11_000001_create_asset_assets_table.php
```

Every table must have:
```php
$table->id();
$table->softDeletes();   // for all business entities
$table->timestamps();
// All FK columns must have ->index()
$table->unsignedBigInteger('location_id')->index();
$table->foreign('location_id')->references('id')->on('setting_locations');
```

## ServiceProvider Registration

```php
// Providers/SettingsServiceProvider.php
public function boot(): void
{
    $this->loadMigrationsFrom(module_path('Settings', 'database/migrations'));
    $this->loadTranslationsFrom(module_path('Settings', 'lang'), 'settings');
    $this->loadViewsFrom(module_path('Settings', 'resources/views'), 'settings');
}
```

## Phase Order (Implementation Dependency Graph)

```
Phase 1: Settings
Phase 2: Asset, Vendor
Phase 3: PM, WorkOrder, Allocation
Phase 4: ServiceRequest, Procurement, Disposal
Phase 5: Reports, Notifications, Ads, Personal
```

A module cannot be implemented if upstream phases are incomplete.
