# API Specification

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**OpenAPI Version:** 3.1.0  
**กลุ่มเป้าหมาย:** Frontend Developers, Backend Developers, API Consumers, QA Engineers

---

## 📋 สารบัญ
1. [General Conventions](#1-general-conventions)
2. [Authentication APIs](#2-authentication-apis)
3. [User APIs](#3-user-apis)
4. [Product APIs](#4-product-apis)
5. [Order APIs](#5-order-apis)
6. [Notification APIs](#6-notification-apis)
7. [Error Responses](#7-error-responses)
8. [Rate Limiting](#8-rate-limiting)
9. [OpenAPI Specification](#9-openapi-specification)

---

## 1. General Conventions

### 1.1 Base URL & Versioning
```
Production:  https://api.revenuemanagement.com/v1
Staging:     https://api-staging.revenuemanagement.com/v1
Development: https://api-dev.revenuemanagement.com/v1
```
- API Versioning via URL path (`/v1/`)
- All endpoints return `application/json`
- Timestamps in ISO 8601 format (`YYYY-MM-DDTHH:mm:ss.sssZ`)
- Currency codes follow ISO 4217 (`USD`, `THB`, `EUR`)
- UUIDs for all resource identifiers

### 1.2 Request/Response Envelope
```json
// Success Response
{
  "data": { ... },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-06-22T10:30:00.000Z"
  }
}

// Paginated Response
{
  "data": [ ... ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalItems": 150,
      "totalPages": 8
    },
    "requestId": "req_abc123",
    "timestamp": "2026-06-22T10:30:00.000Z"
  },
  "links": {
    "self": "/v1/orders?page=1&pageSize=20",
    "next": "/v1/orders?page=2&pageSize=20",
    "last": "/v1/orders?page=8&pageSize=20"
  }
}
```

### 1.3 Query Parameters Standard
| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `page` | integer | Page number (default: 1) | `page=2` |
| `pageSize` | integer | Items per page (default: 20, max: 100) | `pageSize=50` |
| `sort` | string | Field to sort by (`-` for desc) | `sort=-createdAt` |
| `filter` | object | JSON-encoded filter conditions | `filter={"status":"active"}` |
| `search` | string | Full-text search query | `search=invoice` |
| `fields` | string | Comma-separated fields to return | `fields=id,name,status` |

### 1.4 Headers Standard
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes (protected) | `Bearer <jwt_token>` |
| `X-Tenant-Id` | Yes | Tenant identifier for multi-tenancy |
| `X-Correlation-Id` | No | Request tracing ID |
| `Content-Type` | Yes (POST/PUT/PATCH) | `application/json` |
| `Accept` | No | `application/json` |
| `Idempotency-Key` | No (POST) | Prevent duplicate operations |

---

## 2. Authentication APIs

### 2.1 Login
**`POST /v1/auth/login`**

**Request:**
```json
{
  "email": "user@company.com",
  "password": "SecureP@ssw0rd",
  "mfaCode": "123456" // Optional if MFA enabled
}
```

**Response (200 OK):**
```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600,
    "tokenType": "Bearer",
    "user": {
      "id": "usr_789xyz",
      "email": "user@company.com",
      "role": "sales_rep",
      "tenantId": "ten_abc123",
      "permissions": ["quotes.create", "orders.read"]
    }
  }
}
```

### 2.2 Refresh Token
**`POST /v1/auth/refresh`**

**Request:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response (200 OK):**
```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600,
    "tokenType": "Bearer"
  }
}
```

### 2.3 Logout
**`POST /v1/auth/logout`**
- Invalidates refresh token server-side
- Requires `Authorization: Bearer <token>`
- Returns `204 No Content` on success

### 2.4 Password Reset Flow
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/auth/forgot-password` | Request reset email |
| `POST` | `/v1/auth/verify-reset-code` | Verify OTP/code |
| `POST` | `/v1/auth/reset-password` | Set new password |

---

## 3. User APIs

### 3.1 List Users
**`GET /v1/users`**

**Query Params:**
```
?page=1&pageSize=20&sort=-createdAt&filter={"role":"sales_rep","isActive":true}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "usr_789xyz",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@company.com",
      "role": "sales_rep",
      "tenantId": "ten_abc123",
      "isActive": true,
      "createdAt": "2026-01-15T08:00:00.000Z",
      "updatedAt": "2026-06-20T14:30:00.000Z"
    }
  ],
  "meta": {
    "pagination": { "page": 1, "pageSize": 20, "totalItems": 45, "totalPages": 3 }
  }
}
```

### 3.2 Get User by ID
**`GET /v1/users/{id}`**
- Returns `404` if not found or outside tenant scope
- Includes profile, preferences, and audit summary

### 3.3 Create User
**`POST /v1/users`**

**Request:**
```json
{
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane@company.com",
  "role": "billing_specialist",
  "sendInviteEmail": true
}
```

**Response (201 Created):**
```json
{
  "data": {
    "id": "usr_new123",
    "email": "jane@company.com",
    "role": "billing_specialist",
    "status": "pending_activation",
    "createdAt": "2026-06-22T10:30:00.000Z"
  }
}
```

### 3.4 Update User
**`PATCH /v1/users/{id}`**
- Partial updates supported
- Role changes require `admin` or `sales_manager` permission
- Returns `200 OK` with updated resource

### 3.5 Deactivate User
**`POST /v1/users/{id}/deactivate`**
- Soft delete (sets `isActive: false`)
- Revokes active sessions
- Returns `204 No Content`

---

## 4. Product APIs

### 4.1 List Products
**`GET /v1/products`**

**Query Params:**
```
?page=1&pageSize=20&sort=name&filter={"category":"software","isActive":true}&search=enterprise
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "prod_abc123",
      "sku": "ENT-CRM-001",
      "name": "Enterprise CRM License",
      "category": "software",
      "type": "subscription",
      "isActive": true,
      "defaultPrice": { "amount": 1500.00, "currency": "USD" },
      "attributes": { "maxUsers": 100, "storageGB": 500 },
      "createdAt": "2026-01-10T00:00:00.000Z"
    }
  ],
  "meta": { "pagination": { "page": 1, "pageSize": 20, "totalItems": 84, "totalPages": 5 } }
}
```

### 4.2 Get Product Details
**`GET /v1/products/{id}`**
- Includes pricing tiers, related products, inventory status
- Returns `404` if not found

### 4.3 Create Product
**`POST /v1/products`**

**Request:**
```json
{
  "sku": "ENT-CRM-002",
  "name": "Enterprise CRM Premium",
  "category": "software",
  "type": "subscription",
  "defaultPrice": { "amount": 2500.00, "currency": "USD" },
  "attributes": { "maxUsers": 500, "storageGB": 2000, "supportTier": "premium" },
  "pricingTiers": [
    { "minQuantity": 1, "maxQuantity": 10, "price": 2500.00 },
    { "minQuantity": 11, "maxQuantity": null, "price": 2200.00 }
  ]
}
```

**Response (201 Created):**
```json
{
  "data": { "id": "prod_new456", "sku": "ENT-CRM-002", "status": "active", "createdAt": "..." }
}
```

### 4.4 Update Product
**`PUT /v1/products/{id}`** (Full) / `PATCH /v1/products/{id}` (Partial)
- SKU changes require deactivation of old SKU and creation of new
- Price changes create new price book entry with effective date

### 4.5 Price Book Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/price-books` | List price books |
| `POST` | `/v1/price-books` | Create price book |
| `GET` | `/v1/price-books/{id}/entries` | List pricing entries |
| `POST` | `/v1/price-books/{id}/entries` | Add pricing entry |
| `PATCH` | `/v1/price-books/{id}/entries/{entryId}` | Update pricing |

---

## 5. Order APIs

### 5.1 Create Order from Quote
**`POST /v1/orders`**

**Request:**
```json
{
  "quoteId": "quo_789xyz",
  "accountId": "acc_abc123",
  "notes": "Rush delivery requested",
  "billingAddress": { ... },
  "shippingAddress": { ... }
}
```

**Response (201 Created):**
```json
{
  "data": {
    "id": "ord_new123",
    "orderNumber": "ORD-2026-00456",
    "status": "pending",
    "totalAmount": { "amount": 15000.00, "currency": "USD" },
    "items": [ ... ],
    "createdAt": "2026-06-22T10:30:00.000Z"
  }
}
```

### 5.2 List Orders
**`GET /v1/orders`**

**Query Params:**
```
?page=1&pageSize=20&sort=-orderDate&filter={"status":["pending","activated"],"accountId":"acc_abc123"}
```

### 5.3 Get Order Details
**`GET /v1/orders/{id}`**
- Returns full order with items, billing schedule, linked invoice, assets
- Includes audit trail summary

### 5.4 Activate Order
**`POST /v1/orders/{id}/activate`**
- Transitions status from `pending` → `activated`
- Triggers asset generation, billing schedule creation, contract activation
- Idempotent (safe to retry)
- Returns `200 OK` with activated order

### 5.5 Cancel Order
**`POST /v1/orders/{id}/cancel`**

**Request:**
```json
{
  "reason": "Customer requested cancellation",
  "refundAmount": { "amount": 5000.00, "currency": "USD" }
}
```

### 5.6 Order Status Lifecycle
```
pending → activated → fulfilled → closed
   ↓          ↓
cancelled   expired
```
- Status transitions validated server-side
- Invalid transitions return `409 Conflict`

---

## 6. Notification APIs

### 6.1 List Notifications
**`GET /v1/notifications`**

**Query Params:**
```
?page=1&pageSize=20&filter={"isRead":false,"type":"invoice_overdue"}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "not_abc123",
      "type": "invoice_overdue",
      "title": "Invoice INV-2026-001 is overdue",
      "message": "Payment due 5 days ago. Amount: $1,500.00",
      "isRead": false,
      "priority": "high",
      "relatedResource": { "type": "invoice", "id": "inv_789xyz" },
      "createdAt": "2026-06-20T08:00:00.000Z"
    }
  ],
  "meta": { "pagination": { ... } }
}
```

### 6.2 Mark as Read
**`PATCH /v1/notifications/{id}/read`**
- Returns `204 No Content`

### 6.3 Mark All as Read
**`POST /v1/notifications/mark-all-read`**
- Returns `204 No Content`

### 6.4 Notification Preferences
**`GET /v1/notifications/preferences`**
**`PUT /v1/notifications/preferences`**

**Request/Response:**
```json
{
  "email": { "invoice_created": true, "order_activated": true, "marketing": false },
  "push": { "invoice_overdue": true, "payment_received": true },
  "sms": { "critical_alerts": true }
}
```

### 6.5 Real-time Updates (Optional SSE/WebSocket)
```
GET /v1/notifications/stream
Headers: Authorization: Bearer <token>
Content-Type: text/event-stream

data: {"id":"not_abc","type":"payment_received","timestamp":"..."}
```

---

## 7. Error Responses

### 7.1 Standard Error Format (RFC 7807 Inspired)
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "password",
        "message": "Must be at least 8 characters",
        "code": "MIN_LENGTH"
      }
    ],
    "requestId": "req_abc123",
    "timestamp": "2026-06-22T10:30:00.000Z",
    "documentationUrl": "https://docs.revenuemanagement.com/api/errors#VALIDATION_ERROR"
  }
}
```

### 7.2 HTTP Status Code Mapping
| Status | Code | Description |
|--------|------|-------------|
| `400` | `BAD_REQUEST` | Invalid request payload or parameters |
| `401` | `UNAUTHORIZED` | Missing or invalid authentication |
| `403` | `FORBIDDEN` | Insufficient permissions or tenant isolation |
| `404` | `NOT_FOUND` | Resource not found or outside scope |
| `409` | `CONFLICT` | State conflict (e.g., invalid status transition) |
| `422` | `UNPROCESSABLE_ENTITY` | Semantic validation failed |
| `429` | `RATE_LIMITED` | Too many requests |
| `500` | `INTERNAL_ERROR` | Server error |
| `503` | `SERVICE_UNAVAILABLE` | Maintenance or dependency failure |

### 7.3 Error Handling Guidelines
- Never expose stack traces or internal DB details
- Log `requestId` server-side for troubleshooting
- Return consistent structure across all endpoints
- Include `documentationUrl` for self-service debugging
- Use `Idempotency-Key` header to safely retry `POST`/`PATCH`

---

## 8. Rate Limiting

### 8.1 Strategy
- **Algorithm:** Sliding Window Log + Token Bucket fallback
- **Scope:** Per User + Per IP + Per Tenant
- **Storage:** Redis (TTL-based counters)

### 8.2 Default Limits
| Tier | Requests/Minute | Requests/Hour | Burst |
|------|----------------|---------------|-------|
| Free / Trial | 60 | 1,000 | 10 |
| Standard | 300 | 10,000 | 50 |
| Enterprise | 1,200 | 50,000 | 200 |
| Internal / System | Unlimited | Unlimited | N/A |

### 8.3 Rate Limit Headers
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 285
X-RateLimit-Reset: 1719043200
Retry-After: 45
```

### 8.4 Rate Limit Response (429)
```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please retry after 45 seconds.",
    "retryAfter": 45,
    "limit": 300,
    "remaining": 0,
    "resetAt": "2026-06-22T10:31:00.000Z",
    "requestId": "req_abc123"
  }
}
```

### 8.5 Best Practices for Clients
- Respect `Retry-After` header
- Implement exponential backoff
- Cache responses when `Cache-Control` allows
- Use bulk endpoints instead of multiple single requests
- Monitor `X-RateLimit-Remaining` to throttle preemptively

---

## 9. OpenAPI Specification

### 9.1 Specification Overview
The complete API specification is maintained in OpenAPI 3.1.0 format, compliant with JSON Schema 2020-12. It serves as:
- Single source of truth for Frontend/Backend contracts
- Auto-generated SDKs (TypeScript, Go, Python)
- Mock server for parallel development
- Contract testing baseline

### 9.2 OpenAPI 3.1.0 Structure (YAML Snippet)
```yaml
openapi: 3.1.0
info:
  title: Revenue Management API
  version: 1.0.0
  description: RESTful API for Order-to-Cash, CPQ, and Billing operations
  contact:
    name: API Support
    email: api-support@revenuemanagement.com
  license:
    name: Proprietary
    identifier: LicenseRef-Proprietary

servers:
  - url: https://api.revenuemanagement.com/v1
    description: Production
  - url: https://api-staging.revenuemanagement.com/v1
    description: Staging

tags:
  - name: authentication
    description: Auth & session management
  - name: users
    description: User & tenant management
  - name: products
    description: Product catalog & pricing
  - name: orders
    description: Order lifecycle & fulfillment
  - name: notifications
    description: Alerts & preferences

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-Tenant-Id

  schemas:
    Error:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [code, message]
          properties:
            code:
              type: string
              examples: ["VALIDATION_ERROR", "UNAUTHORIZED", "RATE_LIMITED"]
            message:
              type: string
            details:
              type: array
              items:
                type: object
                properties:
                  field: { type: string }
                  message: { type: string }
            requestId: { type: string, format: uuid }
            timestamp: { type: string, format: date-time }

    User:
      type: object
      required: [id, email, role, tenantId]
      properties:
        id: { type: string, format: uuid }
        firstName: { type: ["string", "null"] }
        lastName: { type: ["string", "null"] }
        email: { type: string, format: email }
        role:
          type: string
          enum: [admin, sales_rep, sales_manager, billing_specialist, contract_manager, viewer]
        tenantId: { type: string, format: uuid }
        isActive: { type: boolean, default: true }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }

    Order:
      type: object
      required: [id, orderNumber, status, totalAmount]
      properties:
        id: { type: string, format: uuid }
        orderNumber: { type: string, pattern: "^ORD-\\d{4}-\\d{6}$" }
        status:
          type: string
          enum: [pending, activated, fulfilled, cancelled, expired]
        totalAmount:
          type: object
          required: [amount, currency]
          properties:
            amount: { type: number, format: double, minimum: 0 }
            currency: { type: string, pattern: "^[A-Z]{3}$" }
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        createdAt: { type: string, format: date-time }

  parameters:
    Pagination:
      name: page
      in: query
      schema: { type: integer, minimum: 1, default: 1 }
    PageSize:
      name: pageSize
      in: query
      schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
    Sort:
      name: sort
      in: query
      schema: { type: string, description: "Field name, prefix with - for descending" }

paths:
  /auth/login:
    post:
      tags: [authentication]
      summary: Authenticate user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email: { type: string, format: email }
                password: { type: string, format: password, minLength: 8 }
                mfaCode: { type: string, pattern: "^\\d{6}$" }
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: object
                    properties:
                      accessToken: { type: string }
                      refreshToken: { type: string }
                      expiresIn: { type: integer }
                      user: { $ref: '#/components/schemas/User' }
        '401':
          description: Invalid credentials
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /orders:
    get:
      tags: [orders]
      summary: List orders
      security:
        - bearerAuth: []
        - apiKey: []
      parameters:
        - $ref: '#/components/parameters/Pagination'
        - $ref: '#/components/parameters/PageSize'
        - $ref: '#/components/parameters/Sort'
        - name: filter
          in: query
          schema: { type: string, description: "JSON-encoded filter object" }
      responses:
        '200':
          description: List of orders
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items: { $ref: '#/components/schemas/Order' }
                  meta:
                    type: object
                    properties:
                      pagination:
                        type: object
                        properties:
                          page: { type: integer }
                          pageSize: { type: integer }
                          totalItems: { type: integer }
                          totalPages: { type: integer }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '429': { $ref: '#/components/responses/RateLimited' }

    post:
      tags: [orders]
      summary: Create order from quote
      security:
        - bearerAuth: []
        - apiKey: []
      parameters:
        - name: Idempotency-Key
          in: header
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [quoteId, accountId]
              properties:
                quoteId: { type: string, format: uuid }
                accountId: { type: string, format: uuid }
                notes: { type: string, maxLength: 1000 }
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { $ref: '#/components/schemas/Order' }
        '400': { $ref: '#/components/responses/BadRequest' }
        '409':
          description: Quote already converted or invalid state
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /orders/{orderId}/activate:
    post:
      tags: [orders]
      summary: Activate order
      security:
        - bearerAuth: []
        - apiKey: []
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          description: Order activated
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { $ref: '#/components/schemas/Order' }
        '404': { $ref: '#/components/responses/NotFound' }
        '409': { $ref: '#/components/responses/Conflict' }

  /notifications:
    get:
      tags: [notifications]
      summary: List notifications
      security: [{ bearerAuth: [] }, { apiKey: [] }]
      responses:
        '200':
          description: Paginated notifications
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      type: object
                      properties:
                        id: { type: string, format: uuid }
                        type: { type: string }
                        title: { type: string }
                        message: { type: string }
                        isRead: { type: boolean }
                        priority: { type: string, enum: [low, medium, high, critical] }
                        createdAt: { type: string, format: date-time }
                  meta: { type: object }

  /notifications/{notificationId}/read:
    patch:
      tags: [notifications]
      summary: Mark notification as read
      security: [{ bearerAuth: [] }, { apiKey: [] }]
      parameters:
        - name: notificationId
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '204': { description: Updated successfully }

webhooks:
  orderStatusChanged:
    post:
      summary: Webhook triggered when order status changes
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                orderId: { type: string, format: uuid }
                previousStatus: { type: string }
                newStatus: { type: string }
                timestamp: { type: string, format: date-time }
                tenantId: { type: string, format: uuid }
      responses:
        '200': { description: Webhook received }
```

### 9.3 Contract Validation & Testing
- **Backend:** `@nestjs/swagger` generates OpenAPI at runtime, validated against schema
- **Frontend:** `openapi-typescript` generates typed SDK (`api.d.ts`)
- **CI/CD:** `spectral` lints OpenAPI spec, `dredd` runs contract tests
- **Mock Server:** `prism` serves mock responses for parallel dev
- **Versioning:** Spec versioned with API (`/v1/openapi.yaml`), changelog maintained

### 9.4 SDK Generation Commands
```bash
# TypeScript (Frontend)
npx openapi-typescript openapi.yaml -o src/api/api.d.ts

# Go (Backend Microservice)
openapi-generator-cli generate -i openapi.yaml -g go -o ./sdk/go

# Python (Analytics/ML Service)
openapi-generator-cli generate -i openapi.yaml -g python -o ./sdk/python
```

---

## ภาคผนวก

### ภาคผนวก A: API Checklist
```yaml
Pre-Release Checklist:
  - [ ] All endpoints documented in OpenAPI 3.1.0
  - [ ] Request/response schemas validated
  - [ ] Authentication & authorization tested
  - [ ] Rate limiting configured & tested
  - [ ] Error responses standardized
  - [ ] Pagination, sorting, filtering implemented
  - [ ] Idempotency keys supported for POST
  - [ ] Tenant isolation enforced
  - [ ] CORS headers configured
  - [ ] API versioning strategy applied
  - [ ] SDKs generated & tested
  - [ ] Contract tests passing
  - [ ] Performance tested (p95 < 500ms)
  - [ ] Security scan completed
```

### ภาคผนวก B: Migration Guide (v0 → v1)
- All `camelCase` fields remain unchanged
- `pagination` moved to `meta.pagination`
- `errors` array wrapped in `error` object
- `X-Request-Id` renamed to `requestId` in responses
- Deprecated endpoints return `410 Gone` with migration link
- Grace period: 90 days before v0 shutdown

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นสัญญาญาระหว่าง Frontend และ Backend ในการพัฒนาระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** API Architecture Team
