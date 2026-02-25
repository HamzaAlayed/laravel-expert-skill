# Broadcasting & Real-Time with Laravel Reverb

## Stack Overview

```
Laravel Event → Broadcast → Reverb (WebSocket server) → Echo (JS client) → React
```

Laravel Reverb is the first-party WebSocket server (replaces Pusher/Soketi for self-hosted).

## Installation

```bash
# Server (PHP)
composer require laravel/reverb
php artisan reverb:install

# Client (JS)
npm install --save-dev laravel-echo pusher-js
```

```bash
# .env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http

# Vite
VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"
```

```ts
// resources/js/echo.ts
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

window.Pusher = Pusher

export const echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    wssPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: import.meta.env.VITE_REVERB_SCHEME === 'https',
    enabledTransports: ['ws', 'wss'],
})
```

## Event Types

| Channel Type | Class Interface | Use When |
|---|---|---|
| Public | `ShouldBroadcast` | Anyone can listen (no auth) |
| Private | `ShouldBroadcast` + `private-` prefix | Authenticated users only |
| Presence | `ShouldBroadcast` + `presence-` prefix | Know who is online in the channel |

## Creating a Broadcastable Event

```php
// app/Events/OrderStatusUpdated.php
final class OrderStatusUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("orders.{$this->order->id}"),
            new PrivateChannel("customers.{$this->order->customer_id}"),
        ];
    }

    // Only send specific data to the client (never expose full model)
    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'status'   => $this->order->status,
            'updated_at' => $this->order->updated_at->toIso8601String(),
        ];
    }

    // Custom event name (default: class name)
    public function broadcastAs(): string
    {
        return 'order.status.updated';
    }
}
```

**Dispatch asynchronously (recommended):**
```php
// implements ShouldBroadcastNow to skip queue (use sparingly)
broadcast(new OrderStatusUpdated($order));         // queued (default)
broadcast(new OrderStatusUpdated($order))->toOthers(); // skip current user
```

## Channel Authorization

```php
// routes/channels.php
Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->can('view', Order::findOrFail($orderId));
});

Broadcast::channel('customers.{customerId}', function (User $user, int $customerId) {
    return (int) $user->id === (int) $customerId;
});

// Presence channel — return user data (not just bool)
Broadcast::channel('support.room.{roomId}', function (User $user, int $roomId) {
    if ($user->can('join', SupportRoom::findOrFail($roomId))) {
        return ['id' => $user->id, 'name' => $user->name, 'avatar' => $user->avatar_url];
    }
});
```

## Listening in React

```tsx
// hooks/useOrderStatus.ts
import { useEffect, useState } from 'react'
import { echo } from '@/echo'

export function useOrderStatus(orderId: string, initialStatus: string) {
    const [status, setStatus] = useState(initialStatus)

    useEffect(() => {
        const channel = echo.private(`orders.${orderId}`)

        channel.listen('.order.status.updated', (event: { status: string }) => {
            setStatus(event.status)
        })

        return () => {
            echo.leave(`orders.${orderId}`)
        }
    }, [orderId])

    return status
}
```

```tsx
// Presence channel — who's online
useEffect(() => {
    const channel = echo.join('support.room.1')

    channel
        .here((users: User[]) => setOnlineUsers(users))
        .joining((user: User) => setOnlineUsers(prev => [...prev, user]))
        .leaving((user: User) => setOnlineUsers(prev => prev.filter(u => u.id !== user.id)))
        .listen('.message.sent', (e: MessageEvent) => addMessage(e.message))

    return () => echo.leave('support.room.1')
}, [])
```

## Server-Sent Events (SSE) — Alternative for One-Way Streams

Use SSE for one-directional server → client streams (logs, progress bars). No WebSocket needed.

```php
// routes/web.php
Route::get('/export/progress/{jobId}', function (string $jobId) {
    return response()->stream(function () use ($jobId) {
        while (true) {
            $progress = Cache::get("export.{$jobId}.progress", 0);
            echo "data: " . json_encode(['progress' => $progress]) . "\n\n";
            ob_flush(); flush();
            if ($progress >= 100) break;
            sleep(1);
        }
    }, 200, [
        'Content-Type'  => 'text/event-stream',
        'Cache-Control' => 'no-cache',
        'X-Accel-Buffering' => 'no',
    ]);
});
```

```tsx
useEffect(() => {
    const source = new EventSource(`/export/progress/${jobId}`)
    source.onmessage = (e) => {
        const { progress } = JSON.parse(e.data)
        setProgress(progress)
        if (progress >= 100) source.close()
    }
    return () => source.close()
}, [jobId])
```

## Reverb Production Configuration

```php
// config/reverb.php
'servers' => [
    'reverb' => [
        'host' => '0.0.0.0',
        'port' => 8080,
        'max_request_size' => 10_000,   // bytes
        'scaling' => [
            'enabled' => true,          // horizontal scaling via Redis pub/sub
            'channel' => 'reverb',
            'server'  => 'default',     // Redis connection
        ],
    ],
],
```

**Supervisor config for Reverb:**
```ini
[program:reverb]
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
stdout_logfile=/var/log/reverb.log
```

**Nginx proxy:**
```nginx
location /app/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

## Notifications via Broadcasting

```php
// Notify in real-time through notifications channel
final class OrderShippedNotification extends Notification implements ShouldBroadcast
{
    public function via(object $notifiable): array
    {
        return ['database', 'broadcast'];
    }

    public function toBroadcast(object $notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'order_id' => $this->order->id,
            'message'  => "Your order #{$this->order->id} has shipped!",
        ]);
    }
}
```

```tsx
// Listen to user's notification channel
echo.private(`App.Models.User.${userId}`)
    .notification((notification: { message: string }) => {
        toast.success(notification.message)
    })
```

## Model Broadcasting (Laravel 10+)

Broadcast Eloquent model changes automatically:

```php
// app/Models/Order.php
use Illuminate\Database\Eloquent\BroadcastsEvents;

class Order extends Model
{
    use BroadcastsEvents;

    public function broadcastOn(string $event): array
    {
        return [new PrivateChannel("orders.{$this->id}")];
    }
}
```

Fires `OrderCreated`, `OrderUpdated`, `OrderDeleted` automatically on model events.
