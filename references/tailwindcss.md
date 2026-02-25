# TailwindCSS in Laravel

## Setup with Vite

```bash
npm install -D tailwindcss @tailwindcss/vite
```

```ts
// vite.config.ts
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
    plugins: [
        laravel({ input: ['resources/css/app.css', 'resources/js/app.tsx'] }),
        tailwindcss(),
    ],
})
```

```css
/* resources/css/app.css */
@import "tailwindcss";
```

> Tailwind v4 uses the Vite plugin instead of PostCSS. No `tailwind.config.js` needed for basic use.

## Custom Design System (Theme)

```css
/* resources/css/app.css */
@import "tailwindcss";

@theme {
    /* Brand colors */
    --color-brand-50:  oklch(97% 0.02 250);
    --color-brand-500: oklch(55% 0.18 250);
    --color-brand-900: oklch(25% 0.12 250);

    /* Typography scale */
    --font-sans: 'Inter Variable', ui-sans-serif, system-ui;
    --font-display: 'Cal Sans', var(--font-sans);

    /* Spacing */
    --spacing-18: 4.5rem;
    --spacing-22: 5.5rem;

    /* Border radius */
    --radius-card: 0.75rem;
    --radius-btn:  0.375rem;
}
```

Usage: `bg-brand-500`, `text-brand-50`, `rounded-card`, `font-display`

## Component Patterns

### Class Variance Authority (CVA) — Typed Variants

```bash
npm install class-variance-authority clsx tailwind-merge
```

```tsx
// lib/utils.ts — merge without conflicts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs))
}
```

```tsx
// components/Button.tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const button = cva(
    // Base
    'inline-flex items-center justify-center font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
    {
        variants: {
            variant: {
                primary:   'bg-brand-500 text-white hover:bg-brand-600',
                secondary: 'bg-white text-gray-900 border border-gray-200 hover:bg-gray-50',
                ghost:     'hover:bg-gray-100 text-gray-700',
                danger:    'bg-red-600 text-white hover:bg-red-700',
            },
            size: {
                sm: 'h-8  px-3 text-sm rounded-btn',
                md: 'h-10 px-4 text-sm rounded-btn',
                lg: 'h-12 px-6 text-base rounded-btn',
            },
        },
        defaultVariants: { variant: 'primary', size: 'md' },
    }
)

interface ButtonProps
    extends React.ButtonHTMLAttributes<HTMLButtonElement>,
        VariantProps<typeof button> {}

export function Button({ variant, size, className, ...props }: ButtonProps) {
    return <button className={cn(button({ variant, size }), className)} {...props} />
}
```

### Data Display Card

```tsx
export function StatCard({
    label, value, delta, icon: Icon
}: StatCardProps) {
    return (
        <div className="rounded-card bg-white p-6 shadow-sm ring-1 ring-gray-900/5">
            <div className="flex items-center justify-between">
                <p className="text-sm font-medium text-gray-500">{label}</p>
                <span className="rounded-lg bg-brand-50 p-2 text-brand-500">
                    <Icon className="h-5 w-5" />
                </span>
            </div>
            <p className="mt-2 text-3xl font-semibold tracking-tight text-gray-900">
                {value}
            </p>
            {delta && (
                <p className={cn(
                    'mt-1 text-sm',
                    delta > 0 ? 'text-green-600' : 'text-red-600'
                )}>
                    {delta > 0 ? '↑' : '↓'} {Math.abs(delta)}% vs last period
                </p>
            )}
        </div>
    )
}
```

## Dark Mode

```css
/* app.css — enable dark mode via class strategy */
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));
```

```tsx
// Toggle dark mode
document.documentElement.classList.toggle('dark')

// Component usage
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
```

Store preference in `localStorage`, apply class on `<html>` before React hydrates to avoid flash.

## Responsive Design

Mobile-first breakpoints:
```
sm:  640px   md:  768px   lg: 1024px   xl: 1280px   2xl: 1536px
```

```tsx
// Responsive grid
<div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
```

**Container with responsive padding:**
```tsx
<div className="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
```

## Form Styling

```tsx
// Consistent form inputs
const inputClass = cn(
    'block w-full rounded-btn border border-gray-300 px-3 py-2 text-sm',
    'placeholder:text-gray-400',
    'focus:border-brand-500 focus:outline-none focus:ring-1 focus:ring-brand-500',
    'disabled:cursor-not-allowed disabled:bg-gray-50 disabled:text-gray-500',
)

// Error state
const inputErrorClass = cn(
    inputClass,
    'border-red-300 focus:border-red-500 focus:ring-red-500',
)
```

## Tailwind v4 Key Changes from v3

| v3 | v4 |
|---|---|
| `tailwind.config.js` | `@theme` in CSS |
| `content: [...]` paths | Auto-detected from Vite |
| `@apply` in components | Still works, but prefer CVA |
| PostCSS plugin | `@tailwindcss/vite` Vite plugin |
| Arbitrary values `[1.5rem]` | Still works |
| `dark:` variant | Configured via `@custom-variant` |

## Production Optimization

Tailwind v4 generates only used classes by default via Oxide engine — no additional PurgeCSS step needed. Verify final CSS size:

```bash
npm run build && du -sh public/build/assets/*.css
# Target: < 50KB gzipped for typical apps
```

**Safelist dynamic classes** that are constructed at runtime:
```css
/* app.css */
@source "../js/lib/dynamic-colors.ts";  /* ensure these classes are scanned */
```

Or use complete class names (not string concatenation) so Tailwind detects them statically:
```tsx
// ❌ BAD: Tailwind can't detect this
const cls = `bg-${color}-500`

// ✅ GOOD: Full class names in source
const colorMap = { red: 'bg-red-500', green: 'bg-green-500', blue: 'bg-blue-500' }
const cls = colorMap[color]
```
