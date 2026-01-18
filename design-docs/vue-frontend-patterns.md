# Vue Frontend Patterns

A collection of frontend patterns used across enterprise Vue 3/TypeScript applications.

## Table of Contents

1. [Bundle Optimization Strategy](#1-bundle-optimization-strategy)
2. [Internationalization (i18n)](#2-internationalization-i18n)
3. [TypeScript DTO Transformation](#3-typescript-dto-transformation)
4. [Form Validation Architecture](#4-form-validation-architecture)
5. [Navigation Guards](#5-navigation-guards)
6. [Workflow State Machines (Frontend)](#6-workflow-state-machines-frontend)
7. [Server-Side Data Grid](#7-server-side-data-grid)
8. [Enterprise Component Library](#8-enterprise-component-library)
9. [Role-Based Permissions (Frontend)](#9-role-based-permissions-frontend)

---

## 1. Bundle Optimization Strategy

### Problem
Enterprise SPAs grow to 800KB+ gzipped with heavy dependencies (AG Grid ~450KB, PrimeVue ~150KB, shared library). This impacts load times significantly.

### Solution
Multi-layer optimization achieving 65% bundle reduction.

**Before:** ~800KB monolithic bundle
**After:** ~275KB with lazy loading

**Techniques:**

| Technique | Impact |
|-----------|--------|
| Vendor chunk splitting | Separate Vue, UI, forms, AG Grid |
| Route-level code splitting | `defineAsyncComponent` for views |
| Conditional loading | AG Grid only on routes that need it |
| Peer dependencies | Shared between library and apps |
| Strategic registration | Global for frequent, local for rare |

**Chunk Strategy:**
```
Initial Load (~160KB):
├── vendor-vue.js         ~50KB
├── vendor-ui.js          ~40KB  (core components)
├── main.js               ~30KB  (app shell)
├── vendor-core.js        ~25KB
└── vendor-forms.js       ~15KB

On-Demand (~115KB):
└── ag-grid-chunk.js      ~115KB (loaded when needed)
```

---

## 2. Internationalization (i18n)

### Problem
Multi-language support with locale-aware formatting, runtime language switching, and shared translations between library and apps.

### Solution
Vue I18n with multi-layer message sources.

**Message Hierarchy:**
```
Library messages (base)
    ↓ merged with
Application messages (override/extend)
```

**Implementation:**
```typescript
// Library provides base translations
import { createI18n } from 'vue-i18n'
import libraryMessages from '@corp/ui-library/locales'

// App adds/overrides
import appHr from '@/locales/hr.json'
import appEn from '@/locales/en.json'

const i18n = createI18n({
  messages: {
    hr: { ...libraryMessages.hr, ...appHr },
    en: { ...libraryMessages.en, ...appEn },
  }
})
```

**Formatting:**
- Dates: dayjs with locale
- Numbers/Currency: `Intl.NumberFormat`
- Lazy loading for non-default languages

---

## 3. TypeScript DTO Transformation

### Problem
API responses (DTOs) differ from frontend models: date strings vs Date objects, nested structures, naming conventions.

### Solution
Explicit DTO types with transformation functions.

**Pattern:**
```typescript
// API shape
interface InvoiceDto {
  id: number
  invoice_date: string  // ISO string
  vendor: VendorDto
  status: { code: string; label: string }
}

// Frontend shape
interface Invoice {
  id: number
  invoiceDate: Date     // Date object
  vendor: Vendor
  status: Status
}

// Transform function
function invoiceFromDto(dto: InvoiceDto): Invoice {
  return {
    id: dto.id,
    invoiceDate: new Date(dto.invoice_date),
    vendor: vendorFromDto(dto.vendor),
    status: statusFromDto(dto.status),
  }
}
```

**Benefits:**
- Type-safe at compile time
- Explicit transformation points
- No runtime validation overhead

---

## 4. Form Validation Architecture

### Problem
Complex forms need field validation, cross-field rules, async validation (uniqueness), and server error display.

### Solution
VeeValidate with Yup schemas.

**Schema Definition:**
```typescript
import * as yup from 'yup'

const invoiceSchema = yup.object({
  invoiceNumber: yup.string().required().max(50),
  amount: yup.number().required().positive(),
  dueDate: yup.date().required().min(new Date()),
  vendorId: yup.number().required(),
})
```

**Server Error Integration:**
```typescript
const { setErrors } = useForm()

try {
  await api.submit(data)
} catch (error) {
  if (error.response?.status === 400) {
    // Merge server errors with form
    setErrors(error.response.data.fields)
  }
}
```

**Custom Rules:**
```typescript
// Croatian OIB validation
yup.addMethod(yup.string, 'oib', function() {
  return this.test('oib', 'Invalid OIB', validateOibChecksum)
})
```

---

## 5. Navigation Guards

### Problem
SPA navigation must handle auth checks, authorization, data preloading, unsaved changes warnings, and tenant validation.

### Solution
Composable guard chain with `beforeEach`.

**Guard Order:**
```
auth → tenant → permissions → data loading
```

**Route Meta:**
```typescript
const routes = [
  {
    path: '/invoices',
    component: InvoicesView,
    meta: {
      requiresAuth: true,
      permission: 'VIEW.INVOICE',
    }
  }
]
```

**Guard Implementation:**
```typescript
router.beforeEach(async (to, from) => {
  // 1. Auth check
  if (to.meta.requiresAuth && !isAuthenticated()) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }

  // 2. Permission check
  if (to.meta.permission && !hasPermission(to.meta.permission)) {
    return { name: 'forbidden' }
  }

  return true
})
```

**Unsaved Changes:**
```typescript
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    return confirm('Discard unsaved changes?')
  }
})
```

---

## 6. Workflow State Machines (Frontend)

### Problem
Enterprise workflows have 10-15+ statuses with complex transitions. Frontend should not duplicate business logic for action availability.

### Solution
Backend-driven actions with frontend rendering.

**API Response:**
```typescript
interface Invoice {
  id: number
  status: { code: string; label: string }
  // Backend computes this
  allowedActions: {
    canEdit: boolean
    canApprove: boolean
    canReject: boolean
    canDelete: boolean
  }
}
```

**Frontend Rendering:**
```vue
<template>
  <Button
    v-if="invoice.allowedActions.canApprove"
    label="Approve"
    @click="handleApprove"
  />
  <Button
    v-if="invoice.allowedActions.canReject"
    label="Reject"
    @click="handleReject"
  />
</template>
```

**Benefits:**
- Single source of truth (backend)
- No business logic in frontend
- Secure (can't bypass via UI)

---

## 7. Server-Side Data Grid

### Problem
Client-side grids can't handle 100K+ records. Need filtering, sorting, grouping, and aggregations with sub-second response on millions of records.

### Solution
AG Grid Enterprise Server-Side Row Model with standardized protocol.

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│  AG Grid (Frontend)                                             │
│  - Requests data on scroll, filter, sort                        │
│  - Sends IServerSideGetRowsRequest                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│  Django Backend                                                 │
│  - Translates AG Grid filters to ORM queries                    │
│  - Handles grouping, aggregation                                │
│  - Returns paginated results                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Datasource Composable:**
```typescript
import type { IServerSideDatasource, IServerSideGetRowsParams } from 'ag-grid-community'

export function useGridDatasource<T>(apiService: GridApiService<T>) {
  const loading = ref(false)

  const datasource: IServerSideDatasource = {
    getRows: async (params: IServerSideGetRowsParams) => {
      loading.value = true
      try {
        const response = await apiService.fetchGridData(params.request)
        params.success({ rowData: response.data, rowCount: response.totalCount })
      } catch (e) {
        params.fail()
      } finally {
        loading.value = false
      }
    }
  }

  return { datasource, loading }
}
```

**Filter Translation (Backend):**
```python
# AG Grid filter types → Django ORM
FILTER_MAP = {
    'equals': '__iexact',
    'contains': '__icontains',
    'startsWith': '__istartswith',
    'lessThan': '__lt',
    'greaterThan': '__gt',
    'inRange': '__range',
}

def apply_filters(queryset, filter_model):
    for field, config in filter_model.items():
        lookup = FILTER_MAP[config['type']]
        queryset = queryset.filter(**{f'{field}{lookup}': config['filter']})
    return queryset
```

**Server-Side Selection:**
```typescript
// Flatten AG Grid's hierarchical selection for backend
interface FlattenedSelection {
  selectAll: boolean
  selectedKeys: string[][]   // When selectAll=false
  excludedKeys: string[][]   // When selectAll=true
}
```

---

## 8. Enterprise Component Library

### Problem
Multiple applications need shared UI components with consistent behavior, TypeScript support, and tree-shaking.

### Solution
Multi-entry point Vite library with two-file component pattern.

**Two-File Pattern:**
```
components/
├── Select.ts      # Type declarations (props, slots, emits)
└── Select.vue     # Implementation
```

**Select.ts (Types):**
```typescript
import type { DefineComponent } from 'vue'

export interface SelectProps {
  modelValue: string | number | null
  options: SelectOption[]
  placeholder?: string
  disabled?: boolean
  loading?: boolean
}

export interface SelectSlots {
  option: (props: { option: SelectOption }) => any
  empty: () => any
}

export interface SelectEmits {
  'update:modelValue': [value: string | number | null]
  'change': [value: string | number | null]
}

declare const Select: DefineComponent<SelectProps, {}, {}, {}, {}, {}, {}, SelectEmits>
export default Select
```

**Multi-Entry Build (vite.config.ts):**
```typescript
export default defineConfig({
  build: {
    lib: {
      entry: {
        'index': 'src/index.ts',
        'components/Select': 'src/components/Select.ts',
        'components/DataTable': 'src/components/DataTable.ts',
        'composables/useFormField': 'src/composables/useFormField.ts',
        // ... 120+ entries
      },
      formats: ['es']
    },
    rollupOptions: {
      external: ['vue', 'vue-router', 'pinia'],
    }
  }
})
```

**Plugin System:**
```typescript
// Library exports plugin for consumer setup
export const CoreLibrary = {
  install(app: App, options: LibraryOptions) {
    // Register global components
    app.component('IaSelect', Select)
    app.component('IaButton', Button)

    // Provide global config
    app.provide('libraryConfig', options)

    // Add global properties
    app.config.globalProperties.$permission = (key: string) =>
      options.authScheme.hasPermission(key)
  }
}

// Consumer usage
app.use(CoreLibrary, {
  router,
  i18nMessages: customMessages,
  authScheme: myAuthScheme,
})
```

---

## 9. Role-Based Permissions (Frontend)

### Problem
Frontend needs route-level permissions (page access), action-level permissions (buttons), and menu filtering without duplicating backend logic.

### Solution
Two-layer permission system with backend as source of truth.

**Layer 1: General Permissions (App-level)**
```typescript
// Loaded once on app init
const authStore = useAuthStore()
await authStore.loadPermissions()

// Usage anywhere
const canViewInvoices = authStore.hasPermission('VIEW.INVOICE')
```

**Layer 2: Entity Actions (Per-record)**
```typescript
// Returned with each entity from API
interface Invoice {
  id: number
  // ... fields
  allowedActions: {
    canEdit: boolean
    canApprove: boolean
    canDelete: boolean
  }
}
```

**Route Guard Integration:**
```typescript
const routes = [
  {
    path: '/invoices',
    component: InvoicesView,
    meta: { permission: 'VIEW.INVOICE' }
  },
  {
    path: '/invoices/:id/edit',
    component: InvoiceEditView,
    meta: { permission: 'EDIT.INVOICE' }
  }
]

router.beforeEach((to) => {
  const permission = to.meta.permission
  if (permission && !authStore.hasPermission(permission)) {
    return { name: 'forbidden' }
  }
})
```

**Menu Filtering:**
```typescript
const menuItems = computed(() =>
  allMenuItems.filter(item =>
    !item.permission || authStore.hasPermission(item.permission)
  )
)
```

**Template Usage:**
```vue
<template>
  <!-- General permission -->
  <MenuItem v-if="$permission('VIEW.REPORTS')" label="Reports" />

  <!-- Entity action -->
  <Button v-if="invoice.allowedActions.canApprove" label="Approve" />
</template>
```

---

## Summary

| Pattern | Use Case |
|---------|----------|
| Bundle Optimization | Large SPAs with heavy dependencies |
| Multi-layer i18n | Shared library + app translations |
| DTO Transformation | Type-safe API integration |
| Yup + VeeValidate | Complex form validation |
| Navigation Guards | Auth, permissions, route protection |
| Backend-Driven Actions | Complex workflow state machines |
| Server-Side Grid | 1M+ records with AG Grid |
| Component Library | 120+ shared components with TypeScript |
| Role-Based Permissions | Route + action level access control |
