# ADR-003: Role-Based Permission System

**Status**: Accepted
**Date**: 2024-02
**Context**: Fine-grained permission system for enterprise applications

## Problem

Enterprise applications need route-level permissions (pages accessible to certain roles), action-level permissions (view but not edit), menu adaptation (show only accessible routes), and backend validation. Simple admin/user roles are insufficient.

## Decision

**Two-layer frontend system + three-tier backend enforcement**.

**Frontend Layers:**
1. **General permissions**: Loaded once on app init via `hasPermission(key)` for route guards and menu filtering
2. **Entity actions**: Backend-computed `allowedActions` returned with each entity for action button visibility

**Backend Tiers:**
1. **Group-Based**: Django groups for broad role categories
2. **View-Level**: DRF `permission_classes` for module/feature access
3. **Object-Level**: QuerySet filtering for data isolation (company_id, department, ownership)

Permission format: `ACTION.RESOURCE` (e.g., `VIEW.INVOICE`, `EDIT.INVOICE.APPROVE`)

## Consequences

**Positive**: Simple frontend (just check flags), secure (backend authoritative), fast (permissions cached), flexible (two layers handle different concerns)

**Negative**: Permission key coordination across frontend/backend, entity actions require API call

## Related

- [Workflow State Machines](002-workflow-state-machines.md)
