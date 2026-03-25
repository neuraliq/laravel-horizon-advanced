# Monitoring & Alerting

## Horizon Dashboard

```php
// app/Providers/HorizonServiceProvider.php
protected function gate(): void
{
    Gate::define('viewHorizon', function ($user = null) {
        // Restrict access in production
        if (app()->environment('local')) {
            return true;
        }

        return $user?->is_admin === true;
    });
}
```

Dashboard URL: `/horizon`

## Horizon Metrics API

```php
use Laravel\Horizon\Contracts\MetricsRepository;

$metrics = app(MetricsRepository::class);

// Queue metrics — check Horizon source for available methods
// Common metrics include wait times and throughput
$metrics->snapshot();
```

## Notification Channels

```php
// config/horizon.php
'waits' => [
    // Alert when any queue wait exceeds threshold (seconds)
    'redis:critical' => 10,    // Critical: alert after 10s
    'redis:high' => 30,        // High: alert after 30s
    'redis:default' => 60,     // Default: alert after 1 min
    'redis:exports' => 300,    // Exports: alert after 5 min
],
```

```php
// Custom notification for long waits
// app/Notifications/HorizonLongWait.php
use Laravel\Horizon\Events\LongWaitDetected;

class HorizonLongWait extends Notification
{
    public function __construct(public LongWaitDetected $event) {}

    public function via($notifiable): array
    {
        return ['slack'];
    }

    public function toSlack($notifiable): SlackMessage
    {
        return (new SlackMessage)
            ->error()
            ->content("Queue `{$this->event->queue}` wait time: {$this->event->seconds}s");
    }
}
```

## Health Check Endpoint

```php
Route::get('/api/health/queues', function () {
    $redis = app('redis');
    $queues = ['critical', 'high', 'default', 'notifications', 'exports'];

    $status = collect($queues)->mapWithKeys(function ($queue) use ($redis) {
        $size = $redis->llen("queues:{$queue}");
        $delayed = $redis->zcard("queues:{$queue}:delayed");
        return [$queue => [
            'pending' => $size,
            'delayed' => $delayed,
            'healthy' => $size < 1000,
        ]];
    });

    $allHealthy = $status->every(fn ($q) => $q['healthy']);

    return response()->json([
        'status' => $allHealthy ? 'ok' : 'degraded',
        'queues' => $status,
        'horizon' => [
            'status' => app('horizon.status')->current(),
            'master_pid' => cache('horizon:master'),
        ],
    ], $allHealthy ? 200 : 503);
});
```

## Failed Jobs Dashboard Commands

```bash
# View failed jobs
php artisan queue:failed

# Retry specific job
php artisan queue:retry <job-id>

# Retry all failed jobs
php artisan queue:retry all

# Flush old failed jobs
php artisan queue:flush --hours=48

# Prune Horizon data (keep 24 hours)
php artisan horizon:clear
php artisan horizon:purge
```

## Scheduled Monitoring

```php
// app/Console/Kernel.php
$schedule->command('horizon:snapshot')->everyFiveMinutes();

// Custom queue size monitoring
$schedule->call(function () {
    $redis = app('redis');
    $criticalSize = $redis->llen('queues:critical');

    if ($criticalSize > 100) {
        Notification::route('slack', config('services.slack.alerts'))
            ->notify(new QueueBacklogAlert('critical', $criticalSize));
    }
})->everyMinute();
```
