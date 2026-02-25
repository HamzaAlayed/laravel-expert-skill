# ReactJS & Inertia.js with Laravel

## Stack Overview

```
Laravel (backend) ←→ Inertia.js (protocol) ←→ React (frontend)
```

Inertia replaces Blade for full-page rendering while keeping a monolithic Laravel codebase. No separate API layer needed — controllers return Inertia responses with typed props.

## Installation

```bash
composer require inertiajs/inertia-laravel
npm install @inertiajs/react react react-dom
```

```php
// resources/views/app.blade.php
<!DOCTYPE html>
<html>
<head>
    @viteReactRefresh
    @vite(['resources/css/app.css', 'resources/js/app.tsx'])
    @inertiaHead
</head>
<body>
    @inertia
</body>
</html>
```

```tsx
// resources/js/app.tsx
import { createInertiaApp } from '@inertiajs/react'
import { createRoot } from 'react-dom/client'

createInertiaApp({
    resolve: name => {
        const pages = import.meta.glob('./Pages/**/*.tsx', { eager: true })
        return pages[`./Pages/${name}.tsx`]
    },
    setup({ el, App, props }) {
        createRoot(el).render(<App {...props} />)
    },
})
```

## Controller → Page Props (Typed)

```php
// Laravel controller
final class OrderController extends Controller
{
    public function index(): Response
    {
        return Inertia::render('Orders/Index', [
            'orders'  => OrderResource::collection(
                Order::with('customer')->latest()->paginate(25)
            ),
            'filters' => request()->only(['status', 'search']),
        ]);
    }
}
```

```tsx
// resources/js/Pages/Orders/Index.tsx
import { Head, Link } from '@inertiajs/react'

interface Order {
    id: string
    status: string
    total: { amount: number; currency: string }
    customer: { name: string }
}

interface Props {
    orders: { data: Order[]; links: PaginationLinks; meta: PaginationMeta }
    filters: { status?: string; search?: string }
}

export default function OrdersIndex({ orders, filters }: Props) {
    return (
        <>
            <Head title="Orders" />
            {/* component body */}
        </>
    )
}
```

## Forms with useForm

```tsx
import { useForm } from '@inertiajs/react'

export default function CreateOrder() {
    const { data, setData, post, processing, errors, reset } = useForm({
        customer_id: '',
        items: [] as LineItem[],
    })

    function submit(e: React.FormEvent) {
        e.preventDefault()
        post(route('orders.store'), {
            onSuccess: () => reset(),
            preserveScroll: true,
        })
    }

    return (
        <form onSubmit={submit}>
            <input
                value={data.customer_id}
                onChange={e => setData('customer_id', e.target.value)}
            />
            {errors.customer_id && <p className="text-red-500">{errors.customer_id}</p>}
            <button disabled={processing}>Place Order</button>
        </form>
    )
}
```

## Shared Data (Global Props)

```php
// App\Http\Middleware\HandleInertiaRequests.php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => [
            'user' => $request->user()?->only('id', 'name', 'email', 'roles'),
        ],
        'flash' => [
            'success' => $request->session()->get('success'),
            'error'   => $request->session()->get('error'),
        ],
    ];
}
```

```tsx
// Access anywhere via usePage
import { usePage } from '@inertiajs/react'

const { auth, flash } = usePage<SharedProps>().props
```

## State Management

| Need | Solution |
|---|---|
| Server-synced state | Inertia props + `useForm` |
| Local UI state (modals, toggles) | `useState`, `useReducer` |
| Global client state (cart, theme) | Zustand or React Context |
| Server cache + invalidation | TanStack Query (if using API mode) |

**Zustand store example:**
```tsx
import { create } from 'zustand'

interface CartStore {
    items: CartItem[]
    addItem: (item: CartItem) => void
    removeItem: (id: string) => void
}

const useCart = create<CartStore>((set) => ({
    items: [],
    addItem: (item) => set((s) => ({ items: [...s.items, item] })),
    removeItem: (id) => set((s) => ({ items: s.items.filter(i => i.id !== id) })),
}))
```

## Layouts

```tsx
// resources/js/Layouts/AppLayout.tsx
export default function AppLayout({ children }: { children: React.ReactNode }) {
    return (
        <div className="min-h-screen bg-gray-100">
            <Navbar />
            <main className="py-12">{children}</main>
        </div>
    )
}

// Attach persistent layout to page (avoids re-mount on navigation)
OrdersIndex.layout = (page: React.ReactNode) => <AppLayout>{page}</AppLayout>
```

## Server-Side Rendering (SSR)

```bash
npm install @inertiajs/react
npm install --save-dev @inertiajs/server
```

```tsx
// resources/js/ssr.tsx
import { createInertiaApp } from '@inertiajs/react'
import ReactDOMServer from 'react-dom/server'

export default function render(page: unknown) {
    return createInertiaApp({
        page,
        render: ReactDOMServer.renderToString,
        resolve: name => {
            const pages = import.meta.glob('./Pages/**/*.tsx', { eager: true })
            return pages[`./Pages/${name}.tsx`]
        },
        setup: ({ App, props }) => <App {...props} />,
    })
}
```

```bash
php artisan inertia:start-ssr  # starts Node SSR server
```

## Component Patterns

**Compound components (reusable UI with shared context):**
```tsx
// Modal compound component
const Modal = {
    Root: ModalRoot,
    Trigger: ModalTrigger,
    Content: ModalContent,
    Footer: ModalFooter,
}

// Usage
<Modal.Root>
    <Modal.Trigger>Open</Modal.Trigger>
    <Modal.Content>
        <p>Content here</p>
        <Modal.Footer>
            <button>Confirm</button>
        </Modal.Footer>
    </Modal.Content>
</Modal.Root>
```

**Data table with server-side sort/filter:**
```tsx
function OrdersTable({ orders, filters }: Props) {
    const { get } = useForm(filters)

    function applyFilter(key: string, value: string) {
        get(route('orders.index'), {
            data: { ...filters, [key]: value },
            preserveState: true,
            replace: true,
        })
    }
    // ...
}
```

## Performance Patterns

- Use `<Link prefetch>` for hover-based prefetching of likely next pages
- `preserveState: true` on filter changes avoids full re-render
- Split large page bundles: `import.meta.glob('./Pages/**/*.tsx')` — Vite code-splits automatically
- Avoid large arrays in shared data — they re-serialize on every Inertia visit
- Use `defer` prop for non-critical data loaded after initial render:

```php
return Inertia::render('Dashboard', [
    'stats' => Inertia::defer(fn () => $this->computeExpensiveStats()),
]);
```

## Laravel + React Without Inertia (API Mode)

When building a fully decoupled SPA:
- Use `laravel/sanctum` with `SPA authentication` (cookie-based, same domain)
- Or token-based auth for cross-domain
- Run React dev server separately (Vite); proxy API calls via `vite.config.ts` proxy
- See `references/api.md` for auth strategies
