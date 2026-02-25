# AI Integration in Laravel

## Package Selection

| Package | Purpose | Use When |
|---|---|---|
| `echolabs/prism` | Unified multi-provider LLM interface | Multi-provider support, structured output, tool calling |
| `openai-php/laravel` | Direct OpenAI client | OpenAI-only, low-level control |
| `anthropic/sdk` | Direct Anthropic client | Claude-only, low-level control |
| `pgvector/pgvector` | PostgreSQL vector extension | Embeddings + similarity search |

**Recommended default: `echolabs/prism`** — provider-agnostic, Laravel-native, handles streaming, tool calls, and structured output.

---

## Prism (Unified LLM Interface)

```bash
composer require echolabs/prism
php artisan vendor:publish --tag=prism-config
```

```php
// config/prism.php
'providers' => [
    'anthropic' => ['api_key' => env('ANTHROPIC_API_KEY')],
    'openai'    => ['api_key' => env('OPENAI_API_KEY')],
    'ollama'    => ['url' => env('OLLAMA_URL', 'http://localhost:11434/api')],
],
```

### Text Generation

```php
use EchoLabs\Prism\Prism;
use EchoLabs\Prism\Enums\Provider;

$response = Prism::text()
    ->using(Provider::Anthropic, 'claude-sonnet-4-6')
    ->withSystemPrompt('You are a helpful assistant for an e-commerce platform.')
    ->withPrompt("Summarize this product review: {$review->body}")
    ->withMaxTokens(300)
    ->generate();

echo $response->text;
echo $response->usage->inputTokens;
echo $response->usage->outputTokens;
```

### Streaming Responses

```php
// Controller: stream to HTTP client
public function chat(ChatRequest $request): StreamedResponse
{
    return response()->stream(function () use ($request) {
        $stream = Prism::text()
            ->using(Provider::Anthropic, 'claude-sonnet-4-6')
            ->withMessages($request->messages)
            ->stream();

        foreach ($stream as $chunk) {
            echo "data: " . json_encode(['text' => $chunk->text]) . "\n\n";
            ob_flush(); flush();
        }
        echo "data: [DONE]\n\n";
        ob_flush(); flush();
    }, 200, [
        'Content-Type'      => 'text/event-stream',
        'Cache-Control'     => 'no-cache',
        'X-Accel-Buffering' => 'no',
    ]);
}
```

```tsx
// React: consume SSE stream
async function streamChat(messages: Message[]) {
    const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ messages }),
    })

    const reader = response.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
        const { done, value } = await reader.read()
        if (done) break
        const lines = decoder.decode(value).split('\n')
        for (const line of lines) {
            if (line.startsWith('data: ') && line !== 'data: [DONE]') {
                const { text } = JSON.parse(line.slice(6))
                setContent(prev => prev + text)
            }
        }
    }
}
```

### Multi-Turn Conversations

```php
use EchoLabs\Prism\ValueObjects\Messages\UserMessage;
use EchoLabs\Prism\ValueObjects\Messages\AssistantMessage;

$messages = [
    new UserMessage('What is the capital of France?'),
    new AssistantMessage('Paris.'),
    new UserMessage('And what is its population?'),
];

$response = Prism::text()
    ->using(Provider::Anthropic, 'claude-sonnet-4-6')
    ->withMessages($messages)
    ->generate();
```

### Tool Calling (Function Calling)

```php
use EchoLabs\Prism\Tool;

$orderTool = Tool::as('get_order_status')
    ->for('Get the current status of a customer order')
    ->withStringParameter('order_id', 'The order ID to look up')
    ->using(function (string $order_id): string {
        $order = Order::find($order_id);
        return $order
            ? json_encode(['status' => $order->status, 'updated_at' => $order->updated_at])
            : json_encode(['error' => 'Order not found']);
    });

$response = Prism::text()
    ->using(Provider::Anthropic, 'claude-sonnet-4-6')
    ->withTools([$orderTool])
    ->withMaxSteps(5)   // allow multiple tool call rounds
    ->withPrompt("What is the status of order #12345?")
    ->generate();

// Inspect tool calls made
foreach ($response->steps as $step) {
    foreach ($step->toolCalls as $call) {
        logger("Tool called: {$call->name}", $call->arguments());
    }
}
```

### Structured Output

```php
use EchoLabs\Prism\Schema\ObjectSchema;
use EchoLabs\Prism\Schema\StringSchema;
use EchoLabs\Prism\Schema\ArraySchema;

$schema = new ObjectSchema(
    name: 'product_analysis',
    description: 'Analysis of a product review',
    properties: [
        new StringSchema('sentiment', 'positive, negative, or neutral'),
        new StringSchema('summary', 'One sentence summary'),
        new ArraySchema('key_points', 'Main points mentioned', new StringSchema('point', 'A key point')),
    ],
    requiredFields: ['sentiment', 'summary', 'key_points']
);

$response = Prism::structured()
    ->using(Provider::Anthropic, 'claude-sonnet-4-6')
    ->withSchema($schema)
    ->withPrompt("Analyze this review: {$review->body}")
    ->generate();

$analysis = $response->structured;  // typed array matching schema
```

---

## Embeddings & Vector Search

```bash
composer require openai-php/laravel
# PostgreSQL: enable pgvector extension
# ALTER DATABASE mydb ENABLE extension pgvector;
```

```php
// Migration
$table->vector('embedding', 1536);  // OpenAI ada-002 dimensions
$table->index('embedding');          // use pgvector HNSW index for scale

// Store embedding
$embedding = OpenAI::embeddings()->create([
    'model' => 'text-embedding-3-small',
    'input' => $document->content,
])->embeddings[0]->embedding;

$document->update(['embedding' => $embedding]);

// Similarity search
$similar = Document::selectRaw('*, embedding <=> ? as distance', [json_encode($queryEmbedding)])
    ->orderBy('distance')
    ->limit(5)
    ->get();
```

---

## RAG (Retrieval-Augmented Generation)

```php
// 1. Embed user query
$queryEmbedding = OpenAI::embeddings()->create([
    'model' => 'text-embedding-3-small',
    'input' => $userQuestion,
])->embeddings[0]->embedding;

// 2. Find relevant documents
$docs = Document::selectRaw('*, embedding <=> ? as distance', [json_encode($queryEmbedding)])
    ->orderBy('distance')
    ->limit(5)
    ->get();

// 3. Build context + generate answer
$context = $docs->pluck('content')->implode("\n\n---\n\n");

$response = Prism::text()
    ->using(Provider::Anthropic, 'claude-sonnet-4-6')
    ->withSystemPrompt("Answer using only the provided context. If unsure, say so.")
    ->withPrompt("Context:\n{$context}\n\nQuestion: {$userQuestion}")
    ->generate();
```

---

## Queued AI Operations

Long-running AI tasks (embeddings, batch classification) belong in queued jobs:

```php
// app/Jobs/GenerateProductEmbedding.php
final class GenerateProductEmbedding implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $backoff = 30;

    public function __construct(private readonly int $productId) {}

    public function handle(): void
    {
        $product = Product::findOrFail($this->productId);

        $embedding = OpenAI::embeddings()->create([
            'model' => 'text-embedding-3-small',
            'input' => "{$product->name}: {$product->description}",
        ])->embeddings[0]->embedding;

        $product->update(['embedding' => $embedding]);
    }

    public function failed(\Throwable $e): void
    {
        logger()->error("Embedding failed for product {$this->productId}", ['error' => $e->getMessage()]);
    }
}

// Dispatch after product save
Product::created(fn ($product) => GenerateProductEmbedding::dispatch($product->id));
```

---

## Rate Limiting & Cost Control

```php
// Rate limit AI calls per user (protect against abuse + cost overrun)
RateLimiter::for('ai', function (Request $request) {
    return [
        Limit::perMinute(10)->by($request->user()->id),
        Limit::perDay(100)->by($request->user()->id),
    ];
});

// Track token usage in database
final class AiUsageLogger
{
    public function log(User $user, string $model, int $inputTokens, int $outputTokens): void
    {
        AiUsage::create([
            'user_id'       => $user->id,
            'model'         => $model,
            'input_tokens'  => $inputTokens,
            'output_tokens' => $outputTokens,
            'cost_usd'      => $this->calculateCost($model, $inputTokens, $outputTokens),
        ]);
    }
}
```

---

## Image Generation

```php
$response = OpenAI::images()->create([
    'model'   => 'dall-e-3',
    'prompt'  => "Product photo of: {$product->name}",
    'n'       => 1,
    'size'    => '1024x1024',
    'quality' => 'standard',
    'style'   => 'natural',
]);

$imageUrl = $response->data[0]->url;

// Download and store in media library
$product->addMediaFromUrl($imageUrl)
    ->toMediaCollection('ai-images');
```

---

## AI Testing Patterns

```php
// Mock AI responses in tests — never call real API in tests
it('generates product summary', function () {
    Prism::fake([
        new TextResult(text: 'Compact wireless headphones with 30-hour battery.', finishReason: FinishReason::Stop),
    ]);

    $summary = app(ProductSummarizer::class)->summarize($product);

    expect($summary)->toBe('Compact wireless headphones with 30-hour battery.');
    Prism::assertCallCount(1);
});
```
