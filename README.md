# Advanced Laravel Horizon Skill

An AI agent skill that teaches coding assistants how to design, configure, and monitor production-grade queue infrastructure with Laravel Horizon.

## What This Skill Does

This skill provides procedural knowledge for AI agents working with Laravel Horizon and Redis queues. It covers advanced supervisor design, auto-scaling strategies, failed job handling, and production monitoring — going well beyond basic queue setup.

### Topics Covered

- **Supervisor Configuration** — Multi-supervisor architecture separated by workload type (high-priority, default, long-running)
- **Balancing Strategies** — `auto` with `time` strategy for variable traffic, `simple` for steady workloads
- **Job Design** — Progressive backoff arrays, `failed()` methods, `retryUntil()`, timeouts
- **Batch Processing** — `Bus::batch()` with then/catch callbacks
- **Rate Limiting** — `Redis::throttle()` for API rate limits, `Redis::funnel()` for concurrency limits
- **Job Chaining** — Sequential execution with `Bus::chain()`
- **Unique Jobs** — `ShouldBeUnique` with custom keys and TTL
- **Monitoring** — Horizon dashboard access control, metrics API, wait time alerts
- **Health Checks** — Queue size monitoring, Redis connection health, scheduled snapshots
- **Production Setup** — Supervisor for Horizon, dedicated Redis connection, memory optimization
- **Redis Tuning** — `noeviction` policy, AOF persistence, connection limits, dangerous command lockdown

## Installation

### Via skills CLI (skills.sh)

```bash
npx skills add neuraliq/laravel-horizon-advanced
```

### Via Laravel Boost (skills.laravel.cloud)

```bash
php artisan boost:add-skill neuraliq/laravel-horizon-advanced
```

### Manual

Copy the `skills/SKILL.md` file into your project's `.cursor/skills/`, `.claude/skills/`, or equivalent agent skills directory.

## Compatibility

| Agent | Supported |
|-------|-----------|
| Claude Code | Yes |
| Cursor | Yes |
| Windsurf | Yes |
| GitHub Copilot | Yes |

## Requirements

- Laravel 11+
- Laravel Horizon
- Redis server
- Supervisor (for production)

## File Structure

```
├── skills/
│   └── SKILL.md              # Main skill definition
├── references/
│   ├── monitoring.md         # Dashboard, metrics, alerts, health checks
│   └── production-setup.md   # Supervisor, Redis config, rate limiting, chaining
└── README.md
```

## Key Rules

1. Always use `auto` balancing with `time` strategy for variable workloads
2. Always set `maxTime` and `maxJobs` to prevent zombie workers and memory leaks
3. Always use progressive backoff arrays — `[30, 120, 600]` prevents thundering herd
4. Always separate supervisors by workload type
5. Always implement `failed()` on critical jobs
6. Always run `horizon:terminate` in deployment scripts

## License

MIT
