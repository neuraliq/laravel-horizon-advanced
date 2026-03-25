# Production Setup & Redis Tuning

## Supervisor for Horizon

Horizon itself is the supervisor manager. You only need one Supervisor process for Horizon.

```ini
[program:horizon]
process_name=%(program_name)s
command=php /var/www/html/artisan horizon
directory=/var/www/html
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/supervisor/horizon.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=5
stopwaitsecs=3600
stopsignal=QUIT
```

**`stopwaitsecs=3600`** is critical — gives long-running jobs time to finish during deploys.

## Deployment Integration

```bash
#!/bin/bash
# deploy.sh

# ... git pull, composer install, migrations, etc.

# Terminate Horizon gracefully — finishes current jobs then exits
php artisan horizon:terminate

# Supervisor will auto-restart Horizon with new code
# Wait for Horizon to fully start
sleep 5

# Verify Horizon is running
php artisan horizon:status
```

### Laravel Forge / Envoyer

```bash
# In deployment hooks (after "Activate New Release"):
cd /home/forge/example.com
php artisan horizon:terminate
```

## Redis Configuration for Queues

### Dedicated Redis Connection

```php
// config/database.php
'redis' => [
    'queue' => [
        'url' => env('REDIS_QUEUE_URL'),
        'host' => env('REDIS_QUEUE_HOST', '127.0.0.1'),
        'password' => env('REDIS_QUEUE_PASSWORD'),
        'port' => env('REDIS_QUEUE_PORT', '6379'),
        'database' => env('REDIS_QUEUE_DB', '1'), // Separate DB from cache
    ],
],
```

```php
// config/queue.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'queue', // Use dedicated connection
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => 90,
    'block_for' => 5,
],
```

### Redis Memory Optimization

```conf
# /etc/redis/redis.conf (or redis.conf for your queue instance)

# Set max memory — prevent OOM kills
maxmemory 512mb

# IMPORTANT: never use allkeys-lru for queue Redis
# Queue data must NOT be evicted
maxmemory-policy noeviction

# Persistence — AOF is safer for queues
appendonly yes
appendfsync everysec

# Connection limits
maxclients 1000
timeout 300

# Disable dangerous commands in production
rename-command FLUSHALL ""
rename-command FLUSHDB ""
```

### Redis Memory Monitoring

```bash
# Check Redis memory usage
redis-cli info memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# Check queue sizes
redis-cli llen queues:critical
redis-cli llen queues:default
redis-cli llen queues:notifications

# Monitor Redis operations in real-time (careful in production)
redis-cli monitor | head -100
```

## Rate Limiting Jobs

```php
use Illuminate\Support\Facades\Redis;

class SendApiRequest implements ShouldQueue
{
    public function handle(): void
    {
        // Max 30 requests per minute to external API
        Redis::throttle('external-api')
            ->allow(30)
            ->every(60)
            ->then(function () {
                // Job logic here
                Http::post('https://api.example.com/endpoint', $this->data);
            }, function () {
                // Rate limited — release back to queue
                $this->release(30); // Retry in 30 seconds
            });
    }
}

// Concurrency limiting (max N jobs running simultaneously)
class ProcessReport implements ShouldQueue
{
    public function handle(): void
    {
        Redis::funnel('report-processing')
            ->limit(3) // Max 3 concurrent report jobs
            ->then(function () {
                // Process report
            }, function () {
                $this->release(10);
            });
    }
}
```

## Job Chaining

```php
use Illuminate\Support\Facades\Bus;

// Sequential chain — each job runs after the previous succeeds
Bus::chain([
    new ValidateImportFile($file),
    new ParseImportRows($file),
    new ProcessImportData($file),
    new SendImportCompleteNotification($user),
])->onQueue('imports')->dispatch();
```

## Unique Jobs

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class SyncUserData implements ShouldQueue, ShouldBeUnique
{
    public function __construct(public int $userId) {}

    // Unique for 1 hour
    public int $uniqueFor = 3600;

    // Unique key based on user
    public function uniqueId(): string
    {
        return (string) $this->userId;
    }
}
```
