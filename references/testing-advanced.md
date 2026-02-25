# Advanced Testing in Laravel

## Architecture Fitness Tests (Pest)

Enforce layer boundaries and coding conventions automatically in CI:

```php
// tests/Architecture/LayerTest.php
use Illuminate\Database\Eloquent\Model;

test('domain layer has no framework dependencies')
    ->expect('App\Domain')
    ->not->toUse(Model::class)
    ->not->toUse('Illuminate\Support\Facades')
    ->not->toUse('Illuminate\Http');

test('controllers only depend on application layer')
    ->expect('App\Http\Controllers')
    ->not->toUse('App\Domain');

test('all controllers are final')
    ->expect('App\Http\Controllers')
    ->toBeFinal();

test('no debug functions left in code')
    ->expect(['dd', 'dump', 'ray', 'var_dump'])
    ->not->toBeUsed();

test('all events implement ShouldQueue or ShouldBroadcast')
    ->expect('App\Events')
    ->toImplement(\Illuminate\Contracts\Queue\ShouldQueue::class);
```

## Mutation Testing (Infection PHP)

Mutation testing verifies your tests actually catch bugs — not just that they execute.

```bash
composer require --dev infection/infection
vendor/bin/infection --min-msi=80 --min-covered-msi=90
```

**Mutation Score Indicator (MSI):** % of code mutations caught by tests. Target > 80%.

```xml
<!-- infection.json5 -->
{
    "source": {
        "directories": ["app/Domain", "app/Application"],
        "excludes": ["app/Domain/ValueObjects/Generated"]
    },
    "minMsi": 80,
    "minCoveredMsi": 90,
    "mutators": {
        "@default": true,
        "FunctionCall": false  // disable if too slow
    }
}
```

## Parallel Testing

```bash
# Run tests across multiple processes (CPU-bound speedup)
php artisan test --parallel --processes=8

# Each process gets its own database
# config/database.php handles this automatically via TEST_TOKEN env var
```

```php
// Use RefreshDatabase trait — compatible with parallel testing
uses(RefreshDatabase::class);

// NOT DatabaseTransactions (doesn't work across processes)
```

## Feature Test Patterns

### Testing Commands

```php
it('imports orders from CSV', function () {
    Storage::fake('local');
    Storage::put('imports/orders.csv', file_get_contents(__DIR__ . '/fixtures/orders.csv'));

    $this->artisan('orders:import', ['file' => 'imports/orders.csv'])
        ->expectsOutput('Imported 3 orders')
        ->assertExitCode(0);

    expect(Order::count())->toBe(3);
});
```

### Testing Events & Listeners

```php
it('fires OrderWasPlaced when order is created', function () {
    Event::fake([OrderWasPlaced::class]);

    $handler = app(PlaceOrderHandler::class);
    $handler->handle(new PlaceOrder('order-1', 'customer-1', []));

    Event::assertDispatched(OrderWasPlaced::class, function ($event) {
        return $event->orderId->toString() === 'order-1';
    });
});

it('sends confirmation email when order is placed', function () {
    Mail::fake();

    event(new OrderWasPlaced($order));

    Mail::assertSent(OrderConfirmationMail::class, fn ($mail) =>
        $mail->hasTo($order->customer->email)
    );
});
```

### Testing Jobs & Queues

```php
it('dispatches export job when requested', function () {
    Queue::fake();

    $this->actingAs($user)
        ->postJson('/api/reports/export', ['format' => 'csv'])
        ->assertAccepted();

    Queue::assertPushed(ExportReportJob::class, fn ($job) =>
        $job->format === 'csv' && $job->userId === $user->id
    );
});

it('export job processes correctly', function () {
    Storage::fake('local');
    $job = new ExportReportJob($user->id, 'csv');
    $job->handle();

    Storage::assertExists("exports/report-{$user->id}.csv");
});
```

### Testing Notifications

```php
it('notifies customer when order ships', function () {
    Notification::fake();

    $order->ship();

    Notification::assertSentTo($order->customer, OrderShippedNotification::class,
        fn ($n) => $n->order->id === $order->id
    );
});
```

### Testing Broadcasting

```php
it('broadcasts order status update', function () {
    Event::fake([OrderStatusUpdated::class]);

    $order->update(['status' => 'shipped']);

    Event::assertDispatched(OrderStatusUpdated::class);
    // Or with broadcasting:
    // expect($order->status)->toBe('shipped');
    // Check broadcast payload via broadcastWith()
});
```

### Testing HTTP with Mocked Services

```php
it('syncs order to external warehouse', function () {
    Http::fake([
        'warehouse.api.example.com/*' => Http::response(['synced' => true], 200),
    ]);

    $syncer = app(WarehouseSyncer::class);
    $syncer->sync($order);

    Http::assertSent(fn ($request) =>
        str_contains($request->url(), 'warehouse.api.example.com') &&
        $request['order_id'] === $order->id
    );
});
```

## Browser Testing (Laravel Dusk)

```bash
composer require --dev laravel/dusk
php artisan dusk:install
php artisan dusk:make OrderCheckoutTest
```

```php
// tests/Browser/OrderCheckoutTest.php
final class OrderCheckoutTest extends DuskTestCase
{
    public function test_customer_can_checkout(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->loginAs(User::factory()->create())
                ->visit('/shop')
                ->click('@add-to-cart-btn')
                ->waitForText('Added to cart')
                ->click('@checkout-btn')
                ->waitForLocation('/checkout')
                ->type('@card-number', '4242424242424242')
                ->type('@card-expiry', '12/30')
                ->type('@card-cvc', '123')
                ->press('@place-order-btn')
                ->waitForText('Order confirmed')
                ->assertPathIs('/orders/confirmation');
        });
    }
}
```

```bash
php artisan dusk  # runs headless Chrome tests
```

## Contract Testing

Ensure API consumer expectations are met:

```php
// tests/Contracts/OrderApiContractTest.php
it('order show endpoint matches API contract', function () {
    $order = Order::factory()->withItems(3)->create();

    $response = $this->actingAs($order->customer)
        ->getJson("/api/v1/orders/{$order->id}")
        ->assertOk();

    // Assert exact contract shape
    $response->assertJsonStructure([
        'data' => [
            'id', 'status',
            'total' => ['amount', 'currency'],
            'items' => [['id', 'sku', 'quantity', 'price']],
            'links' => ['self'],
        ],
    ]);

    // Assert no extra fields leak (strict mode)
    expect(array_keys($response->json('data')))
        ->toBe(['id', 'status', 'total', 'items', 'customer', 'links']);
});
```

## Snapshot Testing

```bash
composer require --dev spatie/pest-plugin-snapshots
```

```php
it('order resource matches snapshot', function () {
    $order = Order::factory()->create();
    expect((new OrderResource($order))->toArray(request()))->toMatchSnapshot();
});
```

First run creates the snapshot. Subsequent runs compare against it. Update with `--update-snapshots`.

## Performance Assertions

```php
it('order listing query runs in under 50ms', function () {
    Order::factory(500)->create();

    $start = microtime(true);
    $this->getJson('/api/v1/orders?per_page=25')->assertOk();
    $duration = (microtime(true) - $start) * 1000;

    expect($duration)->toBeLessThan(50);
});

it('order listing makes at most 3 queries', function () {
    Order::factory(25)->create();

    DB::enableQueryLog();
    $this->getJson('/api/v1/orders?per_page=25')->assertOk();
    $queries = DB::getQueryLog();
    DB::disableQueryLog();

    expect(count($queries))->toBeLessThanOrEqual(3);
});
```

## Factory Patterns (Advanced)

```php
// Factories with states and relationships
final class OrderFactory extends Factory
{
    public function placed(): static
    {
        return $this->state(['status' => OrderStatus::Placed]);
    }

    public function shipped(): static
    {
        return $this->state(['status' => OrderStatus::Shipped])
            ->has(Shipment::factory(), 'shipment');
    }

    public function withItems(int $count = 3): static
    {
        return $this->has(OrderItem::factory()->count($count), 'items');
    }

    public function forCustomer(Customer $customer): static
    {
        return $this->state(['customer_id' => $customer->id]);
    }
}

// Usage
$order = Order::factory()->placed()->withItems(5)->forCustomer($customer)->create();
```

## Test Coverage & CI

```bash
# Generate HTML coverage report
php artisan test --coverage --min=85

# CI: fail if coverage drops below threshold
php artisan test --coverage --min=85 --parallel
```

```yaml
# GitHub Actions
- name: Run tests
  run: php artisan test --parallel --coverage --min=85
  env:
    DB_CONNECTION: sqlite
    DB_DATABASE: ':memory:'
```
