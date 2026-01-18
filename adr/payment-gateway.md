# ADR-006: Payment Gateway Integration

**Status**: Accepted
**Date**: 2024-02
**Context**: Multi-gateway payment processing with fee calculation and reconciliation

## Problem

High-volume payment platforms (1M+ users) need robust payment processing across multiple gateways, complex fee calculations, webhook handling, and reconciliation with gateway settlements. Without abstraction, gateway-specific code spreads throughout the application.

## Decision

**Gateway abstraction layer** with adapter pattern.

- Unified interface: `initiate_payment()`, `capture()`, `refund()`, `get_status()`, `verify_webhook()`
- Gateway-specific adapters implement the interface
- Dual-state transaction model: internal state (CREATED→COMPLETED) mapped to gateway state (initialized→settled)
- Idempotency via unique keys prevents duplicate payments
- Fee calculation engine handles percentage + fixed fees, tiers, minimums/maximums
- Webhook processing with HMAC signature verification and atomic updates
- Daily reconciliation matches payments with gateway settlements

## Consequences

**Positive**: Gateway switching without business logic changes, comprehensive audit trail, fee calculation centralized, high volume support

**Negative**: Abstraction may not fit all gateway features, multiple failure modes

## Related

- [Audit Trail](007-audit-trail-compliance.md)
