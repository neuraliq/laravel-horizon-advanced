---
name: laravel-horizon-advanced
description: Advanced Laravel Horizon configuration, supervisor optimization, and queue architecture. Use when configuring Horizon supervisors, tuning worker counts, designing multi-queue architectures, handling failed jobs, or setting up production queue infrastructure.
compatible_agents:
  - Claude Code
  - Cursor
  - Windsurf
  - Copilot
tags:
  - laravel
  - horizon
  - queues
  - redis
  - workers
  - jobs
  - production
  - monitoring
---

# Advanced Laravel Horizon

Horizon provides a beautiful dashboard and code-driven configuration for Laravel Redis queues. This skill covers production-grade patterns for supervisor design, worker optimization, and queue architecture.

## Context

You are working with Laravel applications that need robust background job processing. This skill covers supervisor configuration, worker tuning, queue priority design, and production monitoring with Horizon.

## Rules

1. Always use `auto` balancing with `time` strategy for variable workloads — scales workers based on actual wait time
2. Always set `maxTime` to prevent zombie workers — 3600 (1 hour) is a safe default
3. Always set `maxJobs` to recycle workers periodically — prevents gradual memory leaks
4. Always use progressive `backoff` arrays for retry-able jobs — `[30, 120, 600]` prevents thundering herd
5. Always separate supervisors by workload type — fast user-facing vs slow background processing
6. Never run more workers than your Redis connection can handle — watch `connected_clients`
7. Always set job-level `$timeout` shorter than supervisor timeout — prevents unexpected kills
8. Always implement `failed()` method on critical jobs — silent failures are invisible failures
9. Use `nice` values to prioritize user-facing supervisors — lower number = higher CPU priority
10. Always run `php artisan horizon:terminate` in deployment scripts — ensures workers pick up new code

## Examples

### Supervisor Configuration

```php
// config/horizon.php
'environments' => [
    'production' => [
        'high-priority' => [
            'connection' => 'redis',
            'queue' => ['critical', 'high'],
            'balance' => 'auto',
            'autoScalingStrategy' => 'time',
            'minProcesses' => 2,
            'maxProcesses' => 10,
            'balanceMaxShift' => 3,
            'balanceCooldown' => 3,
            'tries' => 3,
            'timeout' => 30,
            'maxTime' => 3600,
            'maxJobs' => 1000,
            'memory' => 128,
        ],

        'default' => [
            'connection' => 'redis',
            'queue' => ['default', 'notifications'],
            'balance' => 'auto',
            'minProcesses' => 2,
            'maxProcesses' => 8,
            'tries' => 3,
            'timeout' => 60,
            'memory' => 256,
        ],

        'long-running' => [
            'connection' => 'redis',
            'queue' => ['exports', 'reports', 'imports'],
            'balance' => 'simple',
            'minProcesses' => 1,
            'maxProcesses' => 4,
            'tries' => 1,
            'timeout' => 900,
            'memory' => 512,
        ],
    ],
],
```

### Balancing Strategies

```php
// Dynamic scaling (recommended for variable traffic)
'balance' => 'auto',
'autoScalingStrategy' => 'time',
'minProcesses' => 2,
'maxProcesses' => 10,

// Equal distribution (for steady workloads)
'balance' => 'simple',

// Fixed count (for development)
'balance' => 'false',
'processes' => 3,
```

### Job Dispatch with Priority

```php
// Dispatch to specific queue
SendWelcomeEmail::dispatch($user)->onQueue('notifications');
ProcessReport::dispatch($report)->onQueue('reports');

// Or set on the job class
class ChargeCustomer implements ShouldQueue
{
    public string $queue = 'critical';
    public int $tries = 3;
    public int $timeout = 30;
    public array $backoff = [30, 120, 600];
}
```

### Failed Job Handling

```php
class ProcessPayment implements ShouldQueue
{
    public int $tries = 3;
    public array $backoff = [30, 120, 600];

    public function failed(\Throwable $exception): void
    {
        Notification::route('slack', config('services.slack.webhook'))
            ->notify(new PaymentFailedNotification($this->order, $exception));
    }

    public function retryUntil(): DateTime
    {
        return now()->addHours(2);
    }
}
```

### Batch Processing

```php
$batch = Bus::batch([
    new ProcessRow($rows->slice(0, 100)),
    new ProcessRow($rows->slice(100, 100)),
])
    ->then(fn (Batch $batch) => ExportComplete::dispatch($export))
    ->catch(fn (Batch $batch, \Throwable $e) => Log::error('Batch failed'))
    ->onQueue('exports')
    ->name('export-' . now()->format('Y-m-d'))
    ->dispatch();
```

## Anti-Patterns

- Single supervisor for all queues — different workloads need different configs
- No retry limits — failed jobs consume resources forever
- Ignoring backoff — retry storm when issues occur
- Running Horizon in same process as web server — use Supervisor
- Not terminating Horizon on deploy — old code keeps running

## References

- [Laravel Horizon](https://laravel.com/docs/horizon)
- [Redis Queue Configuration](https://laravel.com/docs/queues#redis)
- [Supervisor Configuration](https://laravel.com/docs/queues#supervisor-configuration)

## When to Use This Skill

Use this skill when:
- Configuring Horizon supervisors
- Tuning worker counts
- Designing multi-queue architectures
- Optimizing Redis memory for queues
- Handling failed jobs
- Setting up balancing strategies
- Monitoring queue health in production
