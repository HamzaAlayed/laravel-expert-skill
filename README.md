# laravel-expert

A Claude Code skill for principal-level Laravel architecture, advanced design patterns, and full-stack performance engineering.

## What It Covers

This skill extends [`laravel-specialist`](https://github.com/Jeffallan/laravel-specialist) — it assumes all specialist knowledge and starts above that waterline.

| Reference | Topics |
|---|---|
| `references/architecture.md` | DDD, bounded contexts, aggregate roots, value objects, hexagonal/clean architecture, Pest fitness tests |
| `references/cqrs-eventsourcing.md` | CQRS write/read model separation, command bus, projectors, event store, snapshots |
| `references/performance.md` | Laravel Octane (Swoole/FrankenPHP/RoadRunner), memory leaks, concurrency, EXPLAIN ANALYZE, k6 load testing |
| `references/caching.md` | Multi-layer L1/L2/L3 cache, stampede prevention, Redis pipelines, Lua scripts, tag invalidation |
| `references/scaling.md` | Read replicas, connection pooling, table partitioning, Redis Cluster vs Sentinel, zero-downtime deploys |
| `references/api.md` | API versioning, rate limiting, OpenAPI (Scramble), API contracts, auth strategies, webhooks |
| `references/reactjs.md` | Inertia.js, typed props, `useForm`, shared data, SSR, Zustand, compound components |
| `references/tailwindcss.md` | Tailwind v4, `@theme` design system, CVA typed variants, dark mode, production optimization |
| `references/vite.md` | Laravel Vite plugin, path aliases, code splitting, HMR, bundle analysis, Ziggy typed routes |

## When to Use

**Use `laravel-expert` when:**
- Designing bounded contexts, aggregates, or domain models
- Implementing CQRS, event sourcing, or projection systems
- Tuning Laravel Octane or diagnosing production performance issues
- Architecting multi-layer caching or Redis-at-scale strategies
- Planning horizontal scaling, read replicas, or zero-downtime deployments
- Building advanced REST APIs with versioning, rate limiting, or OpenAPI docs
- Integrating React with Inertia.js, including SSR
- Setting up Tailwind v4 design systems or typed component variants
- Optimizing Vite builds, code splitting, or SSR bundles

**Use `laravel-specialist` instead for:**
- Standard Eloquent ORM, API resources, Livewire
- Sanctum auth, Horizon queue setup, Pest feature tests
- Basic REST patterns and CRUD APIs

## Installation

### Claude Code (recommended)

```bash
# Clone into your agents skills directory
git clone git@github.com:HamzaAlayed/laravel-expert-skill.git \
    ~/.agents/skills/laravel-expert

# Symlink into Claude's skills directory
ln -s ~/.agents/skills/laravel-expert ~/.claude/skills/laravel-expert
```

### Other agents

```bash
git clone git@github.com:HamzaAlayed/laravel-expert-skill.git \
    ~/.agents/skills/laravel-expert
```

Point your agent's skills directory to `~/.agents/skills/`.

## Usage

Once installed, Claude Code automatically loads this skill when your prompt matches its triggers. You can also invoke it explicitly:

```
Use laravel-expert to design the order management bounded context
```

```
Using laravel-expert, set up multi-layer caching for the product catalog API
```

```
With laravel-expert, configure Octane with Swoole and audit our service providers
```

## Stack Requirements

- Laravel 11+
- PHP 8.2+
- Node 20+ / npm 10+
- React 18+
- Tailwind CSS v4
- Vite 6+

## License

MIT
