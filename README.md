# laravel-expert

The most comprehensive Laravel skill available — covers the entire Laravel ecosystem from advanced architecture to full-stack frontend, real-time systems, security, and the broader package ecosystem.

## Install

```bash
npx skills add HamzaAlayed/laravel-expert-skill
```

## What It Covers (17 Reference Files)

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
| `references/broadcasting.md` | Laravel Reverb, Echo, private/presence channels, SSE, model broadcasting, notifications |
| `references/security.md` | OWASP Top 10, CSP, gates/policies, encryption, audit logging, SSRF prevention, rate limiting |
| `references/testing.md` | Unit, feature, mocking (mail/jobs/events/HTTP), Dusk, mutation testing, architecture tests, contracts |
| `references/package-development.md` | Service providers, facades, Testbench, config/migrations, CI matrix, Packagist publishing |
| `references/packages.md` | Spatie suite, Scout + Meilisearch, Cashier billing, Pennant feature flags |
| `references/ai.md` | Prism multi-provider LLM, streaming, tool calling, structured output, embeddings, RAG, queued AI jobs |
| `references/mcp.md` | Build Laravel as an MCP server: tools, resources, prompts, HTTP/stdio transport, Claude integration |

## When to Use

**Use `laravel-expert` for any advanced Laravel topic:**

- Architecture: DDD, CQRS, event sourcing, hexagonal/clean architecture
- Performance: Octane, Redis at scale, multi-layer caching, DB partitioning
- Frontend: React + Inertia.js, TailwindCSS v4, Vite, SSR
- API: Versioning, rate limiting, OpenAPI, webhooks, auth strategies
- Real-time: Reverb WebSockets, Echo, presence channels, SSE
- Admin: Filament 3 panels, resources, widgets, multi-tenancy
- Security: OWASP hardening, CSP, policies, encryption, audit logs
- Testing: Architecture tests, mutation testing, Dusk, contract tests
- Ecosystem: Spatie packages, Scout/search, Cashier, Pennant
- Testing: Unit, feature, Dusk, mutation, architecture tests, mocking, contracts
- Package Development: Service providers, Testbench, Packagist, CI, versioning
- AI: Prism LLM client, streaming, tool calling, RAG pipelines, embeddings
- MCP: Expose Laravel as an MCP server for Claude Code / Claude Desktop

**Use `laravel-specialist` for basics:**
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
