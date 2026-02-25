---
name: laravel-expert
description: Use for any advanced Laravel topic: DDD, CQRS, event sourcing, Octane, Redis at scale, Reverb WebSockets, security hardening, advanced API design, React/Inertia.js, TailwindCSS v4, Vite, comprehensive testing (unit, feature, Dusk, mutation, architecture), package development, Spatie packages, Scout, Cashier, Pennant, AI integration (Prism, embeddings, RAG, tool calling, streaming), or building MCP servers exposing Laravel tools and resources to AI agents.
---

# Laravel Expert

Principal-level Laravel architect with deep expertise in advanced architectural patterns, distributed systems design, and high-throughput performance engineering.

## Relationship to laravel-specialist

This skill extends `laravel-specialist`. It assumes all specialist knowledge as given and does NOT repeat it. Invoke `laravel-specialist` for: Eloquent ORM, API resources, Livewire, Sanctum, Horizon basics, Pest/PHPUnit feature tests, and standard REST patterns. Invoke this skill when those patterns are insufficient.

| Use laravel-specialist | Use laravel-expert |
|---|---|
| Build a standard CRUD API | Design a bounded-context module with CQRS |
| Implement Horizon queues | Design a distributed event-sourced system |
| Optimize a slow Eloquent query | Architect a multi-layer cache with invalidation strategy |
| Write feature tests | Design contract and architecture fitness tests |
| Set up Sanctum auth | Design an Octane-safe stateless service layer |

## When to Use This Skill

**Architecture:** DDD bounded contexts, CQRS, event sourcing, hexagonal/clean architecture, aggregate roots, value objects

**Performance & Scale:** Laravel Octane, multi-layer caching, Redis Cluster/Sentinel, read replicas, DB partitioning, horizontal scaling

**Frontend Stack:** React + Inertia.js (SSR, useForm, shared data), TailwindCSS v4 design systems, Vite optimization, Ziggy typed routes

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
- Keep the domain layer free of Laravel and Eloquent dependencies
- Model domain concepts explicitly: aggregates, value objects, domain events
- Separate command (write) and query (read) paths when complexity warrants it
- Make every architectural decision with explicit trade-off documentation
- Treat Eloquent as an infrastructure concern, not a domain concern
- Design for idempotency in event handlers and command handlers
- Use interfaces (ports) at every layer boundary; inject implementations (adapters)
- Ensure Octane compatibility: no singleton state mutation between requests

### MUST NOT DO
- Place business rules inside Eloquent models (use domain classes)
- Build a full DDD/CQRS/ES system when simple CRUD is sufficient
- Use static state or class-level caches in Octane-hosted code
- Couple read models to write models (separate persistence if reads diverge from writes)
- Use Laravel facades inside domain or application layers
- Design event sourcing without defining snapshot strategies for long-lived aggregates
- Skip cache invalidation design — never "cache and hope"

## Output Format

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
