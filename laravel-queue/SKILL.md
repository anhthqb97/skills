---
name: laravel-queue
description: >-
  Laravel 13 queue system with Horizon. Covers job classes with PHP Attributes,
  job batching, chaining, rate limiting, failed job handling, retry strategies,
  and Horizon monitoring configuration. Use when creating queue jobs, setting up
  Horizon, implementing job batching, handling failed jobs, or designing
  async processing workflows.
  Invoke for Queue, Job, Horizon, batch, chain, ShouldQueue, dispatch, failed, retry, worker.
license: MIT
metadata:
  author: https://github.com/anhthqb97
  version: "1.0.0"
  domain: backend
  triggers: Queue, Job, Horizon, batch, chain, ShouldQueue, dispatch, failed, retry, worker, async
  role: specialist
  scope: implementation
  output-format: code
  related-skills: laravel-specialist, laravel-performance
---

# Laravel Queue Specialist

Async by default. Idempotent jobs. Explicit failure handling.

## Job Anatomy (PHP 8.3 Attributes)

```php
<?php declare(strict_types=1);

namespace Modules\Inventory\App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\Attributes\{Connection, Queue, Tries, Timeout, WithoutOverlapping};
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Modules\Inventory\App\Models\Asset;

#[Connection('redis')]
#[Queue('notifications')]
#[Tries(3)]
#[Timeout(60)]
final class SendAssetNotification implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly Asset $asset,
        private readonly string $event,
    ) {}

    public function handle(): void
    {
        // Idempotency check — safe to retry
        if ($this->asset->notification_sent_at !== null) {
            return;
        }

        // do work...
        $this->asset->update(['notification_sent_at' => now()]);
    }

    public function failed(\Throwable $e): void
    {
        Log::error('SendAssetNotification failed', [
            'asset_id' => $this->asset->id,
            'event'    => $this->event,
            'error'    => $e->getMessage(),
        ]);
    }

    public function retryUntil(): \DateTime
    {
        return now()->addHours(2);
    }
}
```

## Dispatching Jobs

```php
// Simple dispatch
SendAssetNotification::dispatch($asset, 'created');

// Delayed dispatch
SendAssetNotification::dispatch($asset, 'created')
    ->delay(now()->addMinutes(5));

// Dispatch on specific queue
SendAssetNotification::dispatch($asset, 'created')
    ->onQueue('high-priority');

// Conditional dispatch
SendAssetNotification::dispatchIf($asset->status->isActive(), $asset, 'activated');
```

## Job Chaining

```php
// Jobs run sequentially — next only runs if previous succeeds
\Illuminate\Support\Facades\Bus::chain([
    new ProcessAssetImport($file),
    new ValidateAssetData($import),
    new NotifyImportComplete($user),
])->onQueue('imports')->dispatch();
```

## Job Batching

```php
// Jobs run in parallel — track progress
$batch = \Illuminate\Support\Facades\Bus::batch([
    new ProcessAssetChunk($assets->slice(0, 100)),
    new ProcessAssetChunk($assets->slice(100, 100)),
    new ProcessAssetChunk($assets->slice(200)),
])
->name('asset-import-' . $importId)
->allowFailures()
->then(fn(\Illuminate\Bus\Batch $batch) => ImportCompleted::dispatch($importId))
->catch(fn(\Illuminate\Bus\Batch $batch, \Throwable $e) => Log::error('Batch failed', ['batch' => $batch->id, 'error' => $e->getMessage()]))
->finally(fn(\Illuminate\Bus\Batch $batch) => Import::find($importId)?->update(['status' => 'done']))
->onQueue('imports')
->dispatch();

// Check batch progress
$batch = \Illuminate\Support\Facades\Bus::findBatch($batchId);
$batch->progress(); // 0-100
$batch->failedJobIds;
```

## Preventing Overlapping Jobs

```php
#[Connection('redis')]
#[Queue('reports')]
#[Tries(1)]
#[WithoutOverlapping(10)] // 10s lock expiry
final class GenerateInventoryReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function middleware(): array
    {
        return [new \Illuminate\Queue\Middleware\WithoutOverlapping($this->reportType)];
    }
}
```

## Rate-Limited Jobs

```php
use Illuminate\Queue\Middleware\RateLimited;

public function middleware(): array
{
    return [new RateLimited('notifications')];
}

// Define the rate limiter
RateLimiter::for('notifications', fn() => Limit::perMinute(100));
```

## Horizon Configuration

```php
// config/horizon.php
'environments' => [
    'production' => [
        'supervisor-default' => [
            'connection' => 'redis',
            'queue'      => ['high', 'default', 'notifications', 'imports', 'reports'],
            'balance'    => 'auto',
            'minProcesses' => 2,
            'maxProcesses' => 10,
            'tries'      => 3,
            'timeout'    => 60,
        ],
    ],
    'local' => [
        'supervisor-default' => [
            'connection' => 'redis',
            'queue'      => ['high', 'default', 'notifications'],
            'balance'    => 'simple',
            'processes'  => 2,
            'tries'      => 1,
        ],
    ],
],
```

## Queue Priority

| Queue | Use For | Workers |
|-------|---------|---------|
| `high` | User-facing, real-time (OTP, payment confirm) | 4+ |
| `default` | Standard async ops (create/update events) | 2-4 |
| `notifications` | Email, push, SMS | 2 |
| `imports` | File processing, bulk imports | 1-2 |
| `reports` | Report generation, exports | 1 |

## Failed Job Handling

```php
// Retry failed jobs
php artisan queue:retry all
php artisan queue:retry {id}

// Forget failed jobs
php artisan queue:forget {id}
php artisan queue:flush

// Check failed jobs table
php artisan queue:failed
```

## Constraints

### MUST DO
- Use PHP Attributes (`#[Connection]`, `#[Queue]`, `#[Tries]`, `#[Timeout]`) on every job
- Implement `failed(\Throwable $e)` on every job — always log failures
- Make jobs idempotent — safe to run multiple times
- Use `SerializesModels` — never store full model data manually
- Set `retryUntil()` on time-sensitive jobs
- Use Horizon in production — never bare `queue:work`
- Separate queues by priority and type

### MUST NOT DO
- Dispatch jobs inside a `DB::transaction()` — transaction may roll back after dispatch
- Store sensitive data (passwords, tokens) in job constructor
- Use `Tries(0)` — always set a retry limit
- Throw exceptions inside `failed()` — it must never fail itself
- Skip idempotency — duplicate dispatches must be safe
