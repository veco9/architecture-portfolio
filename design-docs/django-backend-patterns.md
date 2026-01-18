# Django Backend Patterns

A collection of backend patterns used across enterprise Django applications.

## Table of Contents

1. [JWT Authentication with Pluggable Backends](#1-jwt-authentication-with-pluggable-backends)
2. [Analytics Caching Strategy](#2-analytics-caching-strategy)
3. [Financial Calculations & Decimal Precision](#3-financial-calculations--decimal-precision)
4. [API Versioning](#4-api-versioning)
5. [Audit Trail for Compliance](#5-audit-trail-for-compliance)
6. [External Data Synchronization](#6-external-data-synchronization)
7. [Multi-Tenant Architecture](#7-multi-tenant-architecture)
8. [Payment Gateway Integration](#8-payment-gateway-integration)
9. [Real-Time Broadcasting with Django Channels](#9-real-time-broadcasting-with-django-channels)
10. [Clean Architecture (Services-Selectors-Views)](#10-clean-architecture-services-selectors-views)
11. [Pessimistic Locking for Concurrent Processing](#11-pessimistic-locking-for-concurrent-processing)

---

## 1. JWT Authentication with Pluggable Backends

### Problem
Enterprise clients use diverse identity providers: Active Directory, Azure AD, Oracle OID, OpenLDAP, or database credentials. Need unified authentication with token blacklisting for secure logout.

### Solution
Custom JWT service with backend selection based on tenant configuration.

**Token Lifecycle:**
- Short-lived access tokens (15-60 min)
- Long-lived refresh tokens (7-30 days) tracked in `OutstandingToken` table
- `BlacklistedToken` table enables immediate logout
- `MAX_LOGIN_LIFETIME` forces periodic re-authentication

**Backend Interface:**
```python
class AuthBackend:
    def authenticate(self, username: str, password: str) -> User | None
    def get_password_info(self, user: User) -> PasswordInfo
    def change_password(self, user: User, old_pw: str, new_pw: str) -> bool
```

**Blacklist Flow:**
1. User logs out → refresh token JTI added to blacklist
2. Token refresh attempt → check blacklist → reject if found
3. Daily cleanup removes expired blacklist entries

---

## 2. Analytics Caching Strategy

### Problem
BI dashboards need sub-second response from data warehouses with millions of records. Queries involve complex aggregations. Data refreshes every 15-60 minutes.

### Solution
Tiered caching with Redis using cache-aside pattern.

**Cache Tiers:**

| Tier | Content | TTL | Size |
|------|---------|-----|------|
| 1 | Filter dimensions (time, geo, categories) | None (invalidate on refresh) | ~10-50 MB |
| 2 | Pre-computed aggregates (KPIs, rankings) | 24h | ~100 MB |
| 3 | User query results (filtered reports) | 1h | Variable |
| 4 | Request memoization (dedup in-flight) | 60s | Small |

**Cache Key Design:**
```
{tenant}:{entity}:{hash(filters)}:{user_permission_hash}
```

**Graceful Degradation:** System works without cache, just slower. Permission-aware bypass for restricted data.

---

## 3. Financial Calculations & Decimal Precision

### Problem
Floating-point arithmetic causes precision errors (`0.1 + 0.2 = 0.30000000000000004`) unacceptable for financial applications.

### Solution
Python `Decimal` with explicit rounding modes.

**Rules:**
```python
# Initialize from strings (not floats)
amount = Decimal('10.50')  # Correct
amount = Decimal(10.50)    # Wrong - precision lost

# Explicit rounding
fee = Decimal('3.333333')
rounded = fee.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

# VAT: round each line item
line_tax = (line_amount * tax_rate).quantize(Decimal('0.01'))
```

**Django Integration:**
```python
amount = models.DecimalField(max_digits=12, decimal_places=2)
```

---

## 4. API Versioning

### Problem
APIs evolve with breaking changes. Need to support multiple client versions without forcing immediate upgrades.

### Solution
URL path versioning with deprecation headers.

**Structure:**
```
/api/v1/invoices/
/api/v2/invoices/
```

**Deprecation Strategy:**
- Maintain N-1 versions (current + previous)
- `Deprecation` and `Sunset` headers warn clients
- Version-specific serializers handle format differences

---

## 5. Audit Trail for Compliance

### Problem
Enterprise applications need automatic change tracking for compliance, debugging, and accountability. Manual logging leads to inconsistent coverage.

### Solution
Auditable abstract base class with dual-database storage.

**How it Works:**
1. Model inherits from `Auditable` base class
2. On load: capture initial state
3. On save: compute diff (old values, new values, changed fields)
4. Get user from request context (thread-local via middleware)
5. Write to separate audit database

**Audit Log Schema:**
```
- created_at, completed_at
- modified_by (user)
- entity_type, operation (CREATE/UPDATE/DELETE)
- record_id
- old_values (JSON)
- new_values (JSON)
- diff (JSON)
```

**Custom Serialization:** Handles Django types (ImageField paths, UUIDs, datetimes) for JSON storage.

---

## 6. External Data Synchronization

### Problem
Blind sync from external systems (ERP, LDAP) risks data loss, propagating errors, and breaking local customizations.

### Solution
Preview-and-Apply pattern with admin approval.

**Workflow:**
1. **Fetch & Compare:** Compute diff (NEW, MODIFIED, DELETED records)
2. **Cache Preview:** Store in `SyncPreview` table with expiration
3. **Admin Review:** Dashboard shows field-level changes
4. **Safe Apply:** `@transaction.atomic` applies approved changes

**Conflict Resolution:**
- SKIP: Keep local value
- OVERWRITE: Use external value
- MERGE: Field-level strategy

**SyncPreview Model:**
```python
class SyncPreview(models.Model):
    entity_type = models.CharField(max_length=100)
    status = models.CharField()  # PENDING, APPROVED, REJECTED, APPLIED
    new_count = models.IntegerField()
    modified_count = models.IntegerField()
    deleted_count = models.IntegerField()
    diff_data = models.JSONField()
    expires_at = models.DateTimeField()
    reviewed_by = models.ForeignKey(User)
```

---

## 7. Multi-Tenant Architecture

### Problem
SaaS platforms must isolate tenant data while balancing operational complexity, cost efficiency, and performance.

### Solution
Row-level tenant isolation with middleware-enforced context.

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Authentication                                        │
│  - JWT contains tenant claim                                    │
│  - Token validation includes tenant membership check            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Middleware                                            │
│  - Extract tenant from request                                  │
│  - Validate user-tenant membership                              │
│  - Set thread-local context                                     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Query Layer                                           │
│  - Base queryset always filters by tenant                       │
│  - Query builder injects tenant_id in WHERE clause              │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Audit                                                 │
│  - All queries logged with tenant context                       │
└─────────────────────────────────────────────────────────────────┘
```

**User-Tenant Junction:**
```python
# Users can access multiple tenants with different roles
class UserTenantRole(models.Model):
    user = models.ForeignKey(User)
    tenant = models.ForeignKey(Tenant)
    role = models.CharField()  # admin, editor, viewer
    granted_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ['user', 'tenant']
```

**Automatic Query Scoping:**
```python
class TenantScopedManager(models.Manager):
    def get_queryset(self):
        tenant = get_current_tenant()  # From middleware thread-local
        return super().get_queryset().filter(tenant_id=tenant)
```

---

## 8. Payment Gateway Integration

### Problem
Payment platforms need robust multi-gateway processing, fee calculation, webhook handling, and reconciliation.

### Solution
Gateway abstraction layer with adapter pattern.

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Payment Service                              │
│  (Business logic, fee calculation, validation)                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                  Gateway Interface                              │
│  initiate_payment(), capture(), refund(), get_status()          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │  Cards      │   │  SEPA       │   │  PayPal     │
  │  Adapter    │   │  Adapter    │   │  Adapter    │
  └─────────────┘   └─────────────┘   └─────────────┘
```

**Gateway Interface:**
```python
class PaymentGateway(ABC):
    @abstractmethod
    def initiate_payment(self, request: PaymentRequest) -> PaymentResult:
        pass

    @abstractmethod
    def capture(self, transaction_id: str, amount: Decimal = None) -> PaymentResult:
        pass

    @abstractmethod
    def refund(self, transaction_id: str, amount: Decimal = None) -> PaymentResult:
        pass

    @abstractmethod
    def verify_webhook(self, payload: bytes, signature: str) -> bool:
        pass
```

**Dual-State Transaction Model:**
```python
class Transaction(models.Model):
    # Internal state
    internal_status = models.CharField()  # CREATED, PROCESSING, COMPLETED, FAILED

    # Gateway state (may differ during sync)
    gateway_status = models.CharField()   # initialized, authorized, settled
    gateway_reference = models.CharField()

    # Reconciliation
    last_synced_at = models.DateTimeField()
    reconciled = models.BooleanField(default=False)
```

**Webhook Processing:**
```python
@csrf_exempt
def webhook_handler(request):
    # 1. Verify signature (HMAC-SHA256)
    if not gateway.verify_webhook(request.body, request.headers['X-Signature']):
        return HttpResponseForbidden()

    # 2. Parse and validate
    event = json.loads(request.body)

    # 3. Idempotency check
    if WebhookEvent.objects.filter(event_id=event['id']).exists():
        return HttpResponse(status=200)  # Already processed

    # 4. Process atomically
    with transaction.atomic():
        WebhookEvent.objects.create(event_id=event['id'], payload=event)
        update_transaction_from_webhook(event)

    return HttpResponse(status=200)
```

---

## 9. Real-Time Broadcasting with Django Channels

### Problem
Real-time applications (dashboards, IoT, collaboration) need push updates without polling.

### Solution
Django Channels with Redis channel layer for WebSocket broadcasting.

**Consumer Implementation:**
```python
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class DashboardConsumer(AsyncJsonWebsocketConsumer):
    group_name = 'dashboard_updates'

    async def connect(self):
        # Authenticate
        if not self.scope['user'].is_authenticated:
            await self.close(code=4001)
            return

        # Join broadcast group
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()
        await self.send_initial_state()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    # Handler for group_send messages (type: 'send_update')
    async def send_update(self, event):
        await self.send_json({'type': 'update', 'data': event['data']})
```

**Broadcasting from Sync Code:**
```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def broadcast_update(data: dict, group: str = 'dashboard_updates'):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        group,
        {'type': 'send_update', 'data': data}
    )

# Usage in Django view
class SensorUpdateView(APIView):
    def post(self, request):
        reading = SensorReading.objects.create(**request.data)
        broadcast_update({'sensor_id': reading.sensor_id, 'value': reading.value})
        return Response({'status': 'ok'})
```

**Channel Layer Config:**
```python
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {'hosts': [(REDIS_HOST, REDIS_PORT)]},
    }
}
```

---

## 10. Clean Architecture (Services-Selectors-Views)

### Problem
Django projects evolve into "fat views" or "fat models" anti-patterns where business logic scatters without clear boundaries.

### Solution
Strict layer separation with Services-Selectors-Views pattern.

**Layer Responsibilities:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Views (API Layer)                                              │
│  • HTTP handling, serialization, permissions                    │
│  • Orchestrates selectors and services                          │
│  • NO business logic                                            │
├─────────────────────────────────────────────────────────────────┤
│  Selectors (Read)          │  Services (Write)                  │
│  • Return QuerySets        │  • Create/Update/Delete            │
│  • Filtering, aggregations │  • Business logic                  │
│  • NO side effects         │  • @transaction.atomic             │
├─────────────────────────────────────────────────────────────────┤
│  Models (Data Layer)                                            │
│  • Field definitions, relationships                             │
│  • NO business logic                                            │
└─────────────────────────────────────────────────────────────────┘
```

**Selector Example:**
```python
def invoice_list(*, company_id: int, status: str = None) -> QuerySet[Invoice]:
    qs = Invoice.objects.filter(company_id=company_id)
    if status:
        qs = qs.filter(status=status)
    return qs.select_related('vendor', 'created_by')
```

**Service Example:**
```python
@transaction.atomic
def invoice_create(*, company_id: int, vendor_id: int, items: list, created_by_id: int) -> Invoice:
    vendor = Vendor.objects.get(pk=vendor_id, company_id=company_id)
    invoice = Invoice.objects.create(company_id=company_id, vendor=vendor, created_by_id=created_by_id)

    total = Decimal('0')
    for item in items:
        InvoiceItem.objects.create(invoice=invoice, **item)
        total += item['quantity'] * item['unit_price']

    invoice.amount = total
    invoice.save(update_fields=['amount'])
    return invoice
```

**View Example:**
```python
class InvoiceListView(APIView):
    def get(self, request):
        invoices = invoice_list(company_id=request.user.company_id)
        return Response(InvoiceSerializer(invoices, many=True).data)

    def post(self, request):
        serializer = InvoiceCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        invoice = invoice_create(company_id=request.user.company_id, **serializer.validated_data)
        return Response(InvoiceSerializer(invoice).data, status=201)
```

---

## 11. Pessimistic Locking for Concurrent Processing

### Problem
Multi-worker systems (email queues, job processors) face race conditions where workers process the same record simultaneously.

### Solution
`select_for_update(skip_locked=True)` for safe concurrent access.

**The Problem:**
```python
# DANGEROUS: Race condition
def process_next():
    email = Email.objects.filter(status='PENDING').first()  # Both workers get same email!
    send_email(email)  # Both send it!
    email.status = 'SENT'
    email.save()
```

**The Solution:**
```python
from django.db import transaction

def process_next():
    with transaction.atomic():
        # skip_locked: if row is locked, skip it and get next available
        email = Email.objects.select_for_update(skip_locked=True).filter(
            status='PENDING'
        ).first()

        if not email:
            return None

        email.status = 'PROCESSING'
        email.save()

        try:
            send_email(email)
            email.status = 'SENT'
        except Exception as e:
            email.status = 'FAILED'
            email.error = str(e)

        email.save()
        return email
```

**Batch Processing:**
```python
def process_batch(batch_size: int = 10):
    with transaction.atomic():
        items = list(
            Task.objects.select_for_update(skip_locked=True)
            .filter(status='PENDING')
            .order_by('priority')[:batch_size]
        )

        for item in items:
            process_item(item)

        return len(items)
```

**Database Support:**
| Database | select_for_update | skip_locked |
|----------|-------------------|-------------|
| PostgreSQL | Yes | Yes (9.5+) |
| MySQL | Yes | Yes (8.0+) |
| Oracle | Yes | Yes |
| SQLite | No | No |

---

## Summary

| Pattern | Use Case |
|---------|----------|
| JWT + Pluggable Backends | Multi-tenant auth with diverse IdPs |
| Tiered Caching | Analytics dashboards with millions of records |
| Decimal Precision | Any financial calculation |
| URL Versioning | Public APIs with multiple client versions |
| Audit Trail | Compliance, debugging, accountability |
| Preview-and-Apply Sync | Safe external data integration |
| Multi-Tenant Architecture | SaaS with row-level isolation |
| Payment Gateway | Multi-gateway payment processing |
| Real-Time Broadcasting | WebSocket push updates |
| Clean Architecture | Services-Selectors-Views separation |
| Pessimistic Locking | Concurrent queue processing |
