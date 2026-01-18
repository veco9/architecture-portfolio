# ADR-010: Clean Architecture for Django APIs

**Status**: Accepted
**Date**: 2024-02
**Context**: Organizing Django applications with clear separation of concerns

## Problem

Django projects often evolve into "fat views" or "fat models" anti-patterns where business logic scatters across view methods or bloats model classes. Without architectural guidelines, teams make inconsistent decisions leading to technical debt.

## Decision

**Services-Selectors-Views** pattern with strict layer separation.

- **Selectors**: Read operations returning QuerySets. No side effects.
- **Services**: Write operations wrapped in `@transaction.atomic`. Business logic lives here.
- **Views**: HTTP handling and orchestration only. Calls selectors for reads, services for writes.
- **Models**: Dumb data containers with field definitions and relationships. No business logic.

For large applications (100+ models), organize into three Django apps: `api/` (views + serializers), `services/` (business logic), `models/` (data layer).

## Consequences

**Positive**: Services testable without HTTP layer, logic reusable from views/tasks/commands, clear boundaries, explicit dependencies via keyword arguments

**Negative**: More files than traditional Django, learning curve, potential over-engineering for simple CRUD

## Related

- [Role-Based Permissions](../fullstack/003-role-based-permissions.md)
- [Pessimistic Locking](011-pessimistic-locking.md)
