# ADR-009: Real-Time Broadcasting with Django Channels

**Status**: Accepted
**Date**: 2024-02
**Context**: WebSocket-based real-time data distribution for high-frequency updates

## Problem

Real-time applications (live dashboards, IoT systems, collaboration tools) require pushing data to clients without polling. HTTP polling creates unnecessary load and latency.

## Decision

**Django Channels** with Redis channel layer for real-time broadcasting.

- WebSocket consumers handle connection lifecycle and group membership
- Redis channel layer distributes messages across multiple server instances
- `async_to_sync()` bridges sync Django views to async channel layer
- Group broadcasting via `channel_layer.group_send()` for one-to-many delivery
- In-memory caching (class-level state) for zero-latency reads during high-frequency broadcasts
- Message optimization via array serialization reduces payload 60-70%
- Dual-token authentication: user token + API client token for fine-grained authorization

## Consequences

**Positive**: Sub-second latency, horizontal scaling via Redis, bidirectional communication, Django integration

**Negative**: Requires ASGI server (Daphne/Uvicorn), Redis dependency, connection management complexity

## Related

- [JWT Authentication](002-jwt-auth-patterns.md)
