# Senior Laravel Architecture Patterns

> Source: [Best Architectural Patterns for Senior Laravel Developers](https://dev.to/abdulsalamamtech/best-architectural-patterns-approach-for-senior-laravel-developers-8m4)

Load this reference when: designing application structure, choosing between patterns, building a new module from scratch, or reviewing architecture decisions.

---

## Pattern Decision Map

| App Complexity | Recommended Pattern |
|----------------|---------------------|
| Simple CRUD | Laravel MVC (default) |
| Medium — multiple domains | Action/Service + Repository + DDD-lite |
| Large — enterprise | Modular Monolith + DDD + Event-driven |
| Extractable microservice | Modular Monolith (modules → services) |

---

## 1. Domain-Driven Design (DDD) / Modular Monolith — `arch-feature-folders`

Organize by **business domain**, not technical type. Each module is a bounded context.

```
Modules/
├── Inventory/     ← Asset management domain
├── WorkOrder/     ← Maintenance domain
├── Procurement/   ← Purchasing domain
├── Notification/  ← Cross-cutting concern
└── Core/          ← Shared kernel (BaseRepository, BaseEnum, etc.)
```

**Three layers inside each module:**

| Layer | Owns | Maps To |
|-------|------|---------|
| **Domain** | Entities, Value Objects, Domain Events, Enums | `Models/`, `Enums/`, `Events/` |
| **Application** | Use Cases, DTOs, Orchestration | `UseCases/`, `DTOs/` |
| **Infrastructure** | DB, External APIs, Mail | `Repositories/`, `Services/` |

---

## 2. Action / UseCase Pattern — `arch-action-classes`

Single-purpose class per operation. Reduces controller bloat. Fully testable in isolation.

```php
<?php declare(strict_types=1);

// One file = one operation
namespace Modules\Inventory\App\UseCases\Asset;

final class CreateAction        {} // create one asset
final class UpdateAction        {} // update fields
final class UpdateStatusAction  {} // status transition only
final class DeleteManyAction    {} // bulk soft delete
final class ExportCsvAction     {} // export to file
```

**Rule:** If an action grows beyond ~30 lines, extract sub-steps into private methods or a Service.

---

## 3. Repository Pattern — `arch-repository-pattern`

Abstracts all Eloquent queries. Allows swapping DB / adding cache without touching business logic.

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

    // Only query methods here — no business logic
    public function findByCode(string $code): ?Asset
    {
        return $this->model->where('code', $code)->first();
    }
}
```

**Dependency Inversion:** Bind interface → implementation in ServiceProvider for swap-ability:

```php
// AppServiceProvider or module ServiceProvider
$this->app->bind(
    \Modules\Inventory\App\Contracts\AssetRepositoryInterface::class,
    \Modules\Inventory\App\Repositories\AssetRepository::class,
);
```

---

## 4. DTO (Data Transfer Object) — `arch-dto-pattern`

Typed structs passed between layers. Replaces raw `array` and ensures type safety.

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

// Controller → Action flow
$action(CreateAssetDTO::fromArray($request->validated()));
```

---

## 5. Value Objects — `arch-value-objects`

Immutable objects that encapsulate domain concepts with validation built in.

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\ValueObjects;

final readonly class AssetCode
{
    public string $value;

    public function __construct(string $value)
    {
        if (!preg_match('/^[A-Z]{2,5}-\d{4}$/', $value)) {
            throw new \InvalidArgumentException("Invalid asset code: {$value}");
        }
        $this->value = strtoupper($value);
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }
}

// Usage in DTO or model cast
$code = new AssetCode('ASSET-0001'); // validates on construction
```

---

## 6. Event-Driven Architecture — `arch-event-driven`

Decouples side effects from the main flow. Listeners handle notifications, audit logs, search indexing.

```
Action::__invoke()
    → repository->create()
    → AssetCreated::dispatch($asset)   ← fire and forget
         ↓
    Listeners (async on queue):
    - SendAssetCreatedNotification
    - WriteAuditLog
    - UpdateSearchIndex
```

```php
// In Action — fire event AFTER successful write
final class CreateAction
{
    public function __invoke(CreateAssetDTO $dto): Asset
    {
        $asset = $this->repository->create([...]);
        AssetCreated::dispatch($asset);   // decoupled side effects
        return $asset;
    }
}

// Listener — runs on queue, fully isolated
final class SendAssetCreatedNotification implements ShouldQueue
{
    public function handle(AssetCreated $event): void
    {
        // send notification about $event->asset
    }
}
```

---

## 7. Strategy Pattern

Handle varying behaviors at runtime — e.g. multiple export formats, payment gateways.

```php
<?php declare(strict_types=1);

interface ExportStrategyInterface
{
    public function export(Collection $data): string; // returns file path
}

final class CsvExportStrategy implements ExportStrategyInterface
{
    public function export(Collection $data): string { /* ... */ }
}

final class ExcelExportStrategy implements ExportStrategyInterface
{
    public function export(Collection $data): string { /* ... */ }
}

// Action selects strategy at runtime
final class ExportAssetsAction
{
    public function __invoke(string $format, Collection $data): string
    {
        $strategy = match($format) {
            'csv'   => new CsvExportStrategy(),
            'excel' => new ExcelExportStrategy(),
            default => throw new \InvalidArgumentException("Unsupported format: {$format}"),
        };

        return $strategy->export($data);
    }
}
```

---

## 8. Service Classes — `arch-service-classes`

Cross-entity or cross-module orchestration. Never owns a single entity's CRUD.

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Services;

// Service = cross-cutting, cross-entity logic
final class AssetTransferService
{
    public function __construct(
        private readonly AssetRepository       $assets,
        private readonly LocationRepository    $locations,
        private readonly AuditRepository       $audit,
    ) {}

    public function transfer(Asset $asset, Location $destination): void
    {
        // Orchestrates multiple repositories + events
        $previousLocation = $asset->location;
        $this->assets->update($asset->id, ['location_id' => $destination->id]);
        $this->audit->log('asset.transferred', compact('asset', 'previousLocation', 'destination'));
        AssetTransferred::dispatch($asset, $previousLocation, $destination);
    }
}
```

---

## When to Use Which Pattern

| Situation | Use |
|-----------|-----|
| Single entity CRUD | Action + Repository |
| Cross-entity orchestration | Service class |
| Decoupled side effects | Event + Listener |
| Varying algorithms at runtime | Strategy pattern |
| Typed data between layers | DTO |
| Domain concept with validation | Value Object |
| Complex query logic | Repository scope method |
| Swappable implementations | Interface + ServiceProvider binding |

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why Bad | Fix |
|--------------|---------|-----|
| Fat Controller | Mixes HTTP + business logic | Move to Action |
| God Model | Model with 30+ methods | Split into scopes, observers, services |
| Direct cross-module import | Tight coupling between modules | Use Events or interface |
| `new ClassName()` in Action | Bypasses container, untestable | Inject via constructor |
| Raw arrays across all layers | No type safety, hard to refactor | Use DTOs |
| Business logic in migration | Migrations are schema only | Use Seeders or Actions |
