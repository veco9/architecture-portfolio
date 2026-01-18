# System Design Portfolio

A collection of architectural patterns, design decisions, and trade-off analyses from 7+ years of building enterprise applications.

## About This Repository

This repository documents architectural patterns and decision-making frameworks developed while building large-scale enterprise systems. All examples are abstracted and generalized - no proprietary code or client-specific details.

**Focus areas:**
- Data-intensive applications (1M+ records, real-time dashboards)
- Multi-tenant SaaS architectures
- Payment processing & financial systems
- Enterprise authentication & authorization
- Frontend performance & component libraries

## Repository Structure

```
├── adr/                           # Architecture Decision Records (focused)
│   ├── payment-gateway.md
│   ├── realtime-broadcasting.md
│   ├── server-side-grid.md
│   ├── clean-architecture.md
│   ├── component-library.md
│   ├── pessimistic-locking.md
│   ├── role-based-permissions.md
│   └── multi-tenant.md
│
└── design-docs/                   # Comprehensive pattern documentation
    ├── django-backend-patterns.md
    ├── vue-frontend-patterns.md
    ├── testing-strategy.md
    ├── dynamic-sql-query-builder.md
    ├── multi-phase-workflow-engine.md
    └── configurable-dashboard-layout.md
```

## Architecture Decision Records

Short, focused records of key architectural decisions.

| ADR | Domain | Description |
|-----|--------|-------------|
| [Payment Gateway](adr/payment-gateway.md) | Backend | Multi-gateway abstraction, dual-state transactions, webhook handling |
| [Real-Time Broadcasting](adr/realtime-broadcasting.md) | Backend | Django Channels, WebSockets, Redis channel layer |
| [Server-Side Grid](adr/server-side-grid.md) | Fullstack | AG Grid with 1M+ records, filter translation, selection state |
| [Clean Architecture](adr/clean-architecture.md) | Backend | Services-Selectors-Views pattern for Django |
| [Component Library](adr/component-library.md) | Frontend | 120+ components, two-file pattern, multi-entry Vite build |
| [Pessimistic Locking](adr/pessimistic-locking.md) | Backend | Concurrent processing with `select_for_update(skip_locked=True)` |
| [Role-Based Permissions](adr/role-based-permissions.md) | Fullstack | Two-layer frontend + three-tier backend enforcement |
| [Multi-Tenant](adr/multi-tenant.md) | Backend | Row-level isolation, middleware context, user-tenant junction |

## Design Documents

Comprehensive documentation of patterns and implementations.

| Document | Description |
|----------|-------------|
| [Django Backend Patterns](design-docs/django-backend-patterns.md) | 11 patterns: JWT auth, caching, payments, real-time, clean architecture, locking |
| [Vue Frontend Patterns](design-docs/vue-frontend-patterns.md) | 9 patterns: bundles, i18n, forms, grids, component library, permissions |
| [Testing Strategy](design-docs/testing-strategy.md) | Full-stack testing: pytest, Vitest, CI/CD integration |
| [Dynamic SQL Query Builder](design-docs/dynamic-sql-query-builder.md) | Database-agnostic query generation with permission-aware filtering |
| [Multi-Phase Workflow Engine](design-docs/multi-phase-workflow-engine.md) | Frontend architecture for complex approval workflows |
| [Configurable Dashboard Layout](design-docs/configurable-dashboard-layout.md) | Drag-and-drop widget system with persistence |

## Technologies

**Backend:** Django, Python, Django REST Framework, Oracle, PostgreSQL, Redis, Celery, RabbitMQ, RESTful APIs, JWT/OAuth2/OIDC, LDAP/AD, pytest

**Frontend:** Vue 3 (Composition API), TypeScript, JavaScript, Pinia, AG Grid Enterprise, AG Charts Enterprise, PrimeVue, Vite, i18n, Storybook, Vee-Validate, Vitest

**DevOps & Tools:** gunicorn, supervisor, npm/PyPI registry, SSH/rsync automation, GitLab CI/CD, Docker, nginx, Git, Jira, Confluence, Payment gateways, Sign-On APIs (Google, Facebook)

## Note

This repository demonstrates architectural thinking and decision-making patterns. It is a clean-room documentation of general concepts, independent of any proprietary implementations.
