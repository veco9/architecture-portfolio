# ADR-011: Pessimistic Locking for Concurrent Processing

**Status**: Accepted
**Date**: 2024-02
**Context**: Safe concurrent processing of shared resources across multiple workers

## Problem

Multi-worker systems processing shared data (email queues, job processors, payment processing) face race conditions. Without proper locking, workers may process the same record simultaneously, causing data corruption or duplicate operations.

## Decision

**Pessimistic locking** with `select_for_update(skip_locked=True)`.

- Two-phase query pattern: find candidate without lock (fast), then lock specific row
- `skip_locked=True` allows workers to skip locked rows and process others
- Lock held within transaction, released on commit/rollback
- Stale claim cleanup for crashed workers via periodic job
- `nowait=True` variant for fail-fast scenarios

Database support: PostgreSQL 9.5+, MySQL 8.0+, Oracle (not SQLite).

## Consequences

**Positive**: Each record processed exactly once, workers don't block each other, atomic with database operations, crash-safe

**Negative**: Database-specific syntax, locks held during processing (keep transactions short), potential starvation

## Related

- [Clean Architecture](010-clean-architecture-django.md) - Services wrap locking logic
