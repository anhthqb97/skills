---
name: laravel-livewire
description: >-
  Laravel Livewire 3 component development. Covers full-page components,
  nested components, Alpine.js integration, real-time validation, file uploads,
  pagination, events, lazy loading, and Volt single-file components. Use when
  building Livewire components, implementing real-time UI, creating forms with
  live validation, or building interactive tables without writing JavaScript.
  Invoke for Livewire, Volt, Alpine, wire:model, wire:click, component, real-time, form.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: frontend
  triggers: Livewire, Volt, Alpine, wire:model, wire:click, component, real-time, form, upload, pagination
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-security
---

# Laravel Livewire 3 Specialist

Livewire 3 + Alpine.js. Thin components. No business logic in components.

## Component Types

| Type | Use When |
|------|----------|
| Full-page component | Replaces a controller+view for a route |
| Nested component | Reusable UI unit (table, form, modal) |
| Volt (single-file) | Simple components — logic + template in one file |

## Full-Page Component

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Livewire\Asset;

use Livewire\Attributes\{Layout, Title, Url};
use Livewire\Component;
use Livewire\WithPagination;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;
use Modules\Inventory\App\Repositories\AssetRepository;

#[Layout('layouts.app')]
#[Title('Asset Management')]
final class AssetIndex extends Component
{
    use WithPagination;

    #[Url(as: 'q')]
    public string $search = '';

    #[Url]
    public string $status = '';

    public int $perPage = 15;

    public function updatingSearch(): void
    {
        $this->resetPage();
    }

    public function updatingStatus(): void
    {
        $this->resetPage();
    }

    public function render(AssetRepository $repository): \Illuminate\View\View
    {
        return view('inventory::livewire.asset.index', [
            'assets'   => $repository->listTable([
                'search'   => $this->search,
                'status'   => $this->status ?: null,
                'per_page' => $this->perPage,
            ]),
            'statuses' => AssetStatusEnum::cases(),
        ]);
    }
}
```

## Form Component with Validation

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Http\Livewire\Asset;

use Livewire\Attributes\{On, Validate};
use Livewire\Component;
use Illuminate\Support\Facades\{DB, Log};
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;
use Modules\Inventory\App\UseCases\Asset\CreateAction;

final class AssetCreateForm extends Component
{
    #[Validate('required|string|max:255')]
    public string $name = '';

    #[Validate('required|string|unique:asset_assets,code')]
    public string $code = '';

    #[Validate('required')]
    public string $status = '';

    #[Validate('required|exists:setting_locations,id')]
    public int|string $location_id = '';

    public bool $isOpen = false;

    #[On('open-asset-create')]
    public function open(): void
    {
        $this->isOpen = true;
        $this->reset(['name', 'code', 'status', 'location_id']);
        $this->resetValidation();
    }

    public function save(CreateAction $action): void
    {
        $this->validate();

        try {
            DB::transaction(fn() => $action(new \Modules\Inventory\App\DTOs\CreateAssetDTO(
                name:       $this->name,
                code:       $this->code,
                status:     $this->status,
                locationId: (int) $this->location_id,
            )));

            $this->isOpen = false;
            $this->dispatch('asset-created');
            $this->dispatch('notify', type: 'success', message: __('inventory::lang.created'));
        } catch (\Throwable $th) {
            Log::error($th);
            $this->dispatch('notify', type: 'error', message: __('inventory::lang.action_failed'));
        }
    }

    public function render(): \Illuminate\View\View
    {
        return view('inventory::livewire.asset.create-form', [
            'statuses'  => AssetStatusEnum::cases(),
            'locations' => \Modules\Setting\App\Models\Location::active()->get(['id', 'name']),
        ]);
    }
}
```

## Blade Template (component view)

```blade
{{-- inventory::livewire.asset.index --}}
<div>
    {{-- Filters --}}
    <div class="flex gap-4 mb-4">
        <input
            wire:model.live.debounce.300ms="search"
            type="text"
            placeholder="{{ __('inventory::lang.search') }}"
            class="input input-bordered"
        />
        <select wire:model.live="status" class="select select-bordered">
            <option value="">{{ __('inventory::lang.all_statuses') }}</option>
            @foreach($statuses as $s)
                <option value="{{ $s->value }}">{{ $s->label() }}</option>
            @endforeach
        </select>
        <button
            wire:click="$dispatch('open-asset-create')"
            class="btn btn-primary"
        >
            {{ __('inventory::lang.create') }}
        </button>
    </div>

    {{-- Table --}}
    <table class="table w-full">
        <thead>
            <tr>
                <th>{{ __('inventory::lang.fields.name') }}</th>
                <th>{{ __('inventory::lang.fields.code') }}</th>
                <th>{{ __('inventory::lang.fields.status') }}</th>
                <th>{{ __('inventory::lang.fields.location') }}</th>
            </tr>
        </thead>
        <tbody>
            @forelse($assets as $asset)
                <tr wire:key="{{ $asset->id }}">
                    <td>{{ $asset->name }}</td>
                    <td>{{ $asset->code }}</td>
                    <td>
                        <span class="badge badge-{{ $asset->status->color() }}">
                            {{ $asset->status->label() }}
                        </span>
                    </td>
                    <td>{{ $asset->location?->name }}</td>
                </tr>
            @empty
                <tr><td colspan="4">{{ __('inventory::lang.no_data') }}</td></tr>
            @endforelse
        </tbody>
    </table>

    {{ $assets->links() }}

    {{-- Nested form modal --}}
    <livewire:inventory::asset-create-form />
</div>
```

## File Upload

```php
use Livewire\WithFileUploads;

final class AssetImport extends Component
{
    use WithFileUploads;

    #[Validate('required|file|mimes:xlsx,csv|max:10240')]
    public ?\Livewire\Features\SupportFileUploads\TemporaryUploadedFile $file = null;

    public function import(ImportAction $action): void
    {
        $this->validate();

        $path = $this->file->store('imports', 'local');
        $action(new \Modules\Inventory\App\DTOs\ImportAssetDTO(path: $path));

        $this->file = null;
        $this->dispatch('import-started');
    }
}
```

## Lazy Loading Heavy Components

```php
#[Lazy]
final class AssetStats extends Component
{
    public function render(AssetRepository $repository): \Illuminate\View\View
    {
        return view('inventory::livewire.asset.stats', [
            'stats' => $repository->getStats(),
        ]);
    }
}
```

```blade
{{-- Show placeholder while loading --}}
<livewire:inventory::asset-stats lazy />
```

## Component Events (inter-component communication)

```php
// Dispatch from child
$this->dispatch('asset-created', id: $asset->id);

// Listen in parent
#[On('asset-created')]
public function refreshList(): void
{
    $this->resetPage();
}
```

## Authorization in Livewire Components

```php
// Always authorize in mount() and in action methods
final class AssetIndex extends Component
{
    public function mount(): void
    {
        $this->authorize('viewAny', Asset::class);
    }

    public function delete(int $id): void
    {
        $asset = Asset::findOrFail($id);
        $this->authorize('delete', $asset);

        try {
            DB::transaction(fn() => app(DeleteAction::class)(new DeleteAssetDTO(id: $id)));
            $this->dispatch('notify', type: 'success', message: __('inventory::lang.deleted'));
        } catch (\Throwable $th) {
            Log::error($th);
            $this->dispatch('notify', type: 'error', message: __('inventory::lang.action_failed'));
        }
    }
}
```

## Computed Properties (cache expensive renders)

```php
use Livewire\Attributes\Computed;

final class AssetIndex extends Component
{
    public string $search = '';

    // Cache result for this render cycle — recalculates only when $search changes
    #[Computed]
    public function assets(): \Illuminate\Pagination\LengthAwarePaginator
    {
        return app(AssetRepository::class)->listTable([
            'search'   => $this->search,
            'per_page' => 15,
        ]);
    }

    #[Computed]
    public function totalActive(): int
    {
        return Asset::active()->count();
    }

    public function render(): \Illuminate\View\View
    {
        return view('inventory::livewire.asset.index');
        // Access in blade: $this->assets, $this->totalActive
    }
}
```

## Polling vs WebSocket Strategy

| Need | Solution | Code |
|------|----------|------|
| Update every N seconds | `wire:poll` | `<div wire:poll.5s>` |
| Real-time push (server → browser) | Laravel Echo + Reverb | `Echo.channel('assets').listen(...)` |
| Job progress tracking | Livewire + polling | `wire:poll.2s="checkStatus"` |
| Chat / live feed | Reverb WebSocket | Full duplex, persistent connection |

```blade
{{-- Polling — refresh stats every 10 seconds --}}
<div wire:poll.10s="$refresh">
    Active assets: {{ $this->totalActive }}
</div>

{{-- Only poll when tab is visible --}}
<div wire:poll.10s.visible="$refresh">
    {{ $this->totalActive }}
</div>
```

## Constraints

### MUST DO
- Inject repositories/actions via `render()` parameter or action method — not constructor
- Use `#[Validate]` attributes on properties — real-time validation
- Use `#[Url]` on filterable properties — bookmarkable URLs
- `wire:key` on every `@foreach` item — prevents DOM diffing bugs
- `DB::transaction()` on all write operations inside components
- `try/catch(\Throwable)` + dispatch notify event on all saves
- Use `$this->resetValidation()` when opening a form modal
- `wire:model.live.debounce.300ms` on search inputs — avoids excessive requests
- `#[Lazy]` on heavy components (stats, charts, large tables)

### MUST NOT DO
- Put business logic or DB queries directly in component — use Actions/Repositories
- Use `wire:model` without `.live` or `.blur` on large forms — causes too many requests
- Skip `wire:key` in loops — causes rendering bugs
- Access `$request` inside Livewire component — use `$this->property`
- Return raw arrays from `render()` — always pass to named view
