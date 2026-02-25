# Performance & Laravel Octane

## Octane: Driver Selection

| Driver | Best For | Trade-offs |
|---|---|---|
| **Swoole** | Highest throughput, coroutines, async tasks | Requires `ext-swoole`; most mature |
| **FrankenPHP** | Zero external dependency, HTTP/3, easy Docker | Newer, smaller ecosystem |
| **RoadRunner** | Go-based, works without PHP extensions | Separate binary to manage |

**Install:**
```bash
composer require laravel/octane
php artisan octane:install  # choose driver
php artisan octane:start --workers=8 --task-workers=4
```

## Request Lifecycle: FPM vs. Octane

```
FPM (per request):
  Boot framework → Handle request → Destroy everything → Repeat

Octane (persistent worker):
  Boot framework ONCE → Handle request → Reset state → Handle next request
                                         ↑
                          Singletons persist across requests
```

**Implication:** Any state accumulated in singletons leaks into the next request.

## Octane-Safe Service Provider Checklist

| Pattern | Safe? | Fix |
|---|---|---|
| `$this->app->bind(Foo::class, ...)` | ✅ New instance per request | — |
| `$this->app->singleton(Foo::class, ...)` with no mutable state | ✅ | — |
| `$this->app->singleton(...)` with request data in properties | ❌ Leaks | Use `bind` or reset in `octane:tick` |
| `static $cache = []` inside a class | ❌ Grows unbounded | Remove or scope to request |
| Event listeners registered inside `handle()` | ❌ Stacks up | Register in `register()` or `boot()` only |
| Eloquent global scopes added per request | ❌ Compounds | Apply in model definition |

**Flush singletons between requests:**
```php
// AppServiceProvider
public function boot(): void
{
    Octane::flushing(function () {
        $this->app->forgetInstance(MyStatefulService::class);
    });
}
```

## Memory Leak Patterns

```php
// ❌ BAD: static cache grows with every request served by this worker
class GeoService
{
    private static array $cache = [];
    public function country(string $ip): string
    {
        return self::$cache[$ip] ??= $this->lookup($ip);
    }
}

// ✅ GOOD: bound per-request
class GeoService
{
    private array $cache = [];  // instance property, new instance each request
}
```

```php
// ❌ BAD: listener stacks up each request
public function handle(Request $request, Closure $next): mixed
{
    DB::listen(fn ($query) => Log::debug($query->sql)); // adds new listener every request
    return $next($request);
}

// ✅ GOOD: register listener in ServiceProvider::boot()
```

## Concurrency Within a Single Request

```php
use Laravel\Octane\Facades\Octane;

[$users, $orders] = Octane::concurrently([
    fn () => User::where('active', true)->count(),
    fn () => Order::whereDate('created_at', today())->count(),
]);
```

Runs tasks in parallel coroutines (Swoole) or Go routines (RoadRunner). Not available in FPM.

## Response Time Budgeting

Target p95 ≤ 200ms for API responses. Allocate budget:

| Layer | Budget | Tool to Measure |
|---|---|---|
| DB queries | 50ms | Telescope, `DB::listen` |
| Cache reads | 5ms | Redis `SLOWLOG` |
| Business logic | 20ms | Xdebug profiler / Blackfire |
| Serialization | 10ms | `microtime()` wrapping |
| Network (Redis/DB) | 15ms | Pingdom / New Relic |

## Database Optimization Beyond N+1

**Covering index (index-only scan):**
```sql
-- Query: SELECT id, status FROM orders WHERE customer_id = ? AND status = 'placed'
ALTER TABLE orders ADD INDEX idx_customer_status_id (customer_id, status, id);
-- All columns in WHERE + SELECT are in the index — no row lookup needed
```

**Partial index (PostgreSQL):**
```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status = 'pending';
-- Only indexes rows where status = 'pending'; smaller, faster for that filter
```

**EXPLAIN ANALYZE (PostgreSQL):**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 'abc' AND status = 'placed';
```
Look for: `Seq Scan` on large tables (add index), `Nested Loop` on large result sets (add covering index), high `Buffers: shared hit` ratio (good cache usage).

**Cursor pagination for large exports:**
```php
// ❌ BAD: loads all 500k rows into memory
Order::all()->each(fn ($o) => $export->add($o));

// ✅ GOOD: streams in chunks of 1000
Order::lazyById(1000)->each(fn ($o) => $export->add($o));
```

## Profiling Tools

| Tool | When to Use | Key Metric |
|---|---|---|
| **Telescope** | Development — query inspection, job monitoring | Slow queries > 100ms |
| **Debugbar** | Local — timeline per request | Total request time breakdown |
| **Clockwork** | Local/staging — detailed timeline | DB, cache, queue time per request |
| **Blackfire** | Production profiling | Call graph, I/O wait, memory |
| **Xdebug + KCachegrind** | CPU-bound optimization | Function-level CPU cycles |

## Load Testing

```bash
# k6 script: ramp to 500 VUs over 2 minutes
k6 run --vus 500 --duration 2m script.js
```

```javascript
// k6 script
import http from 'k6/http';
import { check } from 'k6';

export let options = {
    stages: [
        { duration: '30s', target: 100 },
        { duration: '1m',  target: 500 },
        { duration: '30s', target: 0   },
    ],
    thresholds: {
        http_req_duration: ['p(95)<200'],  // 95% of requests < 200ms
        http_req_failed:   ['rate<0.01'],  // < 1% error rate
    },
};

export default function () {
    const res = http.get('https://api.example.com/orders');
    check(res, { 'status 200': (r) => r.status === 200 });
}
```

**Metrics to capture:**
- `p50`, `p95`, `p99` latency
- Requests/second (throughput)
- Error rate
- Worker CPU and memory under load
- DB connection pool saturation (`SHOW STATUS LIKE 'Threads_connected'`)
