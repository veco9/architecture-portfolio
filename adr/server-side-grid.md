# ADR-001: Server-Side Data Grid Patterns

**Status**: Accepted
**Date**: 2024-01
**Context**: Data-intensive dashboards handling 1M+ records

## Problem

Client-side processing becomes impractical beyond ~10K records. Need to handle filtering, sorting, grouping, pivoting, and aggregations on datasets ranging from 100K to 10M+ records with sub-second response times.

## Decision

**Server-Side Row Model** with AG Grid Enterprise.

- Standardized request/response protocol matching AG Grid's `IServerSideGetRowsRequest`
- Backend translates AG Grid filter model to ORM lookups (text/number/date/set filters)
- Hierarchical grouping: empty `groupKeys` returns top-level, populated keys drill down
- Aggregations computed via database GROUP BY with Sum/Avg/Count/Min/Max
- Server-side selection state: flatten AG Grid's hierarchical selection to portable format for bulk operations
- Exports bypass grid, stream directly from server without pagination

## Consequences

**Positive**: Constant memory footprint, supports all operations at any scale, 50-200ms response for 1M records

**Negative**: Every interaction requires network round-trip, complex API contract, requires database optimization

## Related

- [Component Library](../frontend/001-component-library-design.md)
