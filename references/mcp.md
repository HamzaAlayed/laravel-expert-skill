# MCP (Model Context Protocol) with Laravel

## What is MCP?

MCP is an open protocol that lets AI models (Claude, GPT, etc.) interact with external systems through a standardized interface. A **Laravel application can act as an MCP server**, exposing tools, resources, and prompts that AI agents can call at runtime.

```
AI Agent (Claude Code / Claude Desktop)
    ↕  MCP Protocol (JSON-RPC over stdio / HTTP+SSE)
Laravel MCP Server
    ↕
Database / APIs / Business Logic
```

---

## Installation

```bash
composer require php-mcp/laravel
php artisan vendor:publish --tag=mcp-config
php artisan mcp:install
```

```php
// config/mcp.php
return [
    'server' => [
        'name'    => 'My Laravel App',
        'version' => '1.0.0',
    ],
    'transport' => env('MCP_TRANSPORT', 'http'), // 'http' or 'stdio'
    'route'     => env('MCP_ROUTE', '/mcp'),
    'middleware' => ['api', 'mcp.auth'],
];
```

---

## Defining MCP Tools

Tools are callable functions the AI can invoke — they have inputs, execute logic, and return results.

```php
// app/Mcp/Tools/GetOrderTool.php
use PhpMcp\Laravel\Attributes\Tool;
use PhpMcp\Laravel\Attributes\Param;

final class GetOrderTool
{
    #[Tool(
        name: 'get_order',
        description: 'Retrieve an order by ID including its line items and current status'
    )]
    public function handle(
        #[Param(description: 'The UUID of the order to retrieve')]
        string $order_id
    ): array {
        $order = Order::with('items.product', 'customer')->find($order_id);

        if (! $order) {
            return ['error' => "Order {$order_id} not found"];
        }

        return [
            'id'          => $order->id,
            'status'      => $order->status,
            'total_cents' => $order->total_cents,
            'customer'    => $order->customer->name,
            'items'       => $order->items->map(fn ($item) => [
                'sku'      => $item->product->sku,
                'name'     => $item->product->name,
                'quantity' => $item->quantity,
            ])->all(),
            'created_at' => $order->created_at->toIso8601String(),
        ];
    }
}
```

```php
// app/Mcp/Tools/SearchProductsTool.php
final class SearchProductsTool
{
    #[Tool(
        name: 'search_products',
        description: 'Search the product catalog by keyword, category, or price range'
    )]
    public function handle(
        #[Param(description: 'Search keyword')]
        string $query = '',
        #[Param(description: 'Filter by category name')]
        ?string $category = null,
        #[Param(description: 'Maximum price in cents')]
        ?int $max_price_cents = null,
        #[Param(description: 'Number of results to return (default 10, max 50)')]
        int $limit = 10,
    ): array {
        return Product::search($query)
            ->when($category, fn ($q) => $q->where('category', $category))
            ->when($max_price_cents, fn ($q) => $q->where('price_cents', '<=', $max_price_cents))
            ->take(min($limit, 50))
            ->get()
            ->map(fn ($p) => [
                'id'          => $p->id,
                'name'        => $p->name,
                'sku'         => $p->sku,
                'price_cents' => $p->price_cents,
                'in_stock'    => $p->stock > 0,
            ])
            ->all();
    }
}
```

```php
// app/Mcp/Tools/CreateOrderTool.php
final class CreateOrderTool
{
    public function __construct(private readonly PlaceOrderHandler $handler) {}

    #[Tool(
        name: 'create_order',
        description: 'Place a new order for a customer'
    )]
    public function handle(
        #[Param(description: 'Customer UUID')]
        string $customer_id,
        #[Param(description: 'Array of items: [{sku, quantity}]')]
        array $items,
        #[Param(description: 'Optional coupon code')]
        ?string $coupon_code = null,
    ): array {
        try {
            $orderId = Str::uuid()->toString();
            $this->handler->handle(new PlaceOrder($orderId, $customer_id, $items, $coupon_code));
            return ['success' => true, 'order_id' => $orderId];
        } catch (\DomainException $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }
}
```

---

## Defining MCP Resources

Resources expose read-only data the AI can inspect — documentation, configurations, live data snapshots.

```php
// app/Mcp/Resources/ProductCatalogResource.php
use PhpMcp\Laravel\Attributes\Resource;

final class ProductCatalogResource
{
    #[Resource(
        uri: 'catalog://products/summary',
        name: 'Product Catalog Summary',
        description: 'High-level summary of the product catalog including category counts and inventory status',
        mimeType: 'application/json',
    )]
    public function handle(): string
    {
        return json_encode([
            'total_products'      => Product::count(),
            'active_products'     => Product::where('is_active', true)->count(),
            'out_of_stock'        => Product::where('stock', 0)->count(),
            'categories'          => Category::withCount('products')->get()
                ->map(fn ($c) => ['name' => $c->name, 'count' => $c->products_count])
                ->all(),
            'last_updated'        => Product::max('updated_at'),
        ]);
    }
}
```

```php
// Dynamic resource template — AI can request any order
#[Resource(
    uri: 'orders://{order_id}',
    name: 'Order Details',
    description: 'Full details for a specific order',
    mimeType: 'application/json',
)]
public function handle(string $order_id): string
{
    $order = Order::with('items', 'customer')->findOrFail($order_id);
    return json_encode(OrderData::fromModel($order));
}
```

---

## Defining MCP Prompts

Prompts are reusable message templates the AI can load and use:

```php
// app/Mcp/Prompts/CustomerSupportPrompt.php
use PhpMcp\Laravel\Attributes\Prompt;

final class CustomerSupportPrompt
{
    #[Prompt(
        name: 'customer_support',
        description: 'System prompt for customer support interactions'
    )]
    public function handle(
        #[Param(description: 'Customer name')]
        string $customer_name,
        #[Param(description: 'Order ID if related to a specific order')]
        ?string $order_id = null,
    ): array {
        $context = $order_id
            ? "The customer is asking about order #{$order_id}."
            : 'The customer has a general inquiry.';

        return [
            [
                'role'    => 'user',
                'content' => "You are a helpful support agent for our e-commerce platform. "
                    . "You are speaking with {$customer_name}. {$context} "
                    . "Be concise, empathetic, and solution-focused.",
            ],
        ];
    }
}
```

---

## Registering Tools, Resources & Prompts

```php
// app/Providers/McpServiceProvider.php
use PhpMcp\Laravel\Facades\Mcp;

final class McpServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Mcp::tools([
            GetOrderTool::class,
            SearchProductsTool::class,
            CreateOrderTool::class,
            UpdateOrderStatusTool::class,
            GetCustomerTool::class,
        ]);

        Mcp::resources([
            ProductCatalogResource::class,
            OrderResource::class,
        ]);

        Mcp::prompts([
            CustomerSupportPrompt::class,
        ]);
    }
}
```

---

## HTTP Transport (Remote MCP Server)

Expose over HTTP with SSE for streaming (supports Claude Desktop, Claude Code via `--mcp-server`):

```php
// routes/api.php — auto-registered by package
Route::prefix('mcp')->middleware(['api', 'mcp.auth'])->group(function () {
    Route::post('/', McpController::class);           // JSON-RPC endpoint
    Route::get('/sse', McpSseController::class);      // SSE stream for Claude
});
```

```bash
# .env
MCP_TRANSPORT=http
MCP_ROUTE=/mcp
```

---

## Authentication

```php
// app/Http/Middleware/McpAuth.php
final class McpAuth
{
    public function handle(Request $request, Closure $next): mixed
    {
        $token = $request->bearerToken() ?? $request->header('X-MCP-Token');

        $client = McpClient::where('token_hash', hash('sha256', $token ?? ''))
            ->where('is_active', true)
            ->first();

        if (! $client) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        $request->attributes->set('mcp_client', $client);
        return $next($request);
    }
}
```

---

## stdio Transport (CLI / Claude Code Integration)

```bash
# .env for stdio mode
MCP_TRANSPORT=stdio

# Run as stdio server
php artisan mcp:serve
```

```json
// .claude/mcp_servers.json (Claude Code config)
{
    "my-laravel-app": {
        "command": "php",
        "args": ["/var/www/html/artisan", "mcp:serve"],
        "env": {
            "APP_ENV": "production",
            "MCP_TRANSPORT": "stdio"
        }
    }
}
```

```json
// Claude Desktop: ~/Library/Application Support/Claude/claude_desktop_config.json
{
    "mcpServers": {
        "my-laravel-app": {
            "command": "php",
            "args": ["/path/to/artisan", "mcp:serve"]
        }
    }
}
```

---

## Tool Design Best Practices

**Return structured data, not prose:**
```php
// ❌ BAD — AI has to parse text
return "Order #123 is currently in shipped status, updated 2 hours ago";

// ✅ GOOD — machine-readable
return ['order_id' => '123', 'status' => 'shipped', 'updated_at' => '2026-01-01T10:00:00Z'];
```

**Always return errors as data, not exceptions:**
```php
// ❌ BAD — throws exception that breaks tool call
throw new \RuntimeException('Order not found');

// ✅ GOOD — AI can handle gracefully
return ['error' => 'Order not found', 'order_id' => $orderId];
```

**Scope tools to what the AI actually needs:**
- One tool per action (get, create, update, search)
- Descriptions must tell the AI *when* to use the tool, not just *what* it does
- Parameters must have clear descriptions and sensible defaults

**Idempotent write tools:**
```php
// Accept idempotency key from AI to prevent duplicate operations
#[Param(description: 'Unique key to prevent duplicate orders')]
?string $idempotency_key = null,
```

---

## Testing MCP Tools

```php
it('get_order tool returns correct structure', function () {
    $order = Order::factory()->withItems(2)->create();

    $tool = app(GetOrderTool::class);
    $result = $tool->handle($order->id);

    expect($result)
        ->toHaveKeys(['id', 'status', 'total_cents', 'customer', 'items'])
        ->and($result['items'])->toHaveCount(2);
});

it('get_order tool returns error for unknown order', function () {
    $result = app(GetOrderTool::class)->handle('non-existent-id');

    expect($result)->toHaveKey('error');
});
```
