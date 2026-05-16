---
name: laravel-testing
description: >-
  Laravel 13 testing with Pest PHP. Covers feature tests, unit tests, factories,
  mocking, database testing, HTTP testing, and TDD workflow. Use when writing
  Pest tests, creating model factories, mocking services, testing API endpoints,
  testing queue jobs, or setting up test suites for Laravel applications.
  Invoke for Pest, PHPUnit, factories, fakes, mocking, feature tests, TDD.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: Pest, PHPUnit, test, factory, mock, fake, TDD, feature test, unit test, RefreshDatabase, assertJson
  role: specialist
  scope: testing
  output-format: code
  related-skills: laravel-specialist, php-modernization
---

# Laravel Testing Specialist

Pest PHP first. No PHPUnit verbosity. No untested business logic.

## Test Architecture

```
tests/
├── Feature/          ← HTTP, database, integration
│   └── Modules/
│       └── Inventory/
│           └── AssetTest.php
├── Unit/             ← Pure logic, no DB, no HTTP
│   └── Modules/
│       └── Inventory/
│           └── CreateActionTest.php
└── Pest.php          ← Global uses, helpers
```

## Core Patterns

### Feature Test (API endpoint)

```php
<?php declare(strict_types=1);

use App\Models\User;
use Modules\Inventory\App\Models\Asset;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;

uses(\Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function () {
    $this->user = User::factory()->create();
    $this->actingAs($this->user, 'sanctum');
});

it('creates an asset successfully', function () {
    $location = \Modules\Setting\App\Models\Location::factory()->create();

    $response = $this->postJson(route('api.inventory.assets.store'), [
        'name'        => 'Laptop Dell XPS',
        'code'        => 'ASSET-001',
        'status'      => AssetStatusEnum::ACTIVE->value,
        'location_id' => $location->id,
    ]);

    $response->assertOk()
        ->assertJson(['status' => 'success']);

    $this->assertDatabaseHas('asset_assets', [
        'code' => 'ASSET-001',
    ]);
});

it('fails validation when code is missing', function () {
    $response = $this->postJson(route('api.inventory.assets.store'), [
        'name'   => 'Laptop',
        'status' => AssetStatusEnum::ACTIVE->value,
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['code']);
});

it('returns paginated asset list', function () {
    Asset::factory()->count(5)->create();

    $response = $this->getJson(route('api.inventory.assets.index'));

    $response->assertOk()
        ->assertJsonStructure([
            'status',
            'data' => [
                'data' => [['id', 'name', 'code', 'status']],
                'total',
                'per_page',
            ],
        ]);
});
```

### Unit Test (Action/UseCase)

```php
<?php declare(strict_types=1);

use Modules\Inventory\App\UseCases\Asset\CreateAction;
use Modules\Inventory\App\Repositories\AssetRepository;

it('creates asset via repository', function () {
    $repository = Mockery::mock(AssetRepository::class);
    $repository->shouldReceive('create')
        ->once()
        ->with(['name' => 'Laptop', 'code' => 'A001'])
        ->andReturn(true);

    $action = new CreateAction($repository);
    $action(['name' => 'Laptop', 'code' => 'A001']);
});
```

### Factory

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Modules\Inventory\App\Enums\Models\Asset\AssetStatusEnum;
use Modules\Inventory\App\Models\Asset;

class AssetFactory extends Factory
{
    protected $model = Asset::class;

    public function definition(): array
    {
        return [
            'name'        => $this->faker->words(3, true),
            'code'        => strtoupper($this->faker->bothify('ASSET-####')),
            'status'      => $this->faker->randomElement(AssetStatusEnum::cases())->value,
            'location_id' => \Modules\Setting\App\Models\Location::factory(),
        ];
    }

    public function active(): static
    {
        return $this->state(['status' => AssetStatusEnum::ACTIVE->value]);
    }

    public function disposed(): static
    {
        return $this->state(['status' => AssetStatusEnum::DISPOSED->value]);
    }
}
```

### Testing Queue Jobs

```php
it('dispatches notification job on asset creation', function () {
    \Illuminate\Support\Facades\Queue::fake();

    $location = \Modules\Setting\App\Models\Location::factory()->create();

    $this->postJson(route('api.inventory.assets.store'), [
        'name'        => 'Laptop',
        'code'        => 'A001',
        'status'      => AssetStatusEnum::ACTIVE->value,
        'location_id' => $location->id,
    ])->assertOk();

    \Illuminate\Support\Facades\Queue::assertPushed(
        \Modules\Inventory\App\Jobs\SendAssetNotification::class
    );
});
```

### Testing Events

```php
it('fires AssetCreated event', function () {
    \Illuminate\Support\Facades\Event::fake();

    $action = app(\Modules\Inventory\App\UseCases\Asset\CreateAction::class);
    $action(['name' => 'Laptop', 'code' => 'A001', 'status' => 'active', 'location_id' => 1]);

    \Illuminate\Support\Facades\Event::assertDispatched(
        \Modules\Inventory\App\Events\AssetCreated::class
    );
});
```

### Testing with Storage / Mail Fakes

```php
it('stores uploaded file', function () {
    \Illuminate\Support\Facades\Storage::fake('public');

    $file = \Illuminate\Http\UploadedFile::fake()->image('photo.jpg');

    $this->postJson(route('api.assets.upload'), ['file' => $file])
        ->assertOk();

    \Illuminate\Support\Facades\Storage::disk('public')->assertExists('assets/' . $file->hashName());
});
```

## Constraints

### MUST DO
- `uses(RefreshDatabase::class)` on all feature tests that touch DB
- `actingAs($user, 'sanctum')` for all authenticated endpoints
- Use factories — never raw `Model::create()` in tests
- Name tests in plain English: `it('does X when Y')`
- Assert both HTTP status AND response structure
- Test the unhappy path (validation errors, 404s, 403s)
- Use `Queue::fake()`, `Event::fake()`, `Mail::fake()` — never real side effects
- `Mockery::mock()` for unit tests, factories for feature tests

### MUST NOT DO
- `$this->assertTrue(true)` — meaningless assertions
- Test implementation details (private methods)
- Share state between tests (`static` variables)
- Hit real external APIs in tests — always mock
- Use `@skip` without explanation

## Pest.php Global Setup

```php
<?php declare(strict_types=1);

uses(
    \Tests\TestCase::class,
    \Illuminate\Foundation\Testing\RefreshDatabase::class,
)->in('Feature');

uses(\Tests\TestCase::class)->in('Unit');
```

## Validation Checkpoints

| Stage | Command | Expected |
|-------|---------|----------|
| Run all tests | `php artisan test` or `./vendor/bin/pest` | All green |
| With coverage | `./vendor/bin/pest --coverage --min=80` | ≥80% coverage |
| Parallel | `./vendor/bin/pest --parallel` | Passes |
