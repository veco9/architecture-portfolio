# ADR-001: Multi-Tenant Architecture

**Status**: Accepted
**Date**: 2024-01
**Context**: Enterprise SaaS platforms serving multiple organizations

## Problem

SaaS platforms must isolate tenant data while balancing operational complexity, cost efficiency, and performance. Options include database-per-tenant (strongest isolation), schema-per-tenant (moderate), or row-level filtering (simplest infrastructure).

## Decision

**Row-level tenant isolation** with middleware-enforced context.

- Every table includes `tenant_id` column
- Middleware extracts tenant from JWT, validates membership, sets thread-local context
- Base QuerySet automatically filters by current tenant
- User-tenant junction table supports multi-tenant access with different roles

## Consequences

**Positive**: Single database to manage, easy cross-tenant analytics, fast tenant onboarding

**Negative**: Query overhead (mitigated by composite indexes), shared resource contention, requires tenant filter verification in code reviews

## When to Reconsider

Use database-per-tenant when regulatory compliance requires physical separation, tenants have vastly different scale, or per-tenant schema customization is needed.

## Related

- [JWT Authentication](002-jwt-auth-patterns.md)
