# Caching Strategies in Laravel

## Multi-Layer Cache Model

```
Request
  → L1: In-memory array (same PHP process, this request only)  ~0.01ms
  → L2: Redis (shared across all workers)                       ~1-2ms
  → L3: Database (source of truth)                             ~5-50ms
```

**L1 implementation (request-scoped):**
```php
final class ProductPriceService
{
    private array $cache = [];  // lives only for this request

    public function getPrice(string $productId): Money
    {
        return $this->cache[$productId]
            ??= Cache::remember("price:{$productId}", 300, fn () =>
                Product::findOrFail($productId)->price
            );
    }
}
```

## Laravel Cache Drivers Comparison

| Driver | TTL | Tagging | Atomic ops | Best For |
|---|---|---|---|---|
| `array` | Request-scoped | ✅ | ✅ | Tests, L1 |
| `redis` | Persistent | ✅ | ✅ | General purpose, tagging |
| `memcached` | Persistent | ❌ | ✅ | High-throughput, no tagging |
| `dynamodb` | Persistent | ❌ | Limited | Serverless / AWS-native |

## Cache-Aside Pattern

```php
// Read: check cache first, populate on miss
$orders = Cache::tags(['orders', "customer:{$customerId}"])
    ->remember("customer:{$customerId}:orders", ttl: 300, function () use ($customerId) {
        return Order::where('customer_id', $customerId)
            ->with('lineItems')
            ->get()
            ->toArray();
    });

// Write: invalidate relevant tags
Cache::tags(['orders', "customer:{$customerId}"])->flush();
```

## Cache Stampede Prevention

Multiple workers simultaneously miss a cold key and all query the DB.

```php
// ✅ Atomic lock: only one worker rebuilds; others wait
$value = Cache::lock("rebuild:orders:{$customerId}", 10)
    ->block(5, function () use ($customerId) {
        return Cache::remember("orders:{$customerId}", 300, fn () =>
            Order::where('customer_id', $customerId)->get()
        );
    });
```

**Probabilistic early expiration** (avoids synchronized misses):
```php
function probabilisticGet(string $key, int $ttl, Closure $miss): mixed
{
    $item = Cache::get($key);
    if ($item && random_int(1, 100) > (5 * (1 - ($item['remaining_ttl'] / $ttl)))) {
        return $item['value'];
    }
    $value = $miss();
    Cache::put($key, ['value' => $value, 'remaining_ttl' => $ttl], $ttl);
    return $value;
}
```

## Cache Tagging

Group related keys for bulk invalidation. Redis only (not Memcached).

```php
// Store with multiple tags
Cache::tags(['products', 'category:electronics'])->put("product:{$id}", $data, 3600);

// Invalidate all product caches when any product changes
Cache::tags(['products'])->flush();

// Invalidate only electronics category
Cache::tags(['category:electronics'])->flush();
```

**Redis tagging caveat:** Tags use a Redis SET per tag containing all member keys. On very high cardinality (millions of keys), tag invalidation becomes a slow `DEL` pipeline. Use `SCAN`-based deletion for massive key sets.

## Redis Pipelines

Batch multiple commands into one round trip:

```php
// ❌ BAD: N round trips for N keys
foreach ($userIds as $id) {
    $scores[$id] = Cache::get("score:{$id}");
}

// ✅ GOOD: 1 round trip
$scores = Redis::pipeline(function ($pipe) use ($userIds) {
    foreach ($userIds as $id) {
        $pipe->get("score:{$id}");
    }
});
// $scores is an array of results in order
```

## Redis Lua Scripting (Atomic Operations)

Use for check-and-set operations that must be atomic:

```php
// Atomically increment a counter only if below max
$script = <<<LUA
    local current = tonumber(redis.call('GET', KEYS[1])) or 0
    if current < tonumber(ARGV[1]) then
        return redis.call('INCR', KEYS[1])
    end
    return -1
LUA;

$result = Redis::eval($script, 1, "rate:user:{$userId}", $maxRequests);
// Returns new count, or -1 if already at max
```

## HTTP Response Caching

```php
// Controller: set cache headers
return response()->json($data)
    ->header('Cache-Control', 'public, max-age=300')
    ->header('ETag', md5(serialize($data)));

// Middleware: validate conditional requests
public function handle(Request $request, Closure $next): Response
{
    $response = $next($request);
    $etag = $response->headers->get('ETag');
    if ($etag && $request->header('If-None-Match') === $etag) {
        return response(null, 304);
    }
    return $response;
}
```

## Fragment Caching

Cache expensive computed parts independently:

```php
// Cache expensive aggregation separately from full response
$stats = Cache::remember("dashboard:stats:{$userId}", 60, function () use ($userId) {
    return [
        'total_orders'    => Order::where('user_id', $userId)->count(),
        'revenue_cents'   => Order::where('user_id', $userId)->sum('total_cents'),
        'avg_order_cents' => Order::where('user_id', $userId)->avg('total_cents'),
    ];
});
```

## Cache Warming

```php
// Artisan command to warm caches after deploy
php artisan cache:warm --categories --products

// Or queue jobs for warm-up
WarmProductCacheJob::dispatch()->onQueue('cache-warm');
```

## Invalidation Patterns

**Event-driven (preferred):**
```php
// Domain event listener
class InvalidateProductCache
{
    public function handle(ProductWasUpdated $event): void
    {
        Cache::tags(['products', "product:{$event->productId}"])->flush();
    }
}
```

**TTL-based:** Use short TTLs for volatile data, long TTLs + event invalidation for stable data.

**Explicit purge:** Expose a cache-clear endpoint (secured) for CDN purge webhooks.

## Monitoring Cache Effectiveness

```bash
# Redis hit/miss rate
redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses'

# Calculate hit rate
# hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses)
# Target: > 90% for hot data

# Eviction rate (non-zero means memory pressure)
redis-cli INFO stats | grep evicted_keys

# Memory usage
redis-cli INFO memory | grep used_memory_human
```

**Laravel Telescope** records cache hits/misses per request under the Cache tab.

**Target metrics:**
- Hit rate > 90% for read-heavy caches
- Zero evictions (increase `maxmemory` or reduce TTLs)
- p99 Redis GET < 2ms
