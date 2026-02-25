# Testing in Laravel

## Setup & Configuration

```bash
composer require --dev pestphp/pest pestphp/pest-plugin-laravel
php artisan pest:install
```

```php
// phpunit.xml — test database
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
<env name="CACHE_DRIVER" value="array"/>
<env name="QUEUE_CONNECTION" value="sync"/>
<env name="MAIL_MAILER" value="array"/>
```

```php
// tests/Pest.php — global helpers
uses(Tests\TestCase::class)->in('Feature');
uses(Tests\TestCase::class, RefreshDatabase::class)->in('Feature');

// Custom expectations
expect()->extend('toBeUuid', fn () => $this->toMatch('/^[0-9a-f-]{36}$/'));
```

---

## Unit Tests

Test single classes in isolation — no database, no HTTP, no framework boot.

```php
// tests/Unit/Domain/MoneyTest.php
it('adds two money values of same currency', function () {
    $a = new Money(1000, 'USD');
    $b = new Money(500, 'USD');

    expect($a->add($b)->amount)->toBe(1500)
        ->and($a->add($b)->currency)->toBe('USD');
});

it('throws when adding different currencies', function () {
    $a = new Money(1000, 'USD');
    $b = new Money(500, 'EUR');

    expect(fn () => $a->add($b))->toThrow(\DomainException::class, 'Cannot add different currencies');
});
```

---

## Feature Tests

Test the full HTTP stack — routes, controllers, middleware, database.

```php
// tests/Feature/Api/OrderTest.php
uses(RefreshDatabase::class);

it('authenticated user can create an order', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create(['price_cents' => 1000, 'stock' => 10]);

    $this->actingAs($user)
        ->postJson('/api/v1/orders', [
            'items' => [['sku' => $product->sku, 'quantity' => 2]],
        ])
        ->assertCreated()
        ->assertJsonStructure(['data' => ['id', 'status', 'total']]);

    $this->assertDatabaseHas('orders', ['customer_id' => $user->id, 'status' => 'placed']);
});

it('guest cannot create an order', function () {
    $this->postJson('/api/v1/orders', [])->assertUnauthorized();
});

it('returns 422 when items are missing', function () {
    $this->actingAs(User::factory()->create())
        ->postJson('/api/v1/orders', [])
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['items']);
});
```

### Response Assertions

```php
$response
    ->assertOk()                           // 200
    ->assertCreated()                      // 201
    ->assertAccepted()                     // 202
    ->assertNoContent()                    // 204
    ->assertNotFound()                     // 404
    ->assertUnprocessable()                // 422
    ->assertForbidden()                    // 403
    ->assertUnauthorized()                 // 401
    ->assertJsonPath('data.status', 'placed')
    ->assertJsonCount(3, 'data.items')
    ->assertJsonMissingPath('data.cost_cents')  // never expose internal fields
    ->assertJsonStructure(['data' => ['id', 'status', 'total' => ['amount', 'currency']]]);
```

---

## Database Assertions

```php
$this->assertDatabaseHas('orders', ['id' => $order->id, 'status' => 'shipped']);
$this->assertDatabaseMissing('orders', ['id' => $deleted->id]);
$this->assertDatabaseCount('orders', 3);
$this->assertSoftDeleted('orders', ['id' => $order->id]);
```

---

## Mocking & Faking

### Mail

```php
Mail::fake();

// trigger action that sends mail...

Mail::assertSent(OrderConfirmationMail::class, fn ($mail) =>
    $mail->hasTo($user->email) && $mail->order->id === $order->id
);
Mail::assertNotSent(InvoiceMail::class);
Mail::assertSentCount(1);
```

### Notifications

```php
Notification::fake();

$order->ship();

Notification::assertSentTo($order->customer, OrderShippedNotification::class,
    fn ($n) => $n->order->id === $order->id
);
Notification::assertNothingSentTo($admin);
```

### Events

```php
Event::fake([OrderWasPlaced::class, OrderWasShipped::class]);

$handler->handle(new PlaceOrder('order-1', 'customer-1', []));

Event::assertDispatched(OrderWasPlaced::class, fn ($e) => $e->orderId === 'order-1');
Event::assertNotDispatched(OrderWasShipped::class);
```

### Jobs & Queues

```php
Queue::fake();

$this->postJson('/api/reports/export')->assertAccepted();

Queue::assertPushed(ExportReportJob::class, fn ($job) => $job->userId === $user->id);
Queue::assertPushedOn('exports', ExportReportJob::class);
Queue::assertNothingPushed();
```

### Storage

```php
Storage::fake('s3');

$this->postJson('/api/uploads', ['file' => UploadedFile::fake()->image('photo.jpg', 800, 600)]);

Storage::disk('s3')->assertExists('uploads/photo.jpg');
Storage::disk('s3')->assertMissing('uploads/malware.php');
```

### HTTP (External APIs)

```php
Http::fake([
    'stripe.com/*'          => Http::response(['id' => 'ch_123', 'status' => 'succeeded'], 200),
    'warehouse.example.com' => Http::response(['synced' => true], 200),
    '*'                     => Http::response(['error' => 'unmocked'], 500),  // catch-all
]);

$service->chargeCustomer($user, 2999);

Http::assertSent(fn ($req) =>
    str_contains($req->url(), 'stripe.com') &&
    $req['amount'] === 2999
);
```

---

## Testing Commands

```php
it('imports orders from CSV', function () {
    Storage::fake('local');
    Storage::put('imports/orders.csv', "sku,qty\nABC,2\nDEF,1\n");

    $this->artisan('orders:import', ['file' => 'imports/orders.csv'])
        ->expectsOutput('Imported 2 orders')
        ->doesntExpectOutput('Error')
        ->assertExitCode(0);

    expect(Order::count())->toBe(2);
});

it('dry-run shows output without saving', function () {
    $this->artisan('orders:import', ['--dry-run' => true, 'file' => 'test.csv'])
        ->assertExitCode(0);

    expect(Order::count())->toBe(0);
});
```

---

## Factories (Advanced)

```php
final class OrderFactory extends Factory
{
    public function definition(): array
    {
        return [
            'id'          => (string) Str::uuid(),
            'customer_id' => Customer::factory(),
            'status'      => OrderStatus::Placed,
            'total_cents' => $this->faker->numberBetween(1000, 100000),
        ];
    }

    public function placed(): static   { return $this->state(['status' => OrderStatus::Placed]); }
    public function shipped(): static  { return $this->state(['status' => OrderStatus::Shipped]); }
    public function cancelled(): static { return $this->state(['status' => OrderStatus::Cancelled]); }

    public function withItems(int $count = 3): static
    {
        return $this->has(OrderItem::factory()->count($count), 'items');
    }

    public function forCustomer(Customer $customer): static
    {
        return $this->state(['customer_id' => $customer->id]);
    }

    // Computed state
    public function highValue(): static
    {
        return $this->state(['total_cents' => $this->faker->numberBetween(50000, 500000)]);
    }
}

// Usage
$order = Order::factory()->shipped()->withItems(5)->forCustomer($customer)->create();
$orders = Order::factory()->highValue()->count(10)->create();
```

---

## Architecture Fitness Tests

Enforce layer rules and coding conventions in CI:

```php
// tests/Architecture/LayerTest.php
test('domain layer has no framework dependencies')
    ->expect('App\Domain')
    ->not->toUse(\Illuminate\Database\Eloquent\Model::class)
    ->not->toUse('Illuminate\Support\Facades');

test('all controllers are final')
    ->expect('App\Http\Controllers')
    ->toBeFinal();

test('no debug functions in codebase')
    ->expect(['dd', 'dump', 'ray', 'var_dump', 'die'])
    ->not->toBeUsed();

test('request classes extend FormRequest')
    ->expect('App\Http\Requests')
    ->toExtend(\Illuminate\Foundation\Http\FormRequest::class);
```

---

## Mutation Testing (Infection)

Verifies your tests actually catch bugs, not just execute code.

```bash
composer require --dev infection/infection
vendor/bin/infection --min-msi=80 --min-covered-msi=90
```

```json
// infection.json5
{
    "source": {"directories": ["app/Domain", "app/Application"]},
    "minMsi": 80,
    "minCoveredMsi": 90
}
```

**MSI > 80% means 80% of code mutations are caught by your tests.**

---

## Parallel Testing

```bash
php artisan test --parallel --processes=8
```

Uses `RefreshDatabase` (not `DatabaseTransactions`) — each process gets its own DB.

---

## Browser Testing (Laravel Dusk)

```bash
composer require --dev laravel/dusk
php artisan dusk:install
```

```php
final class CheckoutTest extends DuskTestCase
{
    public function test_user_can_complete_checkout(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->loginAs(User::factory()->create())
                ->visit('/cart')
                ->press('@checkout-btn')
                ->waitForLocation('/checkout')
                ->type('@card-number', '4242424242424242')
                ->select('@shipping-method', 'express')
                ->press('@place-order')
                ->waitForText('Order confirmed')
                ->assertPathBeginsWith('/orders/');
        });
    }
}
```

```bash
php artisan dusk  # headless Chrome
```

---

## Contract Tests

Lock down API response shapes to prevent accidental breaking changes:

```php
it('order resource matches contract', function () {
    $order = Order::factory()->withItems(2)->create();

    $this->actingAs($order->customer)
        ->getJson("/api/v1/orders/{$order->id}")
        ->assertOk()
        ->assertJsonStructure([
            'data' => ['id', 'status', 'total' => ['amount', 'currency'], 'items', 'links'],
        ]);

    // Strict: no extra fields
    $keys = array_keys(
        $this->getJson("/api/v1/orders/{$order->id}")->json('data')
    );
    expect($keys)->toBe(['id', 'status', 'total', 'items', 'customer', 'links']);
});
```

---

## Performance Assertions

```php
it('order listing makes at most 3 queries', function () {
    Order::factory(25)->create();

    DB::enableQueryLog();
    $this->getJson('/api/v1/orders')->assertOk();
    expect(count(DB::getQueryLog()))->toBeLessThanOrEqual(3);
    DB::disableQueryLog();
});
```

---

## Snapshot Testing

```bash
composer require --dev spatie/pest-plugin-snapshots
```

```php
it('order resource matches snapshot', function () {
    $order = Order::factory()->create(['id' => 'fixed-uuid']);
    expect((new OrderResource($order))->toArray(request()))->toMatchSnapshot();
});
// Update: php artisan test --update-snapshots
```

---

## Testing AI & MCP

```php
// Mock Prism in tests — never call real AI APIs
use EchoLabs\Prism\Testing\PrismFake;
use EchoLabs\Prism\ValueObjects\TextResult;
use EchoLabs\Prism\Enums\FinishReason;

it('summarizes a product review', function () {
    Prism::fake([
        new TextResult(text: 'Great product, fast delivery.', finishReason: FinishReason::Stop),
    ]);

    $result = app(ReviewSummarizer::class)->summarize($review);

    expect($result)->toBe('Great product, fast delivery.');
    Prism::assertCallCount(1);
});

// Test MCP tools directly
it('get_order MCP tool returns correct structure', function () {
    $order = Order::factory()->withItems(2)->create();

    $result = app(GetOrderTool::class)->handle($order->id);

    expect($result)->toHaveKeys(['id', 'status', 'total_cents', 'items'])
        ->and($result['items'])->toHaveCount(2);
});
```

---

## CI Configuration (GitHub Actions)

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env: {MYSQL_DATABASE: testing, MYSQL_ROOT_PASSWORD: password}
        options: --health-cmd="mysqladmin ping" --health-interval=10s

    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: {php-version: '8.3', coverage: xdebug}

      - run: composer install --no-interaction
      - run: php artisan key:generate --env=testing

      - name: Run tests
        run: php artisan test --parallel --coverage --min=85
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password
```
