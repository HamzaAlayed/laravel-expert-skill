# Scale & Infrastructure in Laravel

## Stateless Application Design

Every web worker must be interchangeable. Eliminate request-local state.

| Concern | Stateful (bad) | Stateless (good) |
|---|---|---|
| Sessions | File / cookie sessions | `SESSION_DRIVER=redis` |
| File uploads | `storage/app` on local disk | S3 / `FILESYSTEM_DISK=s3` |
| File serving | Direct filesystem read | Signed S3 URLs (`Storage::temporaryUrl`) |
| Cache | `CACHE_DRIVER=file` | `CACHE_DRIVER=redis` |
| Queue | `QUEUE_CONNECTION=sync` | `QUEUE_CONNECTION=redis` |

## Horizontal Web Server Scaling

**Health check endpoint** (required by load balancers):
```php
// routes/api.php
Route::get('/health', function () {
    DB::select('SELECT 1');            // DB reachable
    Redis::ping();                     // Cache reachable
    return response()->json(['status' => 'ok']);
});
```

**Load balancer configuration:**
- Use **round-robin** (stateless app) — no sticky sessions needed
- Health check: `GET /health` → expect HTTP 200
- Drain timeout: 30s to allow in-flight requests to complete before instance termination

**Octane scaling:** Start multiple Octane processes on each server; put Nginx as reverse proxy in front.

```nginx
upstream octane {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    keepalive 64;
}
```

## Queue Worker Scaling

**Horizon balance modes:**

| Mode | Behavior | Use When |
|---|---|---|
| `auto` | Distributes workers based on queue depth | Default, mixed queues |
| `simple` | Equal workers per queue | Queues have equal priority |
| `false` | No auto-balancing | Manual control needed |

**Multi-supervisor topology (horizon.php):**
```php
'supervisors' => [
    'high-priority' => [
        'connection' => 'redis',
        'queue'      => ['critical', 'high'],
        'balance'    => 'auto',
        'maxProcesses' => 20,
        'minProcesses' => 2,
    ],
    'default' => [
        'connection' => 'redis',
        'queue'      => ['default'],
        'balance'    => 'auto',
        'maxProcesses' => 10,
        'minProcesses' => 1,
    ],
    'bulk' => [
        'connection' => 'redis',
        'queue'      => ['exports', 'imports'],
        'balance'    => 'simple',
        'maxProcesses' => 5,
        'timeout'    => 600,
    ],
],
```

**Scale on queue depth:** Use a cron or CloudWatch alarm that checks `horizon:queue-size` and adjusts ECS/K8s replicas.

## Database Scaling

### Read Replicas

```php
// config/database.php
'mysql' => [
    'read'  => ['host' => [env('DB_READ_HOST')]],
    'write' => ['host' => [env('DB_WRITE_HOST')]],
    'sticky' => true,  // reads in same request go to write after a write
    // ...
],
```

```php
// Force write connection for reads requiring fresh data
$order = DB::connection('mysql::write')
    ->table('orders')
    ->find($id);
```

**Replica lag:** Typical MySQL async replication lag is 10-100ms. Never read from replica immediately after a write in the same user flow unless `sticky` is enabled.

### Connection Pooling

PHP does not have persistent connection pools natively. Use **PgBouncer** (PostgreSQL) or **ProxySQL** (MySQL) as a connection pool in front of the database.

```
Workers (100) → PgBouncer (pool: 20) → PostgreSQL (max_connections: 100)
```

ProxySQL can also route `SELECT` statements to read replicas automatically via query rules.

### Table Partitioning (PostgreSQL / MySQL 8+)

**Range partitioning for time-series data:**
```sql
CREATE TABLE events (
    id         BIGINT,
    occurred_at TIMESTAMP NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

Query planner only scans relevant partitions. Drop old partitions (O(1)) instead of DELETE.

### Materialized Views (PostgreSQL)

```sql
CREATE MATERIALIZED VIEW order_revenue_by_month AS
    SELECT DATE_TRUNC('month', created_at) AS month,
           SUM(total_cents) AS revenue_cents,
           COUNT(*) AS order_count
    FROM orders
    GROUP BY 1
WITH DATA;

CREATE UNIQUE INDEX ON order_revenue_by_month (month);
```

Refresh via Laravel job:
```php
DB::statement('REFRESH MATERIALIZED VIEW CONCURRENTLY order_revenue_by_month');
```

`CONCURRENTLY` allows reads during refresh (requires unique index).

### Sharding Signals

Delay sharding until:
- Single DB at max IOPS/connections after pooling + replicas
- Table sizes > 500GB with slow `ALTER TABLE`
- Multi-region write requirements

Sharding adds: cross-shard queries impossible, foreign keys across shards impossible, operational complexity. Consider multi-tenancy at schema level (see below) before sharding.

## Redis at Scale

### Redis Cluster

```php
// config/database.php
'redis' => [
    'clusters' => [
        'default' => [
            ['host' => '10.0.1.1', 'port' => 7001],
            ['host' => '10.0.1.2', 'port' => 7002],
            ['host' => '10.0.1.3', 'port' => 7003],
        ],
    ],
    'options' => ['cluster' => 'redis'],
],
```

**Hash tags** — force related keys to the same slot (enables multi-key operations):
```php
Cache::put('{user:123}:profile', $profile);
Cache::put('{user:123}:permissions', $permissions);
// Both on same node → can pipeline together
```

**Limitation:** `Cache::tags()` uses multi-key operations; requires all tag keys on the same node. Use hash tags on the tag name to ensure this.

### Redis Sentinel vs. Cluster

| | Sentinel | Cluster |
|---|---|---|
| Primary purpose | High availability (failover) | Horizontal sharding |
| Scaling | Vertical only | Horizontal (add nodes) |
| Multi-key ops | Supported | Requires hash tags |
| Use when | Single shard, need HA | > single-node memory limit |

### Eviction Policies

| Policy | Behavior | Use When |
|---|---|---|
| `noeviction` | Reject writes when full | Never for caches |
| `allkeys-lru` | Evict least-recently-used from all keys | General cache |
| `volatile-lru` | Evict LRU from keys with TTL | Mixed cache + durable |
| `allkeys-lfu` | Evict least-frequently-used | Skewed access patterns |

Set in `redis.conf`:
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

### Pub/Sub at Scale

```php
// Broadcaster (write side)
broadcast(new OrderShipped($order))->toOthers();

// Redis pub/sub fan-out → Laravel Echo → WebSocket server (Reverb / Soketi)
```

For >10k concurrent connections, run multiple WebSocket server replicas with Redis pub/sub as the shared bus between them.

## Multi-Tenancy Patterns

| Pattern | Isolation | Complexity | Migration Difficulty |
|---|---|---|---|
| Database per tenant | Highest | High | Low (separate DBs) |
| Schema per tenant (PostgreSQL) | High | Medium | Medium |
| Row-level (shared table + `tenant_id`) | Lowest | Low | High (global scopes) |

**Row-level with global scope:**
```php
// Eloquent model
protected static function booted(): void
{
    static::addGlobalScope('tenant', function (Builder $query) {
        $query->where('tenant_id', app(TenantContext::class)->id());
    });
}
```

Risk: missing `withoutGlobalScope('tenant')` in admin queries exposes wrong tenant data.

## Deployment Strategies

### Zero-Downtime Migrations

Never run destructive migrations in the same deploy as the code that removes the old column.

```
Deploy 1: Add new column (nullable, backward compatible)
Deploy 2: Backfill new column, update application to write both columns
Deploy 3: Update application to read only new column
Deploy 4: Drop old column
```

### Blue/Green with Octane

```bash
# Start new version on port 8001 while 8000 still serves
php artisan octane:start --port=8001

# Run migrations (must be backward compatible with old code on 8000)
php artisan migrate

# Flip load balancer upstream to 8001

# Gracefully stop old workers
php artisan octane:stop --port=8000
```

### Rolling Restarts (Kubernetes)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Octane reload (no downtime, same process):
```bash
php artisan octane:reload  # sends SIGUSR1 to workers
```

## Observability at Scale

**Structured logging:**
```php
Log::info('order.placed', [
    'order_id'    => $orderId,
    'customer_id' => $customerId,
    'total_cents' => $totalCents,
    'duration_ms' => $durationMs,
]);
```

**Distributed tracing (OpenTelemetry):**
```bash
composer require open-telemetry/opentelemetry-auto-laravel
```

Traces propagate `traceparent` header through HTTP calls and queue jobs. View in Jaeger, Tempo, or Datadog APM.

**Key metrics to collect per service:**
- Request rate, error rate, latency (RED metrics)
- DB query count per request, slow query rate
- Cache hit rate
- Queue depth and job failure rate
- Worker memory per process (Octane memory growth indicator)
