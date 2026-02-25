# Vite in Laravel

## Laravel Vite Plugin Basics

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import laravel from 'laravel-vite-plugin'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.tsx',
                'resources/js/admin.tsx',   // separate entry per app
            ],
            refresh: true,  // auto-reload on Blade/PHP changes
        }),
        react(),
        tailwindcss(),
    ],
    resolve: {
        alias: {
            '@': '/resources/js',  // import from '@/components/Button'
        },
    },
})
```

```blade
{{-- Blade: load assets --}}
@viteReactRefresh
@vite(['resources/css/app.css', 'resources/js/app.tsx'])
```

## Path Aliases

```ts
// vite.config.ts
resolve: {
    alias: {
        '@':        '/resources/js',
        '@css':     '/resources/css',
        '@components': '/resources/js/components',
        '@pages':   '/resources/js/Pages',
        '@lib':     '/resources/js/lib',
    },
}
```

```json
// tsconfig.json — keep in sync with vite aliases
{
    "compilerOptions": {
        "paths": {
            "@/*":           ["./resources/js/*"],
            "@components/*": ["./resources/js/components/*"],
            "@lib/*":        ["./resources/js/lib/*"]
        }
    }
}
```

## Code Splitting & Lazy Loading

Vite automatically code-splits at dynamic `import()` boundaries.

**Inertia page-level splitting (automatic):**
```ts
// app.tsx — each page is a separate chunk
resolve: name => {
    const pages = import.meta.glob('./Pages/**/*.tsx')  // lazy by default
    return pages[`./Pages/${name}.tsx`]()
},
```

**Manual lazy loading for heavy components:**
```tsx
import { lazy, Suspense } from 'react'

const HeavyChart = lazy(() => import('@components/HeavyChart'))

function Dashboard() {
    return (
        <Suspense fallback={<ChartSkeleton />}>
            <HeavyChart data={data} />
        </Suspense>
    )
}
```

**Preload hints** — tell browser to fetch a chunk early:
```tsx
import { Link } from '@inertiajs/react'

// Vite prefetches the chunk on hover
<Link href="/orders" prefetch>Orders</Link>
```

## Environment Variables

Vite only exposes variables prefixed with `VITE_` to client code:

```bash
# .env
VITE_APP_NAME="${APP_NAME}"
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
```

```ts
// Access in TypeScript
const appName = import.meta.env.VITE_APP_NAME
const pusherKey = import.meta.env.VITE_PUSHER_APP_KEY
```

**TypeScript types for env vars:**
```ts
// resources/js/env.d.ts
interface ImportMetaEnv {
    readonly VITE_APP_NAME: string
    readonly VITE_PUSHER_APP_KEY: string
}
interface ImportMeta {
    readonly env: ImportMetaEnv
}
```

## Multiple Entry Points

Use separate entries for different audiences (public site vs. admin panel):

```ts
laravel({
    input: [
        'resources/css/app.css',
        'resources/js/app.tsx',         // customer-facing SPA
        'resources/css/admin.css',
        'resources/js/admin.tsx',        // admin panel
        'resources/js/embed.ts',         // embeddable widget (no React)
    ],
})
```

In Blade, load only the relevant entry per layout:
```blade
{{-- layouts/admin.blade.php --}}
@vite(['resources/css/admin.css', 'resources/js/admin.tsx'])
```

## HMR (Hot Module Replacement)

HMR works out of the box. To avoid losing React component state on update, ensure React Fast Refresh is active:

```ts
plugins: [react()]  // includes Fast Refresh automatically
```

**HMR with HTTPS (local):**
```ts
server: {
    https: true,
    host: 'myapp.test',
}
```

Or use Valet/Herd — they handle HTTPS automatically and Vite detects the proxy.

**Behind a reverse proxy (Docker, WSL2):**
```ts
server: {
    host: '0.0.0.0',
    hmr: {
        host: 'localhost',  // browser connects here
    },
}
```

## Build Optimization

```ts
build: {
    rollupOptions: {
        output: {
            // Group vendor libraries into one chunk
            manualChunks: {
                vendor: ['react', 'react-dom', '@inertiajs/react'],
                charts: ['recharts'],
            },
        },
    },
    // Increase chunk size warning threshold (default 500KB)
    chunkSizeWarningLimit: 1000,
},
```

**Analyze bundle size:**
```bash
npm install -D rollup-plugin-visualizer
```
```ts
import { visualizer } from 'rollup-plugin-visualizer'

plugins: [
    // ...
    visualizer({ open: true, gzipSize: true }),  // opens stats.html after build
]
```

Target bundle sizes after gzip:
- Initial JS (app entry): < 150KB
- Vendor chunk: < 200KB
- Per-page chunks: < 50KB

## Asset Fingerprinting & CDN

Vite fingerprints all assets by default (`app-[hash].js`). To serve from CDN:

```ts
// vite.config.ts
build: {
    manifest: true,
    outDir: 'public/build',
}
```

```php
// config/vite.php or .env
ASSET_URL=https://cdn.example.com
```

Laravel's `Vite::asset()` and `@vite` directive automatically prepend `ASSET_URL`.

## SSR Build (Inertia)

```ts
// vite.config.ts — add ssr entry
laravel({
    input: ['resources/js/app.tsx'],
    ssr: 'resources/js/ssr.tsx',
})
```

```bash
# Build both client and SSR bundles
npm run build           # client build → public/build/
npm run build -- --ssr  # SSR build → bootstrap/ssr/ssr.js
```

```bash
# Start SSR server (Node process Laravel communicates with)
php artisan inertia:start-ssr
```

## TypeScript Configuration

```json
// tsconfig.json
{
    "compilerOptions": {
        "target": "ESNext",
        "module": "ESNext",
        "moduleResolution": "bundler",
        "jsx": "react-jsx",
        "strict": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "paths": { "@/*": ["./resources/js/*"] }
    },
    "include": ["resources/js/**/*"]
}
```

## Ziggy (Typed Laravel Routes in JS)

```bash
composer require tightenco/ziggy
npm install ziggy-js
```

```ts
// vite.config.ts
laravel({
    input: ['resources/js/app.tsx'],
    refresh: ['routes/**'],  // re-generate route types on route changes
})
```

```tsx
import { route } from 'ziggy-js'

// Fully typed route() calls matching Laravel's route()
<Link href={route('orders.show', { order: order.id })}>View Order</Link>
```

Generate TypeScript types:
```bash
php artisan ziggy:generate --types  # creates resources/js/ziggy.d.ts
```

## Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Assets 404 in prod | Wrong `APP_URL` | Set `APP_URL` in `.env` to match domain |
| HMR not working behind proxy | Wrong HMR host | Set `server.hmr.host` to public hostname |
| `import.meta.env.*` is undefined | Missing `VITE_` prefix | Rename env var with `VITE_` prefix |
| Dynamic class names purged | String concatenation | Use full class name strings (see tailwindcss.md) |
| Large initial bundle | No code splitting | Use `import.meta.glob` lazy mode for pages |
