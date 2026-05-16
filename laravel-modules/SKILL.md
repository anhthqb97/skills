---
name: laravel-modules
description: >-
  Laravel modular architecture with nwidart/laravel-modules. Covers module
  creation, directory structure, service providers, cross-module communication
  via Events and Services, module-scoped routes, migrations, and lang files.
  Use when creating new modules, structuring cross-module interactions, wiring
  service providers, or designing modular Laravel applications.
  Invoke for module, nwidart, laravel-modules, ServiceProvider, cross-module, event, service.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: module, nwidart, laravel-modules, ServiceProvider, cross-module, event, service, artisan module:make
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-api
---

# Laravel Modules Specialist

One module = one bounded context. No cross-module model imports.

## Module Structure

```
Modules/
└── Inventory/
    ├── App/
    │   ├── Console/
    │   ├── Enums/
    │   │   └── Models/Asset/
    │   │       └── AssetStatusEnum.php
    │   ├── Events/
    │   │   └── AssetCreated.php
    │   ├── Http/
    │   │   ├── Controllers/Api/
    │   │   ├── Requests/Asset/
    │   │   └── Resources/
    │   ├── Jobs/
    │   ├── Listeners/
    │   ├── Models/
    │   │   └── Asset.php
    │   ├── Policies/
    │   ├── Providers/
    │   │   ├── InventoryServiceProvider.php
    │   │   └── RouteServiceProvider.php
    │   ├── Repositories/
    │   │   └── AssetRepository.php
    │   └── UseCases/
    │       └── Asset/
    │           ├── CreateAction.php
    │           ├── UpdateAction.php
    │           └── DeleteAction.php
    ├── Database/
    │   ├── Factories/
    │   ├── Migrations/
    │   └── Seeders/
    ├── lang/
    │   ├── en/
    │   │   └── lang.php
    │   └── vi/
    │       └── lang.php
    ├── resources/
    │   └── views/
    ├── routes/
    │   ├── api.php
    │   └── web.php
    └── module.json
```

## Artisan Commands

```bash
# Create a new module
php artisan module:make Inventory

# Create components inside a module
php artisan module:make-model Asset Inventory
php artisan module:make-migration create_asset_assets_table Inventory
php artisan module:make-controller AssetController Inventory
php artisan module:make-request CreateRequest Inventory
php artisan module:make-job SendAssetNotification Inventory
php artisan module:make-event AssetCreated Inventory
php artisan module:make-listener HandleAssetCreated Inventory
php artisan module:make-policy AssetPolicy Inventory
php artisan module:make-seeder AssetSeeder Inventory
php artisan module:make-factory AssetFactory Inventory
```

## Service Provider

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\Inventory\App\Policies\AssetPolicy;
use Modules\Inventory\App\Models\Asset;

final class InventoryServiceProvider extends ServiceProvider
{
    protected string $moduleName = 'Inventory';
    protected string $moduleNameLower = 'inventory';

    public function register(): void
    {
        $this->app->register(RouteServiceProvider::class);
    }

    public function boot(): void
    {
        $this->registerTranslations();
        $this->registerMigrations();
        $this->registerViews();
        $this->registerPolicies();
        $this->registerEvents();
    }

    private function registerPolicies(): void
    {
        \Illuminate\Support\Facades\Gate::policy(Asset::class, AssetPolicy::class);
    }

    private function registerEvents(): void
    {
        \Illuminate\Support\Facades\Event::listen(
            \Modules\Inventory\App\Events\AssetCreated::class,
            \Modules\Inventory\App\Listeners\SendAssetCreatedNotification::class,
        );
    }

    private function registerTranslations(): void
    {
        $langPath = resource_path('lang/modules/' . $this->moduleNameLower);

        if (is_dir($langPath)) {
            $this->loadTranslationsFrom($langPath, $this->moduleNameLower);
        } else {
            $this->loadTranslationsFrom(
                module_path($this->moduleName, 'lang'),
                $this->moduleNameLower
            );
        }
    }

    private function registerMigrations(): void
    {
        $this->loadMigrationsFrom(module_path($this->moduleName, 'Database/Migrations'));
    }

    private function registerViews(): void
    {
        $this->loadViewsFrom(module_path($this->moduleName, 'resources/views'), $this->moduleNameLower);
    }
}
```

## Cross-Module Communication

Cross-module communication must go through **Events** or **shared Services** — never direct model imports.

```
Inventory Module         →   Event: AssetDisposed
                              ↓
                         Maintenance Module listens → closes open tickets
                         Notification Module listens → sends alert
```

### Firing an Event

```php
// In Action
use Modules\Inventory\App\Events\AssetCreated;

final class CreateAction
{
    public function __invoke(array $data): Asset
    {
        $asset = $this->repository->create($data);
        AssetCreated::dispatch($asset);
        return $asset;
    }
}
```

### Event Class

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Modules\Inventory\App\Models\Asset;

final class AssetCreated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly Asset $asset,
    ) {}
}
```

### Listener in Another Module

```php
<?php declare(strict_types=1);

namespace Modules\Notification\App\Listeners;

use Modules\Inventory\App\Events\AssetCreated;

final class SendAssetCreatedAlert
{
    public function handle(AssetCreated $event): void
    {
        // Access $event->asset — no direct model import needed
        // use data only, don't import Asset model here
    }
}
```

## Module Lang Files

```php
// Modules/Inventory/lang/en/lang.php
<?php declare(strict_types=1);

return [
    'created'       => 'Asset created successfully.',
    'updated'       => 'Asset updated successfully.',
    'deleted'       => 'Asset deleted successfully.',
    'action_failed' => 'Action failed. Please try again.',
    'fields' => [
        'name'        => 'Asset Name',
        'code'        => 'Asset Code',
        'status'      => 'Status',
        'location'    => 'Location',
    ],
    'status' => [
        'active'   => 'Active',
        'inactive' => 'Inactive',
        'disposed' => 'Disposed',
    ],
];

// Modules/Inventory/lang/vi/lang.php — Vietnamese mirror
return [
    'created'       => 'Tạo tài sản thành công.',
    // ...
];
```

## module.json

```json
{
    "name": "Inventory",
    "alias": "inventory",
    "description": "Inventory management module",
    "keywords": [],
    "priority": 0,
    "providers": [
        "Modules\\Inventory\\App\\Providers\\InventoryServiceProvider"
    ],
    "aliases": {},
    "files": []
}
```

## Constraints

### MUST DO
- One module = one bounded context (e.g. Inventory, HR, Setting, Notification)
- Cross-module data sharing via Events or shared Service classes only
- Each module owns its own migrations, lang files, routes, and service provider
- Register Policies inside the module's ServiceProvider — not in `AuthServiceProvider`
- Use `module_path()` helper for all module-internal path resolution
- Use `__('inventory::lang.key')` for all module translations
- Prefix all DB table names with module alias: `asset_assets`, `setting_locations`

### MUST NOT DO
- Import a model from another module directly in an Action or Repository
- Share a single migration file across modules
- Register cross-module event listeners in `EventServiceProvider` — use each module's own provider
- Use hardcoded strings — always `__()` with module namespace
