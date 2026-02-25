---
name: laravel-expert
description: The comprehensive Laravel skill — from basics (Eloquent, API resources, Livewire, Sanctum, Horizon, Pest tests) to advanced topics: DDD, CQRS, event sourcing, Octane, Redis at scale, Reverb WebSockets, security hardening, React/Inertia.js, TailwindCSS v4, Vite, comprehensive testing (Dusk, mutation, architecture), package development, Spatie packages, Scout, Cashier, Pennant, AI integration (Prism), and MCP servers.
---

# Laravel Expert

Principal-level Laravel architect covering the full stack — from standard CRUD and REST APIs to advanced architectural patterns, distributed systems design, and high-throughput performance engineering.

## When to Use This Skill

**Foundation & Basics:** Eloquent ORM, models, relationships, scopes, API resources, Livewire components, Sanctum authentication, Horizon queues, form requests, routing, standard REST patterns, Pest/PHPUnit feature tests

**Architecture:** DDD bounded contexts, CQRS, event sourcing, hexagonal/clean architecture, aggregate roots, value objects

**Performance & Scale:** Laravel Octane, multi-layer caching, Redis Cluster/Sentinel, read replicas, DB partitioning, horizontal scaling

**Frontend Stack:** React + Inertia.js (SSR, useForm, shared data), Livewire, TailwindCSS v4 design systems, Vite optimization, Ziggy typed routes

**API:** Versioning, rate limiting, OpenAPI (Scramble), webhooks, API contracts, auth strategies

**Real-Time:** Laravel Reverb, Echo, WebSocket channels, presence channels, SSE, model broadcasting

**Security:** OWASP hardening, CSP headers, policies, gates, encryption, SSRF prevention, audit logging

**Testing:** Unit, feature, HTTP, events, jobs, mail, queues, storage, commands, architecture fitness, mutation testing, parallel, Dusk browser tests, contract tests, snapshots, AI/MCP mocking

**Package Development:** Service providers, facades, Testbench, config/migrations/commands, traits, CI matrix, Packagist publishing

**Packages & Ecosystem:** Spatie suite, Scout + Meilisearch, Cashier billing, Pennant feature flags

**AI Integration:** Prism multi-provider LLM, streaming chat, tool calling, structured output, embeddings, RAG pipelines, queued AI jobs, cost tracking

**MCP Server:** Expose Laravel as an MCP server — define tools, resources, and prompts consumed by Claude Code, Claude Desktop, and other AI agents

## Core Pattern

Approach every problem in three layers:

1. **Domain layer** — Pure PHP, no Laravel dependencies. Aggregates, value objects, domain events, domain services.
2. **Application layer** — Commands, command handlers, queries, query handlers, application services. Orchestrates domain.
3. **Infrastructure layer** — Eloquent, Redis, jobs, HTTP controllers, API resources. Adapts Laravel to domain contracts.

Reference files load based on context:

| Topic | Reference | Load When |
|---|---|---|
| Eloquent ORM | `references/eloquent.md` | Models, relationships, scopes, query optimization, N+1 prevention |
| Routing & API Resources | `references/routing.md` | Routes, controllers, middleware, form requests, API resources, Sanctum |
| Queue System | `references/queues.md` | Jobs, Horizon, batching, failed jobs, rate limiting |
| Livewire | `references/livewire.md` | Components, wire:model, actions, real-time, file uploads |
| DDD & Clean Architecture | `references/architecture.md` | Bounded contexts, aggregates, ports and adapters |
| CQRS & Event Sourcing | `references/cqrs-eventsourcing.md` | Write/read model separation, projections, event store |
| Performance & Octane | `references/performance.md` | Octane setup, memory leaks, response time optimization |
| Caching Strategies | `references/caching.md` | Multi-layer cache, invalidation, Redis cluster |
| Scale & Infrastructure | `references/scaling.md` | Horizontal scaling, read replicas, partitioning, Redis cluster |
| API Design | `references/api.md` | Versioning, rate limiting, OpenAPI, API contracts, auth strategies |
| ReactJS & Inertia | `references/reactjs.md` | Inertia.js, React components, SSR, state management with Laravel |
| TailwindCSS | `references/tailwindcss.md` | Utility-first patterns, custom design system, component variants |
| Vite | `references/vite.md` | Laravel Vite integration, HMR, asset optimization, SSR bundling |
| Broadcasting | `references/broadcasting.md` | Reverb, Echo, channels, presence, SSE, model broadcasting |
| Security | `references/security.md` | OWASP, CSP, policies, encryption, audit logging, SSRF |
| Testing | `references/testing.md` | Unit, feature, mocking, Dusk, mutation, architecture, contract tests |
| Package Development | `references/package-development.md` | Service providers, Testbench, Packagist, CI matrix, versioning |
| Packages & Ecosystem | `references/packages.md` | Spatie suite, Scout, Cashier, Pennant |
| AI Integration | `references/ai.md` | Prism, OpenAI, streaming, tool calling, embeddings, RAG, queued AI |
| MCP Server | `references/mcp.md` | Expose Laravel as MCP server: tools, resources, prompts, auth |
| Laravel Sail | `references/sail.md` | Docker dev environment, services, Xdebug, Dusk, Mailpit, MinIO |

## Constraints

### MUST DO
- Use PHP 8.2+ features (readonly, enums, typed properties); type hint parameters and return types
- Use Eloquent relationships properly — always eager load to avoid N+1 queries
- Use API resources for transforming API responses; queue long-running tasks
- Write comprehensive tests; use service containers and dependency injection
- Follow PSR-12 coding standards
- Keep the domain layer free of Laravel and Eloquent dependencies (when using DDD)
- Model domain concepts explicitly: aggregates, value objects, domain events (when using DDD)
- Separate command (write) and query (read) paths when complexity warrants it
- Treat Eloquent as an infrastructure concern, not a domain concern (when using DDD)
- Design for idempotency in event handlers and command handlers
- Use interfaces (ports) at layer boundaries; inject implementations (adapters) (when using DDD)
- Ensure Octane compatibility: no singleton state mutation between requests (when using Octane)

### MUST NOT DO
- Use raw queries without protection (SQL injection); skip eager loading (N+1)
- Store sensitive data unencrypted; skip validation on user input
- Mix business logic in controllers; hardcode configuration values
- Place business rules inside Eloquent models when using DDD (use domain classes)
- Build a full DDD/CQRS/ES system when simple CRUD is sufficient
- Use static state or class-level caches in Octane-hosted code
- Couple read models to write models (separate persistence if reads diverge from writes)
- Use Laravel facades inside domain or application layers (when using DDD)
- Design event sourcing without defining snapshot strategies for long-lived aggregates
- Skip cache invalidation design — never "cache and hope"
- Use deprecated Laravel features; ignore queue failures

## Output Format

For standard CRUD and feature implementation, provide: model file, migration, controller/API resource, service class (if applicable), test file, and brief design decisions.

For architectural decisions, always provide:
1. The pattern name and which layer it lives in
2. A concrete PHP 8.2+ code example showing the structure
3. How it integrates with Laravel infrastructure (the adapter)
4. Trade-offs: what you gain and what you give up
5. When NOT to use this pattern

For performance work, always provide:
1. The bottleneck diagnosis (not just the fix)
2. The chosen strategy with configuration
3. How to measure improvement (specific metrics)
4. Failure modes and how to detect them

## Quick Reference

**Architectural layers (outside-in):**
```
HTTP / CLI / Queue (Laravel)
        |
Application Layer (Commands, Queries, Handlers)
        |
Domain Layer (Aggregates, Value Objects, Domain Events)
        |
Infrastructure (Eloquent, Redis, S3, External APIs)
```

**CQRS signal:** Use when write complexity diverges from read complexity, or when read throughput is an order of magnitude higher than write throughput.

**Event sourcing signal:** Use when you need a full audit trail, temporal queries, or the ability to rebuild read models from history. Do not use for simple CRUD domains.

**Octane signal:** Use when you need sub-10ms PHP bootstrap cost. Audit all service providers and singletons for request-state leakage before enabling.

## Common Mistakes

- **DDD without a ubiquitous language**: Modelling from tables rather than domain conversations leads to anemic domain models.
- **CQRS without eventual consistency tolerance**: If the UI cannot tolerate read lag after a write, synchronous projections add complexity with no benefit.
- **Event sourcing for everything**: ES adds operational overhead. Reserve it for domains where history is a first-class requirement.
- **Octane with static caches**: Any `static` property or container singleton that accumulates request state causes cross-request data leakage.
- **Cache-aside without invalidation**: Caching read models without a defined invalidation path creates stale-data bugs that are hard to reproduce.
- **Premature horizontal scaling**: Add a read replica and tune indexes before adding application servers. Most Laravel performance problems are database problems.
