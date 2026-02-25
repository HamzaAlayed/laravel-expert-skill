# CQRS & Event Sourcing in Laravel

## CQRS Overview

Separate the **write path** (commands that mutate state) from the **read path** (queries that return data). Read models are optimized for display; write models enforce invariants.

```
Command → CommandHandler → Aggregate → Repository (write model)
                                     ↓ domain event
                              Projector → Read Model Table
Query  → QueryHandler  → Read Model Table → ReadDTO
```

## Command & Command Handler

```php
// Application/Command/PlaceOrder.php
final readonly class PlaceOrder
{
    public function __construct(
        public string $orderId,
        public string $customerId,
        public array  $lineItems,
    ) {}
}

// Application/Command/PlaceOrderHandler.php
final class PlaceOrderHandler
{
    public function __construct(
        private readonly OrderRepository $orders,
    ) {}

    public function handle(PlaceOrder $command): void
    {
        $order = Order::place(
            OrderId::fromString($command->orderId),
            CustomerId::fromString($command->customerId),
            LineItemCollection::fromArray($command->lineItems),
        );
        $this->orders->save($order);
    }
}
```

## Simple Command Bus (no package required)

```php
// App\Bus\CommandBus.php
final class CommandBus
{
    private array $handlers = [];

    public function __construct(private readonly Container $container) {}

    public function register(string $command, string $handler): void
    {
        $this->handlers[$command] = $handler;
    }

    public function dispatch(object $command): void
    {
        $handler = $this->container->make(
            $this->handlers[$command::class]
                ?? throw new \LogicException("No handler for {$command::class}")
        );
        $handler->handle($command);
    }
}
```

Register in `AppServiceProvider`:
```php
$bus = $this->app->make(CommandBus::class);
$bus->register(PlaceOrder::class, PlaceOrderHandler::class);
```

## Query Handler & Read DTO

```php
// Application/Query/GetOrderSummaryHandler.php
final class GetOrderSummaryHandler
{
    public function __construct(
        private readonly OrderSummaryRepository $readModel,
    ) {}

    public function handle(GetOrderSummary $query): OrderSummaryDto
    {
        return $this->readModel->findById($query->orderId)
            ?? throw new OrderNotFound($query->orderId);
    }
}

// Application/Query/OrderSummaryDto.php
final readonly class OrderSummaryDto
{
    public function __construct(
        public string $orderId,
        public string $customerName,
        public int    $totalCents,
        public string $status,
        public string $placedAt,
    ) {}
}
```

## Read Model (Eloquent-backed projection table)

```php
// Infrastructure/Persistence/OrderSummaryModel.php (Eloquent)
Schema::create('order_summaries', function (Blueprint $table) {
    $table->uuid('order_id')->primary();
    $table->string('customer_name');
    $table->unsignedInteger('total_cents');
    $table->string('status');
    $table->timestamp('placed_at');
    $table->timestamps();
    $table->index(['status', 'placed_at']);
});
```

## Projector

```php
// Infrastructure/Projection/OrderSummaryProjector.php
final class OrderSummaryProjector
{
    public function onOrderWasPlaced(OrderWasPlaced $event): void
    {
        OrderSummaryModel::create([
            'order_id'      => $event->orderId->toString(),
            'customer_name' => $event->customerName,
            'total_cents'   => $event->totalCents,
            'status'        => 'placed',
            'placed_at'     => $event->occurredAt,
        ]);
    }

    public function onOrderWasShipped(OrderWasShipped $event): void
    {
        OrderSummaryModel::where('order_id', $event->orderId->toString())
            ->update(['status' => 'shipped']);
    }
}
```

Register in `EventServiceProvider`:
```php
OrderWasPlaced::class  => [OrderSummaryProjector::class . '@onOrderWasPlaced'],
OrderWasShipped::class => [OrderSummaryProjector::class . '@onOrderWasShipped'],
```

---

## Event Sourcing

### Event Store Schema

```sql
CREATE TABLE domain_events (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    aggregate_id  CHAR(36)     NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    event_type    VARCHAR(255) NOT NULL,
    payload       JSON         NOT NULL,
    metadata      JSON         NOT NULL DEFAULT ('{}'),
    version       INT UNSIGNED NOT NULL,
    occurred_at   TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    INDEX (aggregate_id, version),
    UNIQUE (aggregate_id, version)  -- optimistic concurrency
);
```

### Appending Events

```php
final class EventStore
{
    public function append(string $aggregateId, string $aggregateType, array $events, int $expectedVersion): void
    {
        DB::transaction(function () use ($aggregateId, $aggregateType, $events, $expectedVersion) {
            $currentVersion = DB::table('domain_events')
                ->where('aggregate_id', $aggregateId)
                ->max('version') ?? 0;

            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException($aggregateId, $expectedVersion, $currentVersion);
            }

            $version = $expectedVersion;
            foreach ($events as $event) {
                DB::table('domain_events')->insert([
                    'aggregate_id'   => $aggregateId,
                    'aggregate_type' => $aggregateType,
                    'event_type'     => $event::class,
                    'payload'        => json_encode($event->toArray()),
                    'version'        => ++$version,
                    'occurred_at'    => now()->format('Y-m-d H:i:s.u'),
                ]);
            }
        });
    }

    public function loadEvents(string $aggregateId, int $afterVersion = 0): array
    {
        return DB::table('domain_events')
            ->where('aggregate_id', $aggregateId)
            ->where('version', '>', $afterVersion)
            ->orderBy('version')
            ->get()
            ->map(fn ($row) => $this->deserialize($row))
            ->all();
    }
}
```

### Aggregate Reconstitution

```php
abstract class EventSourcedAggregate
{
    protected int $version = 0;
    private array $uncommittedEvents = [];

    public static function reconstitute(array $events): static
    {
        $instance = new static();
        foreach ($events as $event) {
            $instance->apply($event);
            $instance->version++;
        }
        return $instance;
    }

    protected function recordThat(object $event): void
    {
        $this->apply($event);
        $this->uncommittedEvents[] = $event;
        $this->version++;
    }

    abstract protected function apply(object $event): void;

    public function getUncommittedEvents(): array { return $this->uncommittedEvents; }
    public function getVersion(): int { return $this->version; }
}
```

### Snapshot Strategy

**When to snapshot:** When replaying 500+ events becomes measurable latency (>50ms).

```sql
CREATE TABLE aggregate_snapshots (
    aggregate_id   CHAR(36) PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    state          JSON NOT NULL,
    version        INT UNSIGNED NOT NULL,
    created_at     TIMESTAMP NOT NULL
);
```

**Reconstitution with snapshot:**
```php
public function load(string $aggregateId): MyAggregate
{
    $snapshot = DB::table('aggregate_snapshots')
        ->where('aggregate_id', $aggregateId)
        ->first();

    $afterVersion = 0;
    $aggregate = new MyAggregate();

    if ($snapshot) {
        $aggregate = MyAggregate::fromSnapshot(json_decode($snapshot->state, true));
        $afterVersion = $snapshot->version;
    }

    $events = $this->eventStore->loadEvents($aggregateId, $afterVersion);
    foreach ($events as $event) {
        $aggregate->apply($event);
    }

    return $aggregate;
}
```

## Eventual Consistency: UI Handling

After a write command, the read model may lag by milliseconds.

```php
// Controller: return 202 Accepted with polling URL
return response()->json([
    'status'     => 'accepted',
    'poll_url'   => route('orders.status', $orderId),
    'retry_after' => 1,
], 202);
```

For synchronous projectors (same request/transaction): acceptable for low-throughput writes where lag is unacceptable. Trade-off: slower writes, possible projection failure blocking command.

## Testing CQRS

```php
// Command handler test
it('places an order', function () {
    $handler = new PlaceOrderHandler(new InMemoryOrderRepository());
    $handler->handle(new PlaceOrder('order-1', 'customer-1', [
        ['sku' => 'ABC', 'qty' => 2, 'price_cents' => 1000],
    ]));

    $repo = app(OrderRepository::class);
    $order = $repo->findById(OrderId::fromString('order-1'));
    expect($order->status())->toBe(OrderStatus::Placed);
});

// Projector test
it('projects order summary on placement', function () {
    $projector = new OrderSummaryProjector();
    $projector->onOrderWasPlaced(new OrderWasPlaced(
        orderId: OrderId::fromString('order-1'),
        customerName: 'Alice',
        totalCents: 5000,
        occurredAt: now(),
    ));

    $this->assertDatabaseHas('order_summaries', [
        'order_id' => 'order-1',
        'status'   => 'placed',
    ]);
});
```

## When CQRS Is Overkill

Skip CQRS when:
- The read and write shapes are nearly identical
- The domain has fewer than 3-4 aggregates
- A single developer owns the whole module
- Traffic is low and query performance is not a bottleneck

Start with standard repository + Eloquent. Introduce CQRS when the read model needs to diverge significantly from the write model or when read throughput becomes an order of magnitude higher.
