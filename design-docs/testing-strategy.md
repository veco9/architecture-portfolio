# Testing Strategy

Comprehensive testing approach for full-stack Django + Vue applications.

## Table of Contents

- [Backend Testing (Django/Python)](#backend-testing-djangopython)
- [Frontend Testing (Vue/TypeScript)](#frontend-testing-vuetypescript)
- [CI/CD Integration](#cicd-integration)
- [Summary](#summary)

---

## Backend Testing (Django/Python)

### Test Pyramid

```
         ╱╲
        ╱  ╲         E2E (few)
       ╱────╲        - Critical user journeys
      ╱      ╲       - API contract tests
     ╱────────╲      Integration (some)
    ╱          ╲     - Full request/response
   ╱────────────╲    - Database operations
  ╱              ╲   Unit (many)
 ╱────────────────╲  - Services, selectors
╱                  ╲ - Business logic
```

### Stack

| Tool | Purpose |
|------|---------|
| pytest | Test runner |
| pytest-django | Django integration |
| FactoryBoy | Test data generation |
| responses / httpretty | HTTP mocking |
| freezegun | Time mocking |

### Unit Tests

Test services and selectors with mocked dependencies.

```python
# tests/services/test_invoice_service.py
import pytest
from unittest.mock import Mock, patch
from services.invoice import invoice_create

class TestInvoiceCreate:
    def test_creates_invoice_with_items(self):
        # Arrange
        vendor = Mock(id=1, company_id=10)

        # Act
        invoice = invoice_create(
            company_id=10,
            vendor_id=1,
            invoice_number='INV-001',
            items=[{'product_id': 1, 'quantity': 2, 'unit_price': '10.00'}],
            created_by_id=5,
        )

        # Assert
        assert invoice.invoice_number == 'INV-001'
        assert invoice.amount == Decimal('20.00')

    def test_raises_on_invalid_vendor(self):
        with pytest.raises(ValidationError):
            invoice_create(vendor_id=999, ...)
```

### Integration Tests

Full request/response cycle with test database.

```python
# tests/api/test_invoice_api.py
import pytest
from rest_framework.test import APIClient

@pytest.mark.django_db
class TestInvoiceAPI:
    def setup_method(self):
        self.client = APIClient()
        self.user = UserFactory(company_id=10)
        self.client.force_authenticate(self.user)

    def test_list_invoices_filtered_by_tenant(self):
        # Create invoices for different companies
        InvoiceFactory(company_id=10)  # user's company
        InvoiceFactory(company_id=20)  # different company

        response = self.client.get('/api/v1/invoices/')

        assert response.status_code == 200
        assert len(response.data['results']) == 1  # only own company

    def test_create_invoice_returns_201(self):
        vendor = VendorFactory(company_id=10)

        response = self.client.post('/api/v1/invoices/', {
            'vendor_id': vendor.id,
            'invoice_number': 'INV-001',
            'items': [...]
        })

        assert response.status_code == 201
        assert Invoice.objects.count() == 1
```

### Factory Pattern

```python
# tests/factories.py
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda o: f'{o.username}@example.com')
    company_id = 10

class InvoiceFactory(DjangoModelFactory):
    class Meta:
        model = Invoice

    invoice_number = factory.Sequence(lambda n: f'INV-{n:04d}')
    company_id = 10
    vendor = factory.SubFactory(VendorFactory)
    status = 'DRAFT'
    amount = Decimal('100.00')
```

### Coverage Targets

| Layer | Target |
|-------|--------|
| Services | 80% |
| Critical paths (payments, auth) | 90% |
| Views | 70% |
| Models | 60% |

---

## Frontend Testing (Vue/TypeScript)

### Test Pyramid

```
         ╱╲
        ╱  ╲         E2E (few)
       ╱────╲        - Playwright/Cypress
      ╱      ╲       - Critical flows
     ╱────────╲      Component (some)
    ╱          ╲     - User interactions
   ╱────────────╲    - Rendered output
  ╱              ╲   Unit (many)
 ╱────────────────╲  - Composables
╱                  ╲ - Utilities, transforms
```

### Stack

| Tool | Purpose |
|------|---------|
| Vitest | Test runner (fast, Vite-native) |
| @vue/test-utils | Component mounting |
| @testing-library/vue | User-centric testing |
| MSW | API mocking |
| Storybook + Chromatic | Visual regression |

### Unit Tests

Test composables and utilities in isolation.

```typescript
// tests/composables/useFormField.test.ts
import { describe, it, expect } from 'vitest'
import { useFormField } from '@/composables/useFormField'

describe('useFormField', () => {
  it('returns disabled from props when provided', () => {
    const { disabled } = useFormField({ disabled: true })
    expect(disabled.value).toBe(true)
  })

  it('falls back to form context when prop not provided', () => {
    // ... with provide/inject mock
  })
})
```

```typescript
// tests/utils/transform.test.ts
import { invoiceFromDto } from '@/models/invoice'

describe('invoiceFromDto', () => {
  it('transforms date strings to Date objects', () => {
    const dto = {
      id: 1,
      invoice_date: '2024-01-15T10:00:00Z',
      // ...
    }

    const invoice = invoiceFromDto(dto)

    expect(invoice.invoiceDate).toBeInstanceOf(Date)
  })
})
```

### Component Tests

User-centric testing with Testing Library.

```typescript
// tests/components/InvoiceForm.test.ts
import { render, screen, fireEvent } from '@testing-library/vue'
import InvoiceForm from '@/components/InvoiceForm.vue'

describe('InvoiceForm', () => {
  it('shows validation error on empty submit', async () => {
    render(InvoiceForm)

    await fireEvent.click(screen.getByRole('button', { name: /submit/i }))

    expect(screen.getByText(/invoice number is required/i)).toBeInTheDocument()
  })

  it('calls onSubmit with form data', async () => {
    const onSubmit = vi.fn()
    render(InvoiceForm, { props: { onSubmit } })

    await fireEvent.update(
      screen.getByLabelText(/invoice number/i),
      'INV-001'
    )
    await fireEvent.click(screen.getByRole('button', { name: /submit/i }))

    expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({ invoiceNumber: 'INV-001' })
    )
  })
})
```

### API Mocking with MSW

```typescript
// tests/mocks/handlers.ts
import { rest } from 'msw'

export const handlers = [
  rest.get('/api/v1/invoices/', (req, res, ctx) => {
    return res(ctx.json({
      results: [
        { id: 1, invoice_number: 'INV-001', amount: '100.00' }
      ],
      count: 1
    }))
  }),

  rest.post('/api/v1/invoices/', async (req, res, ctx) => {
    const body = await req.json()
    return res(ctx.status(201), ctx.json({ id: 1, ...body }))
  }),
]
```

### Visual Regression

Storybook for component documentation + Chromatic for snapshot testing.

```typescript
// src/components/Button.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import Button from './Button.vue'

const meta: Meta<typeof Button> = {
  component: Button,
}

export default meta

export const Primary: StoryObj<typeof Button> = {
  args: { label: 'Click me', variant: 'primary' }
}

export const Disabled: StoryObj<typeof Button> = {
  args: { label: 'Disabled', disabled: true }
}
```

### Coverage Targets

| Layer | Target |
|-------|--------|
| Composables/Utilities | 90% |
| Components | 70% |
| Views | 50% |

---

## CI/CD Integration

### Pipeline Stages

```yaml
test:
  stage: test
  parallel:
    - backend-unit
    - backend-integration
    - frontend-unit
    - frontend-component

  backend-unit:
    script:
      - pytest tests/unit -x --cov=services --cov-fail-under=80

  backend-integration:
    script:
      - pytest tests/integration -x --cov=api
    services:
      - postgres:15

  frontend-unit:
    script:
      - npm run test:unit -- --coverage

  frontend-component:
    script:
      - npm run test:component

visual-regression:
  stage: test
  script:
    - npx chromatic --project-token=$CHROMATIC_TOKEN
  only:
    - merge_requests
```

### Test Database Strategy

| Environment | Database | Speed |
|-------------|----------|-------|
| Local dev | PostgreSQL | Realistic |
| CI unit tests | SQLite in-memory | Fast |
| CI integration | PostgreSQL service | Realistic |

---

## Summary

| Aspect | Backend | Frontend |
|--------|---------|----------|
| Runner | pytest | Vitest |
| Data generation | FactoryBoy | MSW + fixtures |
| Mocking | unittest.mock | vi.fn() / MSW |
| Visual testing | N/A | Storybook + Chromatic |
| Coverage target | 80% services | 90% composables |
