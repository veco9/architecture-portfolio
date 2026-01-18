# ADR-001: Enterprise Component Library Design

**Status**: Accepted
**Date**: 2024-01
**Context**: Shared UI library for multiple enterprise Vue 3 applications

## Problem

Multiple enterprise applications need consistent UI/UX, but must balance bundle size (avoid shipping unused components), developer experience (good TypeScript support), and flexibility (app-specific customization). Scale: 120+ components, 30+ composables.

## Decision

**Multi-entry point library** with Vite, enabling tree-shaking at module level.

- Two-file component pattern: `Select.ts` (type declarations) + `Select.vue` (implementation)
- TypeScript declarations extend base component props/slots/emits with app-specific additions
- Multi-entry build: each component/composable becomes separate entry point
- Plugin system: `app.use(CoreLibrary, { router, i18nMessages, authScheme })` for consumer setup
- Global permission helpers: `$permission(key)` available in all templates
- Components wrap PrimeVue with app-specific features while passing through all slots

## Consequences

**Positive**: 120+ components shared across applications, full TypeScript with IDE autocompletion, tree-shaking works at module level, single version to manage

**Negative**: Complex build configuration with dts plugin, two files per component, multiple i18n layers to synchronize

## Related

- [Bundle Optimization](002-bundle-optimization.md)
- [Role-Based Permissions](../fullstack/003-role-based-permissions.md)
