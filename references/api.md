# API Design in Laravel

## Versioning Strategies

| Strategy | URL Example | Best For |
|---|---|---|
| URI segment | `/api/v1/orders` | Public APIs, breaking changes frequent |
| Header | `Accept: application/vnd.api+json;version=2` | Clean URLs, internal APIs |
| Query param | `/api/orders?version=2` | Simple APIs, avoid for public |

**URI versioning setup:**
```php
// routes/api.php
Route::prefix('v1')->name('api.v1.')->group(base_path('routes/api/v1.php'));
Route::prefix('v2')->name('api.v2.')->group(base_path('routes/api/v2.php'));
```

**Version lifecycle policy:** Deprecate with a `Sunset` header before removing:
```php
->withHeaders(['Sunset' => 'Sat, 01 Jan 2027 00:00:00 GMT', 'Deprecation' => 'true'])
```

## API Resource Transformation (Advanced)

```php
// Conditional fields — never expose internal data unintentionally
final class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'status'     => $this->status,
            'total'      => MoneyResource::make($this->total),
            'items'      => OrderItemResource::collection($this->whenLoaded('items')),
            // Only include for admin role
            'cost_cents' => $this->when(
                $request->user()?->hasRole('admin'),
                $this->cost_cents
            ),
            // Only include when relationship is loaded (prevents N+1)
            'customer'   => CustomerResource::make($this->whenLoaded('customer')),
            'links'      => [
                'self'   => route('api.v1.orders.show', $this->id),
                'cancel' => $this->when(
                    $this->status === 'placed',
                    route('api.v1.orders.cancel', $this->id)
                ),
            ],
        ];
    }
}
```

## Rate Limiting

```php
// app/Providers/AppServiceProvider.php (Laravel 11+; RouteServiceProvider removed)
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Authenticated: 1000 req/min per user
    RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(1000)->by($request->user()->id)
            : Limit::perMinute(60)->by($request->ip());
    });

    // Expensive endpoints: 10 req/min per user
    RateLimiter::for('api.heavy', function (Request $request) {
        return Limit::perMinute(10)
            ->by($request->user()?->id ?? $request->ip())
            ->response(fn () => response()->json([
                'message' => 'Too many requests.',
                'retry_after' => 60,
            ], 429));
    });
}
```

```php
// Apply in routes
Route::middleware(['auth:sanctum', 'throttle:api.heavy'])
    ->post('/reports/export', ExportReportController::class);
```

## OpenAPI / Swagger Documentation

```bash
composer require dedoc/scramble  # Auto-generates from code, no annotations
```

Scramble infers routes, request types, and response shapes from Form Requests and API Resources:
```php
// config/scramble.php
'api_path' => 'api',
'api_domain' => null,
'info' => ['title' => 'My API', 'version' => '1.0.0'],
```

For manual control use `#[ResponseBody]` / `#[RequestBody]` attributes from Scramble.

**Serve docs:** `GET /docs/api` — disable in production or gate with auth.

## API Contracts & Consumer-Driven Testing

Define contracts in `tests/Contracts/`:
```php
// tests/Contracts/OrderResourceContract.php
it('order resource matches contract', function () {
    $order = Order::factory()->create();

    $resource = (new OrderResource($order))->response()->getData(true);

    expect($resource['data'])->toHaveKeys(['id', 'status', 'total', 'links']);
    expect($resource['data']['total'])->toHaveKeys(['amount', 'currency']);
});
```

## Authentication Strategies

| Strategy | Use When | Laravel Package |
|---|---|---|
| Sanctum tokens | SPAs, mobile, internal APIs | `laravel/sanctum` (built-in) |
| Passport (OAuth2) | Third-party clients needing scopes/grants | `laravel/passport` |
| JWT stateless | Microservices, stateless APIs | `php-open-source-saver/jwt-auth` |
| API key | Server-to-server, webhooks | Custom middleware |

**API key middleware (server-to-server):**
```php
final class ValidateApiKey
{
    public function handle(Request $request, Closure $next): mixed
    {
        $key = $request->header('X-Api-Key')
            ?? $request->query('api_key');

        $client = ApiClient::where('key_hash', hash('sha256', $key ?? ''))->first();

        if (! $client || ! $client->is_active) {
            return response()->json(['message' => 'Unauthorized.'], 401);
        }

        $request->merge(['api_client' => $client]);
        return $next($request);
    }
}
```

## Form Request Validation (Advanced)

```php
final class CreateOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Order::class);
    }

    public function rules(): array
    {
        return [
            'items'              => ['required', 'array', 'min:1', 'max:50'],
            'items.*.sku'        => ['required', 'string', 'exists:products,sku'],
            'items.*.quantity'   => ['required', 'integer', 'min:1', 'max:999'],
            'shipping_address'   => ['required', new ValidAddress()],
            'coupon_code'        => ['nullable', 'string', new ValidCoupon($this->user())],
        ];
    }

    public function messages(): array
    {
        return [
            'items.*.sku.exists' => 'Product :input does not exist.',
        ];
    }
}
```

## Consistent Error Response Format

```php
// App\Exceptions\Handler.php
public function register(): void
{
    $this->renderable(function (ValidationException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'message' => 'Validation failed.',
                'errors'  => $e->errors(),
            ], 422);
        }
    });

    $this->renderable(function (ModelNotFoundException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json(['message' => 'Resource not found.'], 404);
        }
    });
}
```

## Pagination Standards

```php
// Cursor pagination (preferred for large datasets / infinite scroll)
return OrderResource::collection(
    Order::orderBy('id')->cursorPaginate(25)
);

// Offset pagination (required for page-jump UI)
return OrderResource::collection(
    Order::latest()->paginate(25)
);
```

Response shape for cursor pagination includes `next_cursor` / `prev_cursor` in `meta.links`. Cursor pagination is O(1) regardless of page depth; offset pagination degrades to O(n) at high page numbers.

## Webhooks (Outbound)

```php
// Dispatch asynchronously, retry with exponential backoff
final class DispatchWebhook implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public int $tries = 5;
    public int $backoff = 60; // seconds; doubles per retry

    public function handle(): void
    {
        $response = Http::timeout(10)
            ->withHeaders(['X-Signature' => $this->signature()])
            ->post($this->endpoint->url, $this->payload);

        if ($response->failed()) {
            $this->release($this->backoff * (2 ** ($this->attempts() - 1)));
        }
    }

    private function signature(): string
    {
        return hash_hmac('sha256', json_encode($this->payload), $this->endpoint->secret);
    }
}
```
