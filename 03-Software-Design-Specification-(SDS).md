# Software Design Specification (SDS)
## ระบบจัดการ Revenue Management

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**Tech Stack:** React · NestJS · Go · Python · Redis · Kafka · PostgreSQL  

---

## 📋 สารบัญ
1. [Coding Standards](#1-coding-standards)
2. [Module Design](#2-module-design)
3. [Domain Design](#3-domain-design)
4. [Folder Structure](#4-folder-structure)
5. [Design Patterns](#5-design-patterns)
6. [Dependency Injection](#6-dependency-injection)
7. [Error Handling](#7-error-handling)
8. [Logging Strategy](#8-logging-strategy)
9. [Testing Strategy](#9-testing-strategy)
10. [Security Design](#10-security-design)
11. [Caching Design](#11-caching-design)
12. [Event Design](#12-event-design)

---

## 1. Coding Standards

### 1.1 General Principles

```
┌─────────────────────────────────────────────────────────────┐
│              CODING PRINCIPLES                                │
─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SOLID Principles                                         │
│     ├─ Single Responsibility                                 │
│     ├─ Open/Closed                                           │
│     ├─ Liskov Substitution                                   │
│     ├─ Interface Segregation                                 │
│     └─ Dependency Inversion                                  │
│                                                              │
│  2. DRY (Don't Repeat Yourself)                              │
│  3. KISS (Keep It Simple, Stupid)                            │
│  4. YAGNI (You Aren't Gonna Need It)                         │
│  5. Clean Code                                               │
│     ├─ Meaningful Naming                                     │
│     ├─ Small Functions (< 20 lines)                          │
│     ├─ Single Level of Abstraction                           │
│     └─ Self-Documenting Code                                 │
│                                                              │
│  6. 12-Factor App Methodology                                │
│  7. API-First Design                                         │
│  8. Test-Driven Development (TDD)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Language-Specific Standards

#### 1.2.1 React (Frontend)

```typescript
// File Naming: PascalCase for components, camelCase for utilities
// Component: UserAvatar.tsx, QuoteBuilder.tsx
// Utility: formatDate.ts, calculateDiscount.ts
// Hook: useAuth.ts, useQuote.ts
// Type: types.ts, interfaces.ts

// Component Structure
interface UserAvatarProps {
  userId: string;
  size?: 'sm' | 'md' | 'lg';
  showStatus?: boolean;
}

const UserAvatar: React.FC<UserAvatarProps> = ({
  userId,
  size = 'md',
  showStatus = false,
}) => {
  // Hooks at the top
  const { data: user, isLoading } = useUser(userId);
  
  // Early returns
  if (isLoading) return <Skeleton />;
  if (!user) return null;
  
  // Main render
  return (
    <div className={`avatar avatar-${size}`}>
      <img src={user.avatarUrl} alt={user.name} />
      {showStatus && <StatusIndicator status={user.status} />}
    </div>
  );
};

// Rules:
// - No any type - use proper TypeScript types
// - Functional components only (no class components)
// - Custom hooks for reusable logic
// - React Query / SWR for data fetching
// - Zustand / Redux Toolkit for global state
// - CSS Modules or Tailwind for styling
// - ESLint + Prettier with strict rules
```

#### 1.2.2 NestJS (Backend API)

```typescript
// File Naming: kebab-case for files, PascalCase for classes
// Controller: quotes.controller.ts
// Service: quotes.service.ts
// DTO: create-quote.dto.ts
// Entity: quote.entity.ts
// Module: quotes.module.ts

// Controller Example
@Controller('quotes')
@ApiTags('Quotes')
export class QuotesController {
  constructor(private readonly quotesService: QuotesService) {}

  @Post()
  @ApiOperation({ summary: 'Create new quote' })
  @ApiResponse({ status: 201, type: QuoteResponseDto })
  async create(
    @Body() createQuoteDto: CreateQuoteDto,
    @CurrentUser() user: User,
  ): Promise<QuoteResponseDto> {
    return this.quotesService.create(createQuoteDto, user);
  }

  @Get(':id')
  async findOne(
    @Param('id', ParseUUIDPipe) id: string,
  ): Promise<QuoteResponseDto> {
    return this.quotesService.findOne(id);
  }
}

// Rules:
// - Strict TypeScript (strict: true in tsconfig)
// - Class-validator for DTO validation
// - Swagger/OpenAPI documentation
// - Repository pattern for data access
// - Exception filters for error handling
// - Interceptors for response transformation
// - Guards for authentication/authorization
// - Pipes for validation/transformation
```

#### 1.2.3 Go (High-Performance Services)

```go
// File Naming: snake_case for files, PascalCase for exported
// File: billing_service.go, rating_engine.go
// Package: billing, rating, usage

// Service Example
package billing

type BillingService struct {
    repo       BillingRepository
    cache      CacheClient
    publisher  EventPublisher
    logger     *zap.Logger
}

func NewBillingService(
    repo BillingRepository,
    cache CacheClient,
    publisher EventPublisher,
    logger *zap.Logger,
) *BillingService {
    return &BillingService{
        repo:      repo,
        cache:     cache,
        publisher: publisher,
        logger:    logger,
    }
}

func (s *BillingService) GenerateInvoice(
    ctx context.Context,
    req *GenerateInvoiceRequest,
) (*Invoice, error) {
    // Implementation
}

// Rules:
// - Effective Go guidelines
// - gofmt for formatting
// - golangci-lint for linting
// - Error wrapping with fmt.Errorf
// - Context for cancellation/timeout
// - Interface for dependencies
// - Table-driven tests
// - No global variables
// - Handle errors explicitly
```

#### 1.2.4 Python (AI/ML & Data Processing)

```python
# File Naming: snake_case
# File: revenue_predictor.py, usage_analyzer.py
# Class: PascalCase
# Function/Variable: snake_case
# Constant: UPPER_SNAKE_CASE

# Service Example
from dataclasses import dataclass
from typing import Optional

@dataclass
class RevenuePrediction:
    opportunity_id: str
    predicted_amount: float
    confidence: float
    factors: dict[str, float]

class RevenuePredictor:
    def __init__(
        self,
        model_repository: ModelRepository,
        feature_store: FeatureStore,
        logger: logging.Logger,
    ):
        self._model_repo = model_repository
        self._feature_store = feature_store
        self._logger = logger
    
    async def predict(
        self,
        opportunity_id: str,
    ) -> RevenuePrediction:
        # Implementation
        pass

# Rules:
# - PEP 8 style guide
# - Type hints (mypy)
# - Black for formatting
# - Ruff for linting
# - Pydantic for data validation
# - FastAPI for API endpoints
# - pytest for testing
# - Async/await for I/O operations
```

### 1.3 Git Standards

```yaml
Branch Strategy: Git Flow
  ├─ main: Production-ready code
  ├─ develop: Integration branch
  ├─ feature/REVENUE-123-short-description
  ├─ bugfix/REVENUE-456-fix-invoice-calculation
  ├─ hotfix/REVENUE-789-critical-payment-bug
  ─ release/v1.2.0

Commit Message Format (Conventional Commits):
  <type>(<scope>): <subject>
  
  Types:
  ├─ feat: New feature
  ├─ fix: Bug fix
  ├─ refactor: Code restructuring
  ├─ test: Adding tests
  ├─ docs: Documentation
  ├─ chore: Maintenance
  ├─ perf: Performance improvement
  └─ ci: CI/CD changes
  
  Example:
  feat(billing): add invoice batch processing
  fix(quote): correct discount calculation for tiered pricing
  refactor(auth): migrate to JWT-based authentication

Pull Request Requirements:
  ├─ At least 2 approvals
  ├─ All CI checks pass
  ├─ Code coverage > 80%
  ├─ No merge conflicts
  └─ Linked to JIRA ticket
```

---

## 2. Module Design

### 2.1 System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    React (TypeScript)                         │  │
│  │  ┌──────────┐ ──────────┐ ┌──────────┐ ──────────────┐  │  │
│  │  │ Opportunity│ │   Quote  │ │  Order   │ │   Billing    │  │  │
│  │  │  Module   │ │  Module  │ │  Module  │ │   Module     │  │  │
│  │  └────────── └──────────┘ └──────────┘ └──────────────┘  │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │  │
│  │  │ Contract │ │  Asset   │ │  Report  │ │    Auth      │  │  │
│  │  │  Module  │ │  Module  │ │  Module  │ │   Module     │  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │ HTTP/GraphQL
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              NestJS (API Gateway / BFF)                       │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Authentication │ Authorization │ Rate Limiting      │  │  │
│  │  ──────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Request Validation │ Response Transformation        │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  API Aggregation │ GraphQL Federation                 │  │  │
│  │  └──────────────────────────────────────────────────────  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │ gRPC / HTTP / Kafka
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  MICROSERVICES LAYER                                 │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  NestJS      │  │     Go       │  │    Python    │            │
│  │              │  │              │  │              │            │
│  │ • CPQ Service│  │ • Billing    │  │ • AI/ML      │            │
│  │ • Contract   │  │   Engine     │  │   Service    │            │
│  │ • Order      │  │ • Rating     │  │ • Analytics  │            │
│  │ • Catalog    │  │   Engine     │  │ • Data       │            │
│  │              │  │ • Usage      │  │   Processing │            │
│  │              │  │   Processor  │  │              │            │
│  └──────────────┘  └──────────────┘  └──────────────            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  PostgreSQL  │  │    Redis     │  │    Kafka     │            │
│  │  (Primary    │  │  (Cache +    │  │  (Event      │            │
│  │   Database)  │  │   Session)   │  │   Stream)    │            │
│  └──────────────┘  └──────────────  └──────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Boundary Definition

| Service | Tech | Responsibility | SLA |
|---------|------|----------------|-----|
| **API Gateway** | NestJS | Request routing, auth, aggregation | 99.99% |
| **CPQ Service** | NestJS | Quote creation, pricing, validation | 99.9% |
| **Contract Service** | NestJS | Contract lifecycle, e-signature | 99.9% |
| **Order Service** | NestJS | Order management, activation | 99.9% |
| **Catalog Service** | NestJS | Product catalog, pricing | 99.9% |
| **Billing Engine** | Go | Invoice generation, batch processing | 99.95% |
| **Rating Engine** | Go | Usage rating, pricing calculation | 99.95% |
| **Usage Processor** | Go | High-volume usage ingestion | 99.95% |
| **AI/ML Service** | Python | Revenue prediction, insights | 99.5% |
| **Analytics Service** | Python | Reports, dashboards, ETL | 99.5% |
| **Notification Service** | NestJS | Email, SMS, push notifications | 99.9% |
| **Payment Service** | NestJS | Payment processing, reconciliation | 99.95% |

### 2.3 Inter-Service Communication

```
─────────────────────────────────────────────────────────────────────┐
│              COMMUNICATION PATTERNS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SYNCHRONOUS (Request-Response)                                      │
│  ├─ Protocol: HTTP/REST, gRPC, GraphQL                              │
│  ├─ Use Cases:                                                       │
│  │  ├─ API Gateway → Microservices                                  │
│  │  ├─ CPQ → Catalog (get pricing)                                  │
│  │  ├─ Order → Contract (validate)                                  │
│  │  └─ Frontend → API Gateway                                       │
│  └─ Timeout: 30 seconds default                                      │
│                                                                      │
│  ASYNCHRONOUS (Event-Driven)                                         │
│  ├─ Protocol: Kafka, Redis Pub/Sub                                   │
│  ├─ Use Cases:                                                       │
│  │  ├─ Order Created → Billing (generate invoice)                   │
│  │  ├─ Payment Received → Order (update status)                     │
│  │  ├─ Contract Signed → Asset (create assets)                      │
│  │  └─ Usage Recorded → Rating (calculate charges)                  │
│  └─ Retention: 7 days                                                │
│                                                                      │
│  STREAMING (Real-time)                                               │
│  ├─ Protocol: Kafka Streams, WebSockets                              │
│  ├─ Use Cases:                                                       │
│  │  ├─ Real-time usage monitoring                                   │
│  │  ├─ Live dashboard updates                                       │
│  │  └─ Real-time notifications                                      │
│  └─ Retention: 24 hours                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 Module Dependency Matrix

```
                    CPQ  Contract  Order  Billing  Rating  Usage  AI/ML  Catalog
CPQ                -      R        R      -        -       -      R      R
Contract           R      -        R      -        -       -      -      -
Order              R      R        -      W        -       -      -      R
Billing            -      -        R      -        R       R      -      R
Rating             -      -        -      R        -       R      -      R
Usage              -      -        -      R        R       -      -      -
AI/ML              R      R        R      R        R       R      -      R
Catalog            -      -        -      -        -       -      -      -

Legend: R = Reads from, W = Writes to, - = No direct dependency
```

---

## 3. Domain Design

### 3.1 Domain-Driven Design (DDD) Overview

```
─────────────────────────────────────────────────────────────────────
│                    DOMAIN BOUNDARIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CORE DOMAINS (Business Differentiators)                             │
│  ├─ Revenue Management                                               │
│  ├─ Billing & Invoicing                                              │
│  └─ CPQ (Configure, Price, Quote)                                    │
│                                                                      │
│  SUPPORTING DOMAINS                                                  │
│  ├─ Contract Management                                              │
│  ├─ Order Management                                                 │
│  ├─ Asset Management                                                 │
│  └─ Subscription Management                                          │
│                                                                      │
│  GENERIC DOMAINS                                                     │
│  ├─ Authentication & Authorization                                   │
│  ├─ Notification                                                     │
│  ├─ File Storage                                                     │
│  └─ Audit & Compliance                                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Bounded Contexts

#### 3.2.1 CPQ Context

```typescript
// Entities
class Opportunity {
  id: string;
  name: string;
  accountId: string;
  stage: OpportunityStage;
  amount: Money;
  closeDate: Date;
  probability: number;
}

class Quote {
  id: string;
  opportunityId: string;
  name: string;
  status: QuoteStatus;
  expirationDate: Date;
  lineItems: QuoteLineItem[];
  totalAmount: Money;
  
  // Domain Methods
  addLineItem(item: QuoteLineItem): void;
  removeLineItem(itemId: string): void;
  calculateTotal(): Money;
  submitForApproval(): void;
  approve(approver: User): void;
  reject(approver: User, reason: string): void;
}

class QuoteLineItem {
  id: string;
  quoteId: string;
  productId: string;
  quantity: number;
  unitPrice: Money;
  discount: Discount;
  totalPrice: Money;
}

// Value Objects
class Money {
  amount: number;
  currency: string;
  
  add(other: Money): Money;
  subtract(other: Money): Money;
  multiply(factor: number): Money;
}

class Discount {
  type: 'percentage' | 'fixed';
  value: number;
  
  applyTo(price: Money): Money;
}

// Aggregates
class QuoteAggregate {
  quote: Quote;
  lineItems: QuoteLineItem[];
  approvals: Approval[];
  
  // Aggregate Root methods
  validate(): ValidationResult;
  calculatePricing(pricingEngine: PricingEngine): void;
}
```

#### 3.2.2 Billing Context

```go
// Entities (Go)
type Invoice struct {
    ID              string
    InvoiceNumber   string
    AccountID       string
    Status          InvoiceStatus
    IssueDate       time.Time
    DueDate         time.Time
    LineItems       []InvoiceLineItem
    TotalAmount     Money
    TaxAmount       Money
    DiscountAmount  Money
    NetAmount       Money
    Currency        string
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

type InvoiceLineItem struct {
    ID              string
    InvoiceID       string
    Description     string
    Quantity        float64
    UnitPrice       Money
    Discount        float64
    TaxRate         float64
    TotalAmount     Money
}

type BillingSchedule struct {
    ID              string
    OrderID         string
    BillingType     BillingType
    Frequency       BillingFrequency
    StartDate       time.Time
    EndDate         time.Time
    Amount          Money
    NextBillingDate time.Time
    Status          ScheduleStatus
}

// Domain Events
type InvoiceGenerated struct {
    InvoiceID     string
    AccountID     string
    Amount        Money
    GeneratedAt   time.Time
}

type PaymentReceived struct {
    PaymentID     string
    InvoiceID     string
    Amount        Money
    ReceivedAt    time.Time
}
```

#### 3.2.3 Usage & Rating Context

```python
# Entities (Python)
@dataclass
class UsageRecord:
    id: str
    subscription_id: str
    asset_id: str
    usage_date: datetime
    quantity: Decimal
    unit_of_measure: str
    status: UsageStatus
    created_at: datetime

@dataclass
class RatingResult:
    usage_record_id: str
    rate: Decimal
    calculated_amount: Decimal
    rating_rules_applied: list[str]
    rated_at: datetime

@dataclass
class RatingRule:
    id: str
    product_id: str
    tier_type: TierType
    tiers: list[Tier]
    effective_date: datetime
    expiration_date: Optional[datetime]

@dataclass
class Tier:
    from_quantity: Decimal
    to_quantity: Optional[Decimal]
    rate: Decimal
```

### 3.3 Domain Events Catalog

```yaml
# Domain Events
OpportunityCreated:
  payload:
    opportunity_id: string
    account_id: string
    amount: decimal
    close_date: datetime
  published_by: OpportunityService
  consumed_by: [AI/ML Service, Notification Service]

QuoteSubmitted:
  payload:
    quote_id: string
    opportunity_id: string
    total_amount: decimal
    submitted_by: string
  published_by: CPQ Service
  consumed_by: [Contract Service, Notification Service]

QuoteApproved:
  payload:
    quote_id: string
    approved_by: string
    approved_at: datetime
  published_by: CPQ Service
  consumed_by: [Order Service, Notification Service]

OrderActivated:
  payload:
    order_id: string
    account_id: string
    activation_date: datetime
  published_by: Order Service
  consumed_by: [Billing Service, Asset Service, Contract Service]

InvoiceGenerated:
  payload:
    invoice_id: string
    account_id: string
    amount: decimal
    due_date: datetime
  published_by: Billing Service
  consumed_by: [Notification Service, Analytics Service, Payment Service]

PaymentReceived:
  payload:
    payment_id: string
    invoice_id: string
    amount: decimal
    received_at: datetime
  published_by: Payment Service
  consumed_by: [Billing Service, Order Service, Analytics Service]

ContractSigned:
  payload:
    contract_id: string
    account_id: string
    signed_at: datetime
  published_by: Contract Service
  consumed_by: [Order Service, Asset Service, Notification Service]

UsageRecorded:
  payload:
    usage_id: string
    subscription_id: string
    quantity: decimal
    recorded_at: datetime
  published_by: Usage Processor
  consumed_by: [Rating Engine, Analytics Service]
```

---

## 4. Folder Structure

### 4.1 React Frontend

```
revenue-management-frontend/
├── public/
│   ├── index.html
│   └── assets/
── src/
│   ├── api/                          # API client configuration
│   │   ├── client.ts                 # Axios instance
│   │   ├── endpoints.ts              # API endpoints
│   │   └── interceptors.ts           # Request/response interceptors
│   │
│   ├── assets/                       # Static assets
│   │   ├── images/
│   │   ├── icons/
│   │   └── styles/
│   │       ├── globals.css
│   │       └── variables.css
│   │
│   ├── components/                   # Shared components
│   │   ├── ui/                       # Primitive UI components
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   ├── Modal/
│   │   │   ├── Table/
│   │   │   └── index.ts
│   │   ├── forms/                    # Form components
│   │   ├── charts/                   # Chart components
│   │   └── layout/                   # Layout components
│   │
│   ├── features/                     # Feature modules (by domain)
│   │   ├── opportunity/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   ├── quote/
│   │   │   ├── components/
│   │   │   │   ├── QuoteBuilder/
│   │   │   │   ├── QuoteLineEditor/
│   │   │   │   └── QuoteApproval/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   ├── types/
│   │   │   └── utils/
│   │   ├── order/
│   │   ├── billing/
│   │   ├── contract/
│   │   ├── asset/
│   │   ── report/
│   │
│   ├── hooks/                        # Shared custom hooks
│   │   ├── useAuth.ts
│   │   ├── useDebounce.ts
│   │   ├── useLocalStorage.ts
│   │   └── usePermissions.ts
│   │
│   ├── lib/                          # Third-party library configs
│   │   ├── react-query.ts
│   │   ├── zustand.ts
│   │   └── i18n.ts
│   │
│   ├── pages/                        # Page components (routing)
│   │   ├── DashboardPage.tsx
│   │   ├── OpportunityListPage.tsx
│   │   ├── QuoteDetailPage.tsx
│   │   └── ...
│   │
│   ├── store/                        # Global state
│   │   ├── authStore.ts
│   │   ├── uiStore.ts
│   │   ── index.ts
│   │
│   ├── types/                        # Global type definitions
│   │   ├── api.ts
│   │   ├── common.ts
│   │   └── index.ts
│   │
│   ├── utils/                        # Utility functions
│   │   ├── format.ts
│   │   ├── validation.ts
│   │   └── constants.ts
│   │
│   ├── App.tsx
│   ├── main.tsx
│   └── vite-env.d.ts
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── vite.config.ts
└── package.json
```

### 4.2 NestJS Backend

```
revenue-management-api/
├── src/
│   ├── common/                       # Shared utilities
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   └── roles.decorator.ts
│   │   ├── dto/
│   │   │   ├── pagination.dto.ts
│   │   │   └── response.dto.ts
│   │   ├── enums/
│   │   ├── exceptions/
│   │   │   ├── business.exception.ts
│   │   │   ── validation.exception.ts
│   │   ├── filters/
│   │   │   ── http-exception.filter.ts
│   │   ├── guards/
│   │   │   ├── auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   └── transform.interceptor.ts
│   │   ├── interfaces/
│   │   ├── pipes/
│   │   │   └── validation.pipe.ts
│   │   └── utils/
│   │
│   ├── config/                       # Configuration
│   │   ├── app.config.ts
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   ├── kafka.config.ts
│   │   └── index.ts
│   │
│   ├── database/                     # Database layer
│   │   ├── entities/
│   │   │   ├── opportunity.entity.ts
│   │   │   ├── quote.entity.ts
│   │   │   ├── order.entity.ts
│   │   │   ── ...
│   │   ├── migrations/
│   │   ├── repositories/
│   │   │   ├── opportunity.repository.ts
│   │   │   └── ...
│   │   └── seeds/
│   │
│   ├── modules/                      # Feature modules
│   │   ├── auth/
│   │   │   ├── dto/
│   │   │   ├── guards/
│   │   │   ├── strategies/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.module.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── cpq/
│   │   │   ├── dto/
│   │   │   │   ├── create-quote.dto.ts
│   │   │   │   ├── update-quote.dto.ts
│   │   │   │   └── quote-response.dto.ts
│   │   │   ├── entities/
│   │   │   ├── repositories/
│   │   │   ├── services/
│   │   │   │   ├── quote.service.ts
│   │   │   │   ├── pricing.service.ts
│   │   │   │   ── approval.service.ts
│   │   │   ├── controllers/
│   │   │   │   └── quote.controller.ts
│   │   │   ├── events/
│   │   │   ├── cpq.module.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── contract/
│   │   ├── order/
│   │   ├── billing/
│   │   ├── catalog/
│   │   ├── payment/
│   │   └── notification/
│   │
│   ├── shared/                       # Shared modules
│   │   ├── cache/
│   │   │   ├── cache.module.ts
│   │   │   └── cache.service.ts
│   │   ├── event/
│   │   │   ├── event.module.ts
│   │   │   └── event.service.ts
│   │   ├── logger/
│   │   ── http/
│   │
│   ├── app.module.ts
│   ├── main.ts
│   └── bootstrap.ts
│
├── test/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docker/
│   ├── Dockerfile
│   ── docker-compose.yml
│
├── .eslintrc.js
├── .prettierrc
├── nest-cli.json
├── tsconfig.json
└── package.json
```

### 4.3 Go Services

```
revenue-management-go/
├── cmd/                              # Application entry points
│   ├── billing-service/
│   │   ── main.go
│   ├── rating-engine/
│   │   └── main.go
│   └── usage-processor/
│       ── main.go
│
├── internal/                         # Private application code
│   ├── billing/
│   │   ├── domain/
│   │   │   ├── invoice.go
│   │   │   ├── billing_schedule.go
│   │   │   └── invoice_line_item.go
│   │   ├── dto/
│   │   │   ├── request.go
│   │   │   └── response.go
│   │   ├── handler/
│   │   │   ├── http_handler.go
│   │   │   └── grpc_handler.go
│   │   ├── repository/
│   │   │   ├── invoice_repository.go
│   │   │   └── postgres_repository.go
│   │   ├── service/
│   │   │   ├── billing_service.go
│   │   │   └── invoice_generator.go
│   │   └── usecase/
│   │       └── generate_invoice_usecase.go
│   │
│   ├── rating/
│   │   ├── domain/
│   │   ├── dto/
│   │   ├── handler/
│   │   ├── repository/
│   │   ├── service/
│   │   │   ├── rating_service.go
│   │   │   └── pricing_calculator.go
│   │   ── usecase/
│   │
│   ├── usage/
│   │   ├── domain/
│   │   ├── dto/
│   │   ├── handler/
│   │   ├── repository/
│   │   ├── service/
│   │   │   ├── usage_service.go
│   │   │   └── usage_aggregator.go
│   │   └── usecase/
│   │
│   ── pkg/                          # Internal shared packages
│       ├── money/
│       │   ── money.go
│       ├── validator/
│       ├── logger/
│       └── config/
│
├── pkg/                              # Public shared packages
│   ├── event/
│   │   ├── kafka/
│   │   │   ├── producer.go
│   │   │   └── consumer.go
│   │   └── events.go
│   ├── cache/
│   │   └── redis.go
│   ├── database/
│   │   └── postgres.go
│   └── middleware/
│       ├── auth.go
│       ├── logging.go
│       └── recovery.go
│
├── api/                              # API definitions
│   ├── proto/
│   │   ├── billing.proto
│   │   ├── rating.proto
│   │   └── usage.proto
│   └── openapi/
│       └── billing.yaml
│
├── configs/                          # Configuration files
│   ├── config.yaml
│   └── config.local.yaml
│
├── deployments/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── k8s/
│
├── scripts/
│   ├── migrate.sh
│   └── seed.sh
│
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### 4.4 Python Services

```
revenue-management-python/
├── src/
│   ├── ai_service/                   # AI/ML Service
│   │   ├── __init__.py
│   │   ├── main.py                   # FastAPI app
│   │   ├── api/
│   │   │   ├── routes/
│   │   │   │   ├── prediction.py
│   │   │   │   └── insights.py
│   │   │   └── dependencies.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── security.py
│   │   │   └── logging.py
│   │   ├── domain/
│   │   │   ├── models.py
│   │   │   └── schemas.py
│   │   ├── services/
│   │   │   ├── predictor.py
│   │   │   ├── feature_store.py
│   │   │   └── model_registry.py
│   │   ├── repositories/
│   │   ├── ml/
│   │   │   ├── models/
│   │   │   │   ├── revenue_predictor.py
│   │   │   │   └── churn_predictor.py
│   │   │   ├── training/
│   │   │   └── evaluation/
│   │   └── utils/
│   │
│   ├── analytics_service/            # Analytics Service
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── api/
│   │   ├── core/
│   │   ├── domain/
│   │   ├── services/
│   │   │   ├── report_generator.py
│   │   │   ├── etl_pipeline.py
│   │   │   └── dashboard_service.py
│   │   ├── repositories/
│   │   └── utils/
│   │
│   └── shared/                       # Shared code
│       ├── __init__.py
│       ├── database/
│       │   ├── connection.py
│       │   └── base.py
│       ├── cache/
│       │   └── redis_client.py
│       ├── event/
│       │   └── kafka_client.py
│       ├── logging/
│       │   └── setup.py
│       └── utils/
│           ├── money.py
│           └── datetime_utils.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
│
├── notebooks/                        # Jupyter notebooks
│   ├── revenue_analysis.ipynb
│   └── model_training.ipynb
│
├── configs/
│   ├── config.yaml
│   └── config.local.yaml
│
├── deployments/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── requirements.txt
├── requirements-dev.txt
── pyproject.toml
├── Makefile
└── README.md
```

---

## 5. Design Patterns

### 5.1 Architectural Patterns

#### 5.1.1 Repository Pattern

```typescript
// NestJS - Repository Pattern
export interface IQuoteRepository {
  findById(id: string): Promise<Quote | null>;
  findByOpportunityId(opportunityId: string): Promise<Quote[]>;
  save(quote: Quote): Promise<Quote>;
  update(quote: Quote): Promise<Quote>;
  delete(id: string): Promise<void>;
  findWithFilters(filters: QuoteFilter): Promise<PaginatedResult<Quote>>;
}

@Injectable()
export class QuoteRepository implements IQuoteRepository {
  constructor(
    @InjectRepository(QuoteEntity)
    private readonly quoteEntityRepo: Repository<QuoteEntity>,
  ) {}

  async findById(id: string): Promise<Quote | null> {
    const entity = await this.quoteEntityRepo.findOne({
      where: { id },
      relations: ['lineItems', 'approvals'],
    });
    return entity ? QuoteMapper.toDomain(entity) : null;
  }

  async save(quote: Quote): Promise<Quote> {
    const entity = QuoteMapper.toEntity(quote);
    const saved = await this.quoteEntityRepo.save(entity);
    return QuoteMapper.toDomain(saved);
  }
}
```

```go
// Go - Repository Pattern
type InvoiceRepository interface {
    Create(ctx context.Context, invoice *Invoice) error
    FindByID(ctx context.Context, id string) (*Invoice, error)
    FindByAccountID(ctx context.Context, accountID string, limit, offset int) ([]Invoice, error)
    Update(ctx context.Context, invoice *Invoice) error
    Delete(ctx context.Context, id string) error
    FindPendingInvoices(ctx context.Context) ([]Invoice, error)
}

type postgresInvoiceRepository struct {
    db *sql.DB
}

func NewPostgresInvoiceRepository(db *sql.DB) InvoiceRepository {
    return &postgresInvoiceRepository{db: db}
}

func (r *postgresInvoiceRepository) Create(ctx context.Context, invoice *Invoice) error {
    query := `INSERT INTO invoices (id, invoice_number, account_id, status, issue_date, due_date, total_amount, currency, created_at)
              VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)`
    
    _, err := r.db.ExecContext(ctx, query,
        invoice.ID,
        invoice.InvoiceNumber,
        invoice.AccountID,
        invoice.Status,
        invoice.IssueDate,
        invoice.DueDate,
        invoice.TotalAmount.Amount,
        invoice.TotalAmount.Currency,
        invoice.CreatedAt,
    )
    
    if err != nil {
        return fmt.Errorf("creating invoice: %w", err)
    }
    return nil
}
```

#### 5.1.2 Strategy Pattern

```typescript
// Pricing Strategy
export interface IPricingStrategy {
  calculate(lineItem: QuoteLineItem, context: PricingContext): Money;
}

@Injectable()
export class StandardPricingStrategy implements IPricingStrategy {
  calculate(lineItem: QuoteLineItem, context: PricingContext): Money {
    const basePrice = lineItem.unitPrice.multiply(lineItem.quantity);
    const discount = lineItem.discount.applyTo(basePrice);
    return basePrice.subtract(discount);
  }
}

@Injectable()
export class TieredPricingStrategy implements IPricingStrategy {
  constructor(private readonly tierService: TierService) {}

  calculate(lineItem: QuoteLineItem, context: PricingContext): Money {
    const tiers = this.tierService.getTiers(lineItem.productId);
    let totalAmount = new Money(0, lineItem.unitPrice.currency);
    let remainingQuantity = lineItem.quantity;
    
    for (const tier of tiers) {
      if (remainingQuantity <= 0) break;
      
      const tierQuantity = Math.min(
        remainingQuantity,
        tier.maxQuantity - tier.minQuantity + 1
      );
      
      const tierAmount = tier.rate.multiply(tierQuantity);
      totalAmount = totalAmount.add(tierAmount);
      remainingQuantity -= tierQuantity;
    }
    
    return totalAmount;
  }
}

@Injectable()
export class VolumeDiscountStrategy implements IPricingStrategy {
  calculate(lineItem: QuoteLineItem, context: PricingContext): Money {
    const basePrice = lineItem.unitPrice.multiply(lineItem.quantity);
    const discountPercentage = this.getVolumeDiscount(lineItem.quantity);
    const discount = basePrice.multiply(discountPercentage / 100);
    return basePrice.subtract(discount);
  }
  
  private getVolumeDiscount(quantity: number): number {
    if (quantity >= 1000) return 20;
    if (quantity >= 500) return 15;
    if (quantity >= 100) return 10;
    return 0;
  }
}

// Strategy Factory
@Injectable()
export class PricingStrategyFactory {
  constructor(
    private readonly standardStrategy: StandardPricingStrategy,
    private readonly tieredStrategy: TieredPricingStrategy,
    private readonly volumeStrategy: VolumeDiscountStrategy,
  ) {}

  getStrategy(pricingType: PricingType): IPricingStrategy {
    switch (pricingType) {
      case PricingType.STANDARD:
        return this.standardStrategy;
      case PricingType.TIERED:
        return this.tieredStrategy;
      case PricingType.VOLUME_DISCOUNT:
        return this.volumeStrategy;
      default:
        return this.standardStrategy;
    }
  }
}
```

#### 5.1.3 Observer Pattern (Event-Driven)

```typescript
// NestJS - Event Emitter
export class QuoteDomainEvents {
  static readonly QUOTE_CREATED = 'quote.created';
  static readonly QUOTE_UPDATED = 'quote.updated';
  static readonly QUOTE_SUBMITTED = 'quote.submitted';
  static readonly QUOTE_APPROVED = 'quote.approved';
  static readonly QUOTE_REJECTED = 'quote.rejected';
}

@Injectable()
export class QuoteService {
  constructor(
    private readonly quoteRepository: IQuoteRepository,
    private readonly eventEmitter: EventEmitter2,
    private readonly kafkaProducer: KafkaProducerService,
  ) {}

  async submitForApproval(quoteId: string, userId: string): Promise<Quote> {
    const quote = await this.quoteRepository.findById(quoteId);
    if (!quote) {
      throw new NotFoundException(`Quote ${quoteId} not found`);
    }
    
    quote.submitForApproval();
    const updatedQuote = await this.quoteRepository.update(quote);
    
    // Local event (in-process)
    this.eventEmitter.emit(QuoteDomainEvents.QUOTE_SUBMITTED, {
      quoteId: quote.id,
      submittedBy: userId,
      submittedAt: new Date(),
    });
    
    // Distributed event (Kafka)
    await this.kafkaProducer.publish('quote-events', {
      event: 'QuoteSubmitted',
      payload: {
        quoteId: quote.id,
        opportunityId: quote.opportunityId,
        totalAmount: quote.totalAmount,
        submittedBy: userId,
      },
      timestamp: new Date().toISOString(),
    });
    
    return updatedQuote;
  }
}
```

#### 5.1.4 Circuit Breaker Pattern

```go
// Go - Circuit Breaker
type CircuitBreaker struct {
    name          string
    state         CircuitState
    failureCount  int
    successCount  int
    lastFailure   time.Time
    threshold     int
    timeout       time.Duration
    mu            sync.RWMutex
}

type CircuitState int

const (
    Closed CircuitState = iota
    Open
    HalfOpen
)

func (cb *CircuitBreaker) Execute(ctx context.Context, fn func() error) error {
    cb.mu.RLock()
    state := cb.state
    cb.mu.RUnlock()
    
    switch state {
    case Open:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.mu.Lock()
            cb.state = HalfOpen
            cb.mu.Unlock()
        } else {
            return ErrCircuitOpen
        }
    case HalfOpen:
        // Allow one request through
    }
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failureCount++
        cb.lastFailure = time.Now()
        if cb.failureCount >= cb.threshold {
            cb.state = Open
        }
        return err
    }
    
    cb.successCount++
    cb.failureCount = 0
    if cb.state == HalfOpen {
        cb.state = Closed
    }
    return nil
}

// Usage
breaker := NewCircuitBreaker("payment-gateway", 5, 30*time.Second)

err := breaker.Execute(ctx, func() error {
    return paymentGateway.Charge(ctx, request)
})
```

#### 5.1.5 Saga Pattern (Distributed Transactions)

```typescript
// NestJS - Saga Pattern for Order Creation
@Injectable()
export class OrderCreationSaga {
  constructor(
    private readonly orderService: OrderService,
    private readonly billingService: BillingService,
    private readonly assetService: AssetService,
    private readonly contractService: ContractService,
    private readonly sagaOrchestrator: SagaOrchestrator,
  ) {}

  async executeOrderCreation(orderId: string): Promise<void> {
    const saga = this.sagaOrchestrator.createSaga('OrderCreation');
    
    // Step 1: Activate Order
    saga.addStep({
      name: 'ActivateOrder',
      execute: () => this.orderService.activate(orderId),
      compensate: () => this.orderService.deactivate(orderId),
    });
    
    // Step 2: Create Billing Schedule
    saga.addStep({
      name: 'CreateBillingSchedule',
      execute: () => this.billingService.createSchedule(orderId),
      compensate: () => this.billingService.cancelSchedule(orderId),
    });
    
    // Step 3: Generate Assets
    saga.addStep({
      name: 'GenerateAssets',
      execute: () => this.assetService.generateFromOrder(orderId),
      compensate: () => this.assetService.revokeFromOrder(orderId),
    });
    
    // Step 4: Create Contract
    saga.addStep({
      name: 'CreateContract',
      execute: () => this.contractService.createFromOrder(orderId),
      compensate: () => this.contractService.cancel(orderId),
    });
    
    try {
      await saga.execute();
    } catch (error) {
      // Compensation will be executed automatically
      throw new SagaExecutionFailedException(error);
    }
  }
}
```

#### 5.1.6 CQRS Pattern

```typescript
// Command Side
export class CreateQuoteCommand {
  constructor(
    public readonly opportunityId: string,
    public readonly name: string,
    public readonly lineItems: CreateQuoteLineItemDto[],
    public readonly userId: string,
  ) {}
}

@Injectable()
export class CreateQuoteCommandHandler implements ICommandHandler<CreateQuoteCommand> {
  constructor(
    private readonly quoteRepository: IQuoteRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: CreateQuoteCommand): Promise<string> {
    const quote = Quote.create({
      opportunityId: command.opportunityId,
      name: command.name,
      createdBy: command.userId,
    });
    
    for (const item of command.lineItems) {
      quote.addLineItem(QuoteLineItem.create(item));
    }
    
    await this.quoteRepository.save(quote);
    
    this.eventBus.publish(new QuoteCreatedEvent(quote.id));
    
    return quote.id;
  }
}

// Query Side
export class GetQuoteQuery {
  constructor(public readonly quoteId: string) {}
}

@Injectable()
export class GetQuoteQueryHandler implements IQueryHandler<GetQuoteQuery> {
  constructor(
    private readonly quoteReadModel: QuoteReadModel,
  ) {}

  async execute(query: GetQuoteQuery): Promise<QuoteReadDto> {
    return this.quoteReadModel.findById(query.quoteId);
  }
}

// Read Model (Materialized View)
@Injectable()
export class QuoteReadModel {
  constructor(
    @InjectRepository(QuoteReadEntity)
    private readonly readRepo: Repository<QuoteReadEntity>,
  ) {}

  async findById(id: string): Promise<QuoteReadDto> {
    const entity = await this.readRepo.findOne({ where: { id } });
    return QuoteReadMapper.toDto(entity);
  }

  @OnEvent('quote.created')
  @OnEvent('quote.updated')
  async handleQuoteEvent(event: QuoteEvent) {
    // Update read model
    await this.readRepo.save(QuoteReadMapper.toEntity(event));
  }
}
```

---

## 6. Dependency Injection

### 6.1 NestJS DI Container

```typescript
// Module Definition
@Module({
  imports: [
    TypeOrmModule.forFeature([QuoteEntity, QuoteLineItemEntity]),
    RedisModule,
    KafkaModule,
    EventModule,
  ],
  controllers: [QuoteController],
  providers: [
    QuoteService,
    PricingService,
    ApprovalService,
    {
      provide: 'IQuoteRepository',
      useClass: QuoteRepository,
    },
    {
      provide: 'IPricingStrategy',
      useFactory: (config: ConfigService) => {
        const strategy = config.get('pricing.strategy');
        switch (strategy) {
          case 'tiered':
            return new TieredPricingStrategy();
          case 'volume':
            return new VolumeDiscountStrategy();
          default:
            return new StandardPricingStrategy();
        }
      },
      inject: [ConfigService],
    },
  ],
  exports: [QuoteService, 'IQuoteRepository'],
})
export class CpqModule {}

// Service with DI
@Injectable()
export class QuoteService {
  constructor(
    @Inject('IQuoteRepository')
    private readonly quoteRepository: IQuoteRepository,
    private readonly pricingService: PricingService,
    private readonly approvalService: ApprovalService,
    private readonly eventEmitter: EventEmitter2,
    private readonly logger: LoggerService,
    private readonly configService: ConfigService,
  ) {}
}
```

### 6.2 Go DI (Wire)

```go
// wire.go - Dependency Injection with Wire
//go:build wireinject
// +build wireinject

package main

import (
    "github.com/google/wire"
)

func InitializeBillingService(cfg *Config) (*BillingService, error) {
    wire.Build(
        // Providers
        NewPostgresDB,
        NewRedisClient,
        NewKafkaProducer,
        NewLogger,
        
        // Repositories
        NewInvoiceRepository,
        NewBillingScheduleRepository,
        
        // Services
        NewBillingService,
        NewInvoiceGenerator,
        NewPaymentProcessor,
        
        // Handlers
        NewBillingHTTPHandler,
        NewBillingGRPCHandler,
    )
    return &BillingService{}, nil
}

// Generated by wire (wire_gen.go)
func InitializeBillingService(cfg *Config) (*BillingService, error) {
    logger := NewLogger(cfg.LogLevel)
    db := NewPostgresDB(cfg.Database)
    redisClient := NewRedisClient(cfg.Redis)
    kafkaProducer := NewKafkaProducer(cfg.Kafka)
    
    invoiceRepo := NewInvoiceRepository(db)
    scheduleRepo := NewBillingScheduleRepository(db)
    
    invoiceGenerator := NewInvoiceGenerator(invoiceRepo, logger)
    paymentProcessor := NewPaymentProcessor(kafkaProducer, logger)
    
    billingService := NewBillingService(
        invoiceRepo,
        scheduleRepo,
        invoiceGenerator,
        paymentProcessor,
        redisClient,
        logger,
    )
    
    return billingService, nil
}
```

### 6.3 Python DI (Dependency Injector)

```python
# containers.py
from dependency_injector import containers, providers
from src.shared.database.connection import DatabaseConnection
from src.shared.cache.redis_client import RedisClient
from src.shared.event.kafka_client import KafkaClient
from src.ai_service.services.predictor import RevenuePredictor
from src.ai_service.repositories.model_repository import ModelRepository

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    
    # Infrastructure
    database = providers.Singleton(
        DatabaseConnection,
        connection_string=config.database.connection_string,
    )
    
    redis_client = providers.Singleton(
        RedisClient,
        host=config.redis.host,
        port=config.redis.port,
    )
    
    kafka_client = providers.Singleton(
        KafkaClient,
        brokers=config.kafka.brokers,
    )
    
    # Repositories
    model_repository = providers.Factory(
        ModelRepository,
        database=database,
    )
    
    # Services
    revenue_predictor = providers.Factory(
        RevenuePredictor,
        model_repository=model_repository,
        redis_client=redis_client,
    )

# main.py
from src.ai_service.containers import Container

container = Container()
container.config.from_yaml('configs/config.yaml')

app = FastAPI()

@app.on_event('startup')
async def startup():
    await container.redis_client().connect()
    await container.kafka_client().connect()

@app.on_event('shutdown')
async def shutdown():
    await container.redis_client().disconnect()
    await container.kafka_client().disconnect()

@app.get('/predict/{opportunity_id}')
async def predict(
    opportunity_id: str,
    predictor: RevenuePredictor = Depends(container.revenue_predictor),
):
    return await predictor.predict(opportunity_id)
```

---

## 7. Error Handling

### 7.1 Error Hierarchy

```typescript
// NestJS - Error Hierarchy
export class AppException extends Error {
  constructor(
    public readonly code: string,
    public readonly message: string,
    public readonly statusCode: number,
    public readonly details?: any,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class BusinessException extends AppException {
  constructor(code: string, message: string, details?: any) {
    super(code, message, HttpStatus.BAD_REQUEST, details);
  }
}

export class ValidationException extends BusinessException {
  constructor(errors: ValidationError[]) {
    super('VALIDATION_ERROR', 'Validation failed', errors);
  }
}

export class NotFoundException extends AppException {
  constructor(resource: string, id: string) {
    super('NOT_FOUND', `${resource} with id ${id} not found`, HttpStatus.NOT_FOUND);
  }
}

export class ConflictException extends AppException {
  constructor(message: string, details?: any) {
    super('CONFLICT', message, HttpStatus.CONFLICT, details);
  }
}

export class UnauthorizedException extends AppException {
  constructor(message = 'Unauthorized') {
    super('UNAUTHORIZED', message, HttpStatus.UNAUTHORIZED);
  }
}

export class ForbiddenException extends AppException {
  constructor(message = 'Forbidden') {
    super('FORBIDDEN', message, HttpStatus.FORBIDDEN);
  }
}

export class InternalServerException extends AppException {
  constructor(message = 'Internal server error', details?: any) {
    super('INTERNAL_ERROR', message, HttpStatus.INTERNAL_SERVER_ERROR, details);
  }
}
```

### 7.2 Global Exception Filter

```typescript
// NestJS - Global Exception Filter
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    let errorResponse: ErrorResponse;
    
    if (exception instanceof AppException) {
      errorResponse = {
        statusCode: exception.statusCode,
        timestamp: new Date().toISOString(),
        path: request.url,
        code: exception.code,
        message: exception.message,
        details: exception.details,
      };
    } else if (exception instanceof HttpException) {
      const status = exception.getStatus();
      errorResponse = {
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
        code: 'HTTP_ERROR',
        message: exception.message,
      };
    } else {
      // Unexpected error
      this.logger.error('Unexpected error', exception);
      errorResponse = {
        statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
        timestamp: new Date().toISOString(),
        path: request.url,
        code: 'INTERNAL_ERROR',
        message: 'Internal server error',
      };
    }
    
    // Log error
    this.logger.error('Request failed', {
      method: request.method,
      url: request.url,
      statusCode: errorResponse.statusCode,
      error: errorResponse.message,
      stack: exception instanceof Error ? exception.stack : undefined,
    });
    
    response.status(errorResponse.statusCode).json(errorResponse);
  }
}

// Error Response DTO
export interface ErrorResponse {
  statusCode: number;
  timestamp: string;
  path: string;
  code: string;
  message: string;
  details?: any;
}
```

### 7.3 Go Error Handling

```go
// Go - Error Handling
package errors

import (
    "errors"
    "fmt"
    "net/http"
)

type AppError struct {
    Code       string            `json:"code"`
    Message    string            `json:"message"`
    StatusCode int               `json:"statusCode"`
    Details    map[string]interface{} `json:"details,omitempty"`
    Err        error             `json:"-"`
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// Sentinel errors
var (
    ErrNotFound     = errors.New("resource not found")
    ErrConflict     = errors.New("resource conflict")
    ErrValidation   = errors.New("validation failed")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
)

// Error constructors
func NewNotFound(resource, id string) *AppError {
    return &AppError{
        Code:       "NOT_FOUND",
        Message:    fmt.Sprintf("%s with id %s not found", resource, id),
        StatusCode: http.StatusNotFound,
        Err:        ErrNotFound,
    }
}

func NewValidation(errors map[string]string) *AppError {
    return &AppError{
        Code:       "VALIDATION_ERROR",
        Message:    "Validation failed",
        StatusCode: http.StatusBadRequest,
        Details:    map[string]interface{}{"errors": errors},
        Err:        ErrValidation,
    }
}

func NewInternal(message string, err error) *AppError {
    return &AppError{
        Code:       "INTERNAL_ERROR",
        Message:    message,
        StatusCode: http.StatusInternalServerError,
        Err:        err,
    }
}

// Error handler middleware
func ErrorHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                appErr := &AppError{
                    Code:       "PANIC",
                    Message:    "Internal server error",
                    StatusCode: http.StatusInternalServerError,
                }
                writeError(w, appErr)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

func writeError(w http.ResponseWriter, appErr *AppError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(appErr.StatusCode)
    json.NewEncoder(w).Encode(appErr)
}
```

### 7.4 Error Handling Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│              ERROR HANDLING BEST PRACTICES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. NEVER swallow errors                                             │
│     ✗ try { ... } catch (e) { /* ignore */ }                        │
│     ✓ try { ... } catch (e) { logger.error(e); throw e; }           │
│                                                                      │
│  2. Use specific error types                                         │
│     ✗ throw new Error('Something went wrong')                       │
│     ✓ throw new NotFoundException('Quote', id)                      │
│                                                                      │
│  3. Include context in errors                                        │
│     ✗ throw new Error('Failed to save')                             │
│     ✓ throw new Error(`Failed to save quote ${id}: ${err.message}`) │
│                                                                      │
│  4. Handle errors at appropriate level                               │
│     - Repository: Technical errors (DB connection)                   │
│     - Service: Business errors (validation, rules)                   │
│     - Controller: HTTP errors (status codes)                         │
│     - Global: Unexpected errors                                      │
│                                                                      │
│  5. Don't expose internal details                                    │
│     ✗ { error: 'SQL syntax error at line 5' }                       │
│     ✓ { error: 'Invalid request data' }                             │
│                                                                      │
│  6. Use error codes for client handling                              │
│     { code: 'QUOTE_EXPIRED', message: '...' }                       │
│                                                                      │
│  7. Implement retry with backoff for transient errors                │
│  8. Use circuit breaker for external dependencies                    │
│  9. Log errors with correlation IDs                                  │
│  10. Monitor error rates and alert on spikes                         │
│                                                                      │
─────────────────────────────────────────────────────────────────────
```

---

## 8. Logging Strategy

### 8.1 Logging Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LOGGING PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │  Application │───▶│   Log        │───▶│   Central    │         │
│  │  (All        │    │   Aggregator │    │   Log        │         │
│  │   Services)  │    │   (Fluentd/  │    │   Storage    │         │
│  │              │    │    Vector)   │    │   (Elastic/  │         │
│  │              │    │              │    │    S3)       │         │
│  └──────────────┘    └──────────────┘    └──────────────┘         │
│         │                    │                    │                │
│         │                    ▼                    ▼                │
│         │           ┌──────────────┐    ┌──────────────┐         │
│         │           │   Real-time  │    │   Analysis   │         │
│         │           │   Alerting   │    │   & Search   │         │
│         │           │   (PagerDuty)│    │   (Kibana)   │         │
│         │           └──────────────┘    └──────────────┘         │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────┐                                                │
│  │   Structured │                                                │
│  │   Logs       │                                                │
│  │   (JSON)     │                                                │
│  ──────────────┘                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Log Levels

```typescript
// Log Level Definitions
export enum LogLevel {
  ERROR = 'error',    // System failures, data corruption
  WARN  = 'warn',     // Potential issues, degraded performance
  INFO  = 'info',     // Business transactions, important events
  DEBUG = 'debug',    // Detailed processing info (dev only)
  TRACE = 'trace',    // Very detailed tracing (dev only)
}

// Usage Guidelines
const logger = new LoggerService();

// ERROR: System failures
logger.error('Database connection failed', {
  correlationId: 'abc-123',
  service: 'billing-service',
  error: err,
  retryCount: 3,
});

// WARN: Potential issues
logger.warn('API rate limit approaching', {
  correlationId: 'abc-123',
  currentUsage: 950,
  limit: 1000,
  percentage: 95,
});

// INFO: Business transactions
logger.info('Invoice generated', {
  correlationId: 'abc-123',
  invoiceId: 'INV-001',
  accountId: 'ACC-123',
  amount: 1500.00,
  userId: 'user-456',
});

// DEBUG: Detailed info (dev/staging only)
logger.debug('Pricing calculation details', {
  correlationId: 'abc-123',
  lineItems: [...],
  appliedRules: [...],
  calculationSteps: [...],
});
```

### 8.3 Structured Log Format

```json
{
  "timestamp": "2026-06-22T10:30:00.000Z",
  "level": "INFO",
  "service": "billing-service",
  "version": "1.2.3",
  "environment": "production",
  "correlationId": "abc-123-def-456",
  "traceId": "trace-789",
  "spanId": "span-012",
  "userId": "user-456",
  "tenantId": "tenant-789",
  "action": "invoice.generated",
  "message": "Invoice INV-001 generated successfully",
  "metadata": {
    "invoiceId": "INV-001",
    "accountId": "ACC-123",
    "amount": 1500.00,
    "currency": "USD",
    "duration_ms": 245
  },
  "context": {
    "ip": "192.168.1.1",
    "userAgent": "Mozilla/5.0...",
    "region": "ap-southeast-1"
  }
}
```

### 8.4 Language-Specific Logging

#### NestJS (Winston)

```typescript
// logger.service.ts
import { Injectable, LoggerService as NestLoggerService } from '@nestjs/common';
import * as winston from 'winston';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class LoggerService implements NestLoggerService {
  private logger: winston.Logger;
  
  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),
      ),
      transports: [
        new winston.transports.Console(),
        new winston.transports.File({ 
          filename: 'logs/error.log', 
          level: 'error' 
        }),
        new winston.transports.File({ 
          filename: 'logs/combined.log' 
        }),
      ],
    });
  }

  log(message: string, context?: any) {
    this.logger.info(message, { context });
  }

  error(message: string, trace?: string, context?: any) {
    this.logger.error(message, { trace, context });
  }

  warn(message: string, context?: any) {
    this.logger.warn(message, { context });
  }

  debug(message: string, context?: any) {
    this.logger.debug(message, { context });
  }

  // Create child logger with correlation ID
  createCorrelationLogger(correlationId: string) {
    return this.logger.child({ correlationId });
  }
}

// Usage in service
@Injectable()
export class BillingService {
  private readonly logger: winston.Logger;
  
  constructor(private readonly loggerService: LoggerService) {
    this.logger = loggerService.createCorrelationLogger('billing');
  }

  async generateInvoice(invoiceId: string) {
    const startTime = Date.now();
    
    this.logger.info('Starting invoice generation', { invoiceId });
    
    try {
      // ... business logic
      const duration = Date.now() - startTime;
      this.logger.info('Invoice generated successfully', { 
        invoiceId, 
        duration_ms: duration 
      });
    } catch (error) {
      this.logger.error('Failed to generate invoice', { 
        invoiceId, 
        error: error.message,
        stack: error.stack 
      });
      throw error;
    }
  }
}
```

#### Go (Zap)

```go
// logger.go
package logger

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

type Logger struct {
    zap *zap.Logger
}

func NewLogger(level string) *Logger {
    var zapLevel zapcore.Level
    switch level {
    case "debug":
        zapLevel = zap.DebugLevel
    case "warn":
        zapLevel = zap.WarnLevel
    case "error":
        zapLevel = zap.ErrorLevel
    default:
        zapLevel = zap.InfoLevel
    }

    config := zap.Config{
        Level:       zap.NewAtomicLevelAt(zapLevel),
        Development: false,
        Encoding:    "json",
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "timestamp",
            LevelKey:       "level",
            NameKey:        "logger",
            CallerKey:      "caller",
            MessageKey:     "message",
            StacktraceKey:  "stacktrace",
            LineEnding:     zapcore.DefaultLineEnding,
            EncodeLevel:    zapcore.LowercaseLevelEncoder,
            EncodeTime:     zapcore.ISO8601TimeEncoder,
            EncodeDuration: zapcore.StringDurationEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
        },
        OutputPaths:      []string{"stdout"},
        ErrorOutputPaths: []string{"stderr"},
    }

    zapLogger, err := config.Build()
    if err != nil {
        panic(err)
    }

    return &Logger{zap: zapLogger}
}

func (l *Logger) Info(msg string, fields ...zap.Field) {
    l.zap.Info(msg, fields...)
}

func (l *Logger) Error(msg string, fields ...zap.Field) {
    l.zap.Error(msg, fields...)
}

func (l *Logger) With(fields ...zap.Field) *Logger {
    return &Logger{zap: l.zap.With(fields...)}
}

// Usage
logger := logger.NewLogger("info")
correlationLogger := logger.With(
    zap.String("correlationId", "abc-123"),
    zap.String("service", "billing-service"),
)

correlationLogger.Info("Invoice generated",
    zap.String("invoiceId", "INV-001"),
    zap.Float64("amount", 1500.00),
    zap.Int("duration_ms", 245),
)
```

#### Python (structlog)

```python
# logging_setup.py
import structlog
import logging
import sys

def setup_logging(level: str = "INFO", environment: str = "production"):
    shared_processors = [
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    structlog.configure(
        processors=[
            *shared_processors,
            structlog.contextvars.merge_contextvars,
            structlog.processors.JSONRenderer() if environment == "production" 
            else structlog.dev.ConsoleRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, level.upper()),
    )

# Usage
import structlog

logger = structlog.get_logger()

async def generate_invoice(invoice_id: str, account_id: str):
    log = logger.bind(
        invoice_id=invoice_id,
        account_id=account_id,
        correlation_id="abc-123",
    )
    
    log.info("invoice_generation_started")
    
    try:
        # ... business logic
        log.info(
            "invoice_generated_successfully",
            amount=1500.00,
            currency="USD",
            duration_ms=245,
        )
    except Exception as e:
        log.error(
            "invoice_generation_failed",
            error=str(e),
            exc_info=True,
        )
        raise
```

### 8.5 Correlation ID Propagation

```typescript
// NestJS - Correlation ID Middleware
@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = 
      req.headers['x-correlation-id'] as string || 
      uuidv4();
    
    req['correlationId'] = correlationId;
    res.setHeader('X-Correlation-Id', correlationId);
    
    next();
  }
}

// Propagation to downstream services
@Injectable()
export class HttpServiceWithCorrelation {
  constructor(
    private readonly httpService: HttpService,
    @Inject(CORRELATION_ID) private readonly correlationId: string,
  ) {}

  async get<T>(url: string): Promise<T> {
    return this.httpService.get(url, {
      headers: {
        'X-Correlation-Id': this.correlationId,
      },
    }).toPromise();
  }
}
```

---

## 9. Testing Strategy

### 9.1 Testing Pyramid

```
                    ┌─────────┐
                    │   E2E   │  10%
                   ┌┴─────────┴┐
                   │Integration│  20%
                  ┌┴───────────┴┐
                  │    Unit     │  70%
                 ┌┴─────────────┴┐
                 │               │
                 │  Test Pyramid │
                 │               │
                 └───────────────┘
```

### 9.2 Unit Testing

#### NestJS (Jest)

```typescript
// quote.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { QuoteService } from './quote.service';
import { QuoteRepository } from './quote.repository';
import { PricingService } from './pricing.service';
import { NotFoundException } from '../common/exceptions';

describe('QuoteService', () => {
  let service: QuoteService;
  let repository: jest.Mocked<QuoteRepository>;
  let pricingService: jest.Mocked<PricingService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        QuoteService,
        {
          provide: 'IQuoteRepository',
          useValue: {
            findById: jest.fn(),
            save: jest.fn(),
            update: jest.fn(),
          },
        },
        {
          provide: PricingService,
          useValue: {
            calculate: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<QuoteService>(QuoteService);
    repository = module.get('IQuoteRepository');
    pricingService = module.get(PricingService);
  });

  describe('findOne', () => {
    it('should return a quote when found', async () => {
      const quote = {
        id: 'quote-123',
        name: 'Test Quote',
        status: 'DRAFT',
      };
      repository.findById.mockResolvedValue(quote);

      const result = await service.findOne('quote-123');

      expect(result).toEqual(quote);
      expect(repository.findById).toHaveBeenCalledWith('quote-123');
    });

    it('should throw NotFoundException when quote not found', async () => {
      repository.findById.mockResolvedValue(null);

      await expect(service.findOne('non-existent'))
        .rejects
        .toThrow(NotFoundException);
    });
  });

  describe('calculateTotal', () => {
    it('should calculate total from line items', async () => {
      const lineItems = [
        { quantity: 2, unitPrice: 100, discount: 0 },
        { quantity: 1, unitPrice: 50, discount: 10 },
      ];
      pricingService.calculate.mockResolvedValue(245);

      const result = await service.calculateTotal(lineItems);

      expect(result).toBe(245);
      expect(pricingService.calculate).toHaveBeenCalled();
    });
  });
});
```

#### Go (Table-Driven Tests)

```go
// billing_service_test.go
package billing

import (
    "context"
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockInvoiceRepository struct {
    mock.Mock
}

func (m *MockInvoiceRepository) Create(ctx context.Context, invoice *Invoice) error {
    args := m.Called(ctx, invoice)
    return args.Error(0)
}

func TestBillingService_GenerateInvoice(t *testing.T) {
    tests := []struct {
        name        string
        request     *GenerateInvoiceRequest
        mockSetup   func(*MockInvoiceRepository)
        wantErr     bool
        wantInvoice *Invoice
    }{
        {
            name: "successful invoice generation",
            request: &GenerateInvoiceRequest{
                AccountID: "acc-123",
                OrderID:   "ord-456",
                Amount:    Money{Amount: 1500, Currency: "USD"},
            },
            mockSetup: func(repo *MockInvoiceRepository) {
                repo.On("Create", mock.Anything, mock.Anything).
                    Return(nil)
            },
            wantErr: false,
            wantInvoice: &Invoice{
                AccountID: "acc-123",
                OrderID:   "ord-456",
                Status:    InvoiceStatusGenerated,
            },
        },
        {
            name: "duplicate invoice number",
            request: &GenerateInvoiceRequest{
                AccountID:     "acc-123",
                OrderID:       "ord-456",
                InvoiceNumber: "INV-001",
            },
            mockSetup: func(repo *MockInvoiceRepository) {
                repo.On("Create", mock.Anything, mock.Anything).
                    Return(ErrDuplicateInvoice)
            },
            wantErr: true,
        },
        {
            name: "invalid amount",
            request: &GenerateInvoiceRequest{
                AccountID: "acc-123",
                OrderID:   "ord-456",
                Amount:    Money{Amount: 0, Currency: "USD"},
            },
            mockSetup:   func(repo *MockInvoiceRepository) {},
            wantErr:     true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := new(MockInvoiceRepository)
            tt.mockSetup(repo)
            
            service := NewBillingService(repo, nil, nil, nil)
            
            invoice, err := service.GenerateInvoice(context.Background(), tt.request)
            
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, invoice)
                assert.Equal(t, tt.wantInvoice.AccountID, invoice.AccountID)
            }
            
            repo.AssertExpectations(t)
        })
    }
}
```

#### Python (pytest)

```python
# test_predictor.py
import pytest
from unittest.mock import AsyncMock, patch
from src.ai_service.services.predictor import RevenuePredictor
from src.ai_service.domain.models import RevenuePrediction

@pytest.fixture
def mock_model_repository():
    return AsyncMock()

@pytest.fixture
def mock_feature_store():
    return AsyncMock()

@pytest.fixture
def predictor(mock_model_repository, mock_feature_store):
    return RevenuePredictor(mock_model_repository, mock_feature_store)

@pytest.mark.asyncio
class TestRevenuePredictor:
    async def test_predict_success(self, predictor, mock_model_repository, mock_feature_store):
        # Arrange
        opportunity_id = "opp-123"
        mock_model_repository.get_model.return_value = MockModel()
        mock_feature_store.get_features.return_value = {
            "historical_amount": 10000,
            "deal_size": 15000,
            "probability": 0.75,
        }
        
        # Act
        result = await predictor.predict(opportunity_id)
        
        # Assert
        assert isinstance(result, RevenuePrediction)
        assert result.opportunity_id == opportunity_id
        assert result.predicted_amount > 0
        assert 0 <= result.confidence <= 1
        mock_model_repository.get_model.assert_called_once()
        mock_feature_store.get_features.assert_called_once_with(opportunity_id)

    async def test_predict_opportunity_not_found(self, predictor, mock_feature_store):
        # Arrange
        mock_feature_store.get_features.side_effect = OpportunityNotFoundError("opp-999")
        
        # Act & Assert
        with pytest.raises(OpportunityNotFoundError):
            await predictor.predict("opp-999")

    async def test_predict_with_invalid_opportunity_id(self, predictor):
        # Arrange
        invalid_id = ""
        
        # Act & Assert
        with pytest.raises(ValueError, match="Invalid opportunity ID"):
            await predictor.predict(invalid_id)
```

### 9.3 Integration Testing

```typescript
// quote.integration.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { TypeOrmModule } from '@nestjs/typeorm';
import { QuoteModule } from './quote.module';
import { QuoteEntity } from './entities/quote.entity';

describe('QuoteController (Integration)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          host: 'localhost',
          port: 5432,
          username: 'test',
          password: 'test',
          database: 'revenue_test',
          entities: [QuoteEntity],
          synchronize: true,
        }),
        QuoteModule,
      ],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /quotes', () => {
    it('should create a new quote', () => {
      return request(app.getHttpServer())
        .post('/quotes')
        .send({
          opportunityId: 'opp-123',
          name: 'Test Quote',
          lineItems: [
            { productId: 'prod-1', quantity: 2, unitPrice: 100 },
          ],
        })
        .set('Authorization', 'Bearer test-token')
        .expect(201)
        .expect((res) => {
          expect(res.body.id).toBeDefined();
          expect(res.body.name).toBe('Test Quote');
          expect(res.body.status).toBe('DRAFT');
        });
    });

    it('should return 400 for invalid data', () => {
      return request(app.getHttpServer())
        .post('/quotes')
        .send({})
        .set('Authorization', 'Bearer test-token')
        .expect(400);
    });
  });
});
```

### 9.4 E2E Testing

```typescript
// e2e/quote-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Quote Creation Flow', () => {
  test('should complete full quote creation flow', async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');
    
    // Navigate to opportunities
    await page.goto('/opportunities');
    await page.click('text=New Opportunity');
    
    // Create opportunity
    await page.fill('[name="name"]', 'Test Opportunity');
    await page.fill('[name="amount"]', '10000');
    await page.click('button:has-text("Save")');
    
    // Create quote
    await page.click('button:has-text("New Quote")');
    await page.fill('[name="quoteName"]', 'Test Quote');
    
    // Add line items
    await page.click('button:has-text("Add Product")');
    await page.selectOption('[name="product"]', 'Product A');
    await page.fill('[name="quantity"]', '5');
    await page.click('button:has-text("Add")');
    
    // Submit for approval
    await page.click('button:has-text("Submit for Approval")');
    
    // Verify
    await expect(page.locator('text=Quote Submitted')).toBeVisible();
    await expect(page.locator('[data-testid="quote-status"]')).toHaveText('PENDING_APPROVAL');
  });
});
```

### 9.5 Test Coverage Requirements

| Layer | Coverage Target | Tools |
|-------|----------------|-------|
| Unit Tests | > 80% | Jest, Go test, pytest |
| Integration Tests | > 70% | Supertest, Testcontainers |
| E2E Tests | Critical paths | Playwright, Cypress |
| Performance Tests | SLA compliance | k6, Artillery |
| Security Tests | OWASP Top 10 | OWASP ZAP, Snyk |

---

## 10. Security Design

### 10.1 Authentication & Authorization

```typescript
// JWT Authentication
@Injectable()
export class AuthService {
  constructor(
    private readonly jwtService: JwtService,
    private readonly userService: UserService,
    private readonly bcryptService: BcryptService,
  ) {}

  async login(loginDto: LoginDto): Promise<AuthResponse> {
    const user = await this.userService.findByEmail(loginDto.email);
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await this.bcryptService.compare(
      loginDto.password,
      user.passwordHash,
    );
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const tokens = await this.generateTokens(user);
    return {
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      user: UserMapper.toDto(user),
    };
  }

  private async generateTokens(user: User): Promise<TokenPair> {
    const payload = {
      sub: user.id,
      email: user.email,
      roles: user.roles,
      tenantId: user.tenantId,
    };

    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(payload, {
        expiresIn: '15m',
        secret: process.env.JWT_SECRET,
      }),
      this.jwtService.signAsync(payload, {
        expiresIn: '7d',
        secret: process.env.JWT_REFRESH_SECRET,
      }),
    ]);

    return { accessToken, refreshToken };
  }
}

// JWT Guard
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly jwtService: JwtService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      const payload = await this.jwtService.verifyAsync(token);
      request.user = payload;
      return true;
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private extractToken(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}

// Role-Based Guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      'roles',
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}

// Usage
@Controller('quotes')
@UseGuards(JwtAuthGuard)
export class QuoteController {
  @Post()
  @Roles('SALES_REP', 'SALES_MANAGER')
  async create(@Body() dto: CreateQuoteDto, @CurrentUser() user: User) {
    return this.quoteService.create(dto, user);
  }

  @Get(':id')
  @Roles('SALES_REP', 'SALES_MANAGER', 'BILLING_SPECIALIST')
  async findOne(@Param('id') id: string) {
    return this.quoteService.findOne(id);
  }
}
```

### 10.2 Input Validation

```typescript
// DTO Validation with class-validator
export class CreateQuoteDto {
  @IsUUID()
  @IsNotEmpty()
  opportunityId: string;

  @IsString()
  @MinLength(3)
  @MaxLength(100)
  name: string;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => CreateQuoteLineItemDto)
  @ArrayMinSize(1)
  lineItems: CreateQuoteLineItemDto[];

  @IsOptional()
  @IsDateString()
  expirationDate?: string;
}

export class CreateQuoteLineItemDto {
  @IsUUID()
  productId: string;

  @IsNumber()
  @Min(1)
  quantity: number;

  @IsNumber()
  @Min(0)
  unitPrice: number;

  @IsOptional()
  @IsNumber()
  @Min(0)
  @Max(100)
  discount?: number;
}
```

### 10.3 Data Encryption

```typescript
// Encryption Service
@Injectable()
export class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;

  constructor() {
    this.key = crypto.scryptSync(
      process.env.ENCRYPTION_KEY,
      'salt',
      32,
    );
  }

  encrypt(data: string): EncryptedData {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
    };
  }

  decrypt(encryptedData: EncryptedData): string {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.key,
      Buffer.from(encryptedData.iv, 'hex'),
    );
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Encrypt sensitive fields
@Entity()
export class PaymentMethod {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  @Encrypt()  // Custom decorator
  cardNumber: string;

  @Column()
  @Encrypt()
  cvv: string;

  @Column()
  expiryDate: string;
}
```

### 10.4 Rate Limiting

```typescript
// Rate Limiting with Redis
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(
    private readonly redisService: RedisService,
    @Inject(CONFIG_SERVICE) private readonly configService: ConfigService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const key = this.generateKey(request);
    const limit = this.getLimit(context);
    const window = this.getWindow(context);

    const current = await this.redisService.increment(key, window);

    if (current > limit) {
      throw new TooManyRequestsException(
        `Rate limit exceeded. Try again in ${window} seconds`,
      );
    }

    // Set headers
    const response = context.switchToHttp().getResponse();
    response.setHeader('X-RateLimit-Limit', limit);
    response.setHeader('X-RateLimit-Remaining', limit - current);
    response.setHeader('X-RateLimit-Reset', Math.ceil(Date.now() / 1000) + window);

    return true;
  }

  private generateKey(request: Request): string {
    const userId = request.user?.id || 'anonymous';
    const endpoint = request.path;
    return `rate_limit:${userId}:${endpoint}`;
  }

  private getLimit(context: ExecutionContext): number {
    return this.reflector.get('rateLimit', context.getHandler()) || 100;
  }

  private getWindow(context: ExecutionContext): number {
    return this.reflector.get('rateLimitWindow', context.getHandler()) || 60;
  }
}

// Usage
@Controller('api')
export class ApiController {
  @Get('quotes')
  @RateLimit({ limit: 100, window: 60 })  // 100 requests per minute
  async getQuotes() {
    // ...
  }

  @Post('quotes')
  @RateLimit({ limit: 10, window: 60 })  // 10 requests per minute
  async createQuote() {
    // ...
  }
}
```

### 10.5 Security Headers

```typescript
// Helmet Configuration
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'"],
        fontSrc: ["'self'"],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'"],
        frameSrc: ["'none'"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
    },
    referrerPolicy: { policy: 'no-referrer' },
  }),
);

// CORS Configuration
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS.split(','),
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Correlation-Id'],
  credentials: true,
  maxAge: 3600,
});
```

---

## 11. Caching Design

### 11.1 Caching Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CACHING LAYERS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  L1: In-Memory Cache (Application Level)                             │
│  ├─ Technology: Node.js Map, Go sync.Map                            │
│  ├─ Use Case: Session data, frequently accessed objects             │
│  ├─ TTL: 5 minutes                                                  │
│  └─ Size Limit: 100 MB                                              │
│                                                                      │
│  L2: Distributed Cache (Redis)                                       │
│  ├─ Technology: Redis Cluster                                       │
│  ├─ Use Case: Product catalog, pricing, user sessions               │
│  ├─ TTL: 1 hour (configurable)                                      │
│  └─ Size Limit: 10 GB                                               │
│                                                                      │
│  L3: Database Query Cache                                            │
│  ├─ Technology: PostgreSQL Query Cache, Materialized Views          │
│  ├─ Use Case: Complex aggregations, reports                         │
│  ├─ TTL: 15 minutes                                                 │
│  └─ Refresh: Scheduled or event-driven                              │
│                                                                      │
│  L4: CDN Cache                                                       │
│  ├─ Technology: CloudFront, Cloudflare                              │
│  ├─ Use Case: Static assets, public APIs                            │
│  ├─ TTL: 24 hours                                                   │
│  └─ Invalidation: Manual or event-driven                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Redis Cache Implementation

```typescript
// NestJS - Redis Cache Service
@Injectable()
export class CacheService {
  private readonly client: Redis;
  private readonly defaultTTL = 3600; // 1 hour

  constructor(private readonly configService: ConfigService) {
    this.client = new Redis({
      host: configService.get('REDIS_HOST'),
      port: configService.get('REDIS_PORT'),
      password: configService.get('REDIS_PASSWORD'),
      db: configService.get('REDIS_DB'),
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
      maxRetriesPerRequest: 3,
    });
  }

  async get<T>(key: string): Promise<T | null> {
    const data = await this.client.get(key);
    if (!data) return null;
    return JSON.parse(data) as T;
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    const expiration = ttl || this.defaultTTL;
    await this.client.setex(key, expiration, serialized);
  }

  async delete(key: string): Promise<void> {
    await this.client.del(key);
  }

  async deletePattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern);
    if (keys.length > 0) {
      await this.client.del(...keys);
    }
  }

  // Cache-Aside Pattern
  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number,
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached) return cached;

    const data = await fetcher();
    await this.set(key, data, ttl);
    return data;
  }

  // Cache-Invalidation on Update
  async invalidateRelated(entityType: string, entityId: string): Promise<void> {
    await this.deletePattern(`${entityType}:${entityId}:*`);
    await this.deletePattern(`${entityType}:list:*`);
  }
}

// Usage in Service
@Injectable()
export class ProductService {
  constructor(
    private readonly productRepository: ProductRepository,
    private readonly cacheService: CacheService,
  ) {}

  async findById(id: string): Promise<Product> {
    return this.cacheService.getOrSet(
      `product:${id}`,
      () => this.productRepository.findById(id),
      3600, // 1 hour TTL
    );
  }

  async update(id: string, data: UpdateProductDto): Promise<Product> {
    const product = await this.productRepository.update(id, data);
    
    // Invalidate cache
    await this.cacheService.delete(`product:${id}`);
    await this.cacheService.deletePattern('product:list:*');
    
    return product;
  }

  async findAll(filters: ProductFilter): Promise<PaginatedResult<Product>> {
    const cacheKey = `product:list:${JSON.stringify(filters)}`;
    
    return this.cacheService.getOrSet(
      cacheKey,
      () => this.productRepository.findWithFilters(filters),
      900, // 15 minutes TTL
    );
  }
}
```

### 11.3 Cache Invalidation Strategies

```typescript
// Cache Invalidation Patterns

// 1. Cache-Aside (Lazy Loading)
async getProduct(id: string): Promise<Product> {
  const cached = await cache.get(`product:${id}`);
  if (cached) return cached;
  
  const product = await db.findById(id);
  await cache.set(`product:${id}`, product, 3600);
  return product;
}

// 2. Write-Through
async updateProduct(id: string, data: UpdateDto): Promise<Product> {
  const product = await db.update(id, data);
  await cache.set(`product:${id}`, product, 3600);
  return product;
}

// 3. Write-Behind (Write-Back)
async updateProductAsync(id: string, data: UpdateDto): Promise<void> {
  await cache.set(`product:${id}`, { ...data, pending: true }, 3600);
  await queue.add('sync-product', { id, data });
}

// 4. Event-Driven Invalidation
@OnEvent('product.updated')
async handleProductUpdate(event: ProductUpdatedEvent) {
  await this.cacheService.delete(`product:${event.productId}`);
  await this.cacheService.deletePattern('product:list:*');
}

// 5. Time-Based Expiration
// Set TTL based on data volatility
const TTL_CONFIG = {
  product_catalog: 3600,      // 1 hour
  pricing: 300,               // 5 minutes
  user_session: 1800,         // 30 minutes
  report_data: 900,           // 15 minutes
  configuration: 86400,       // 24 hours
};
```

### 11.4 Cache Key Design

```typescript
// Cache Key Convention
export class CacheKeys {
  // Entity keys
  static product = (id: string) => `product:${id}`;
  static quote = (id: string) => `quote:${id}`;
  static order = (id: string) => `order:${id}`;
  static invoice = (id: string) => `invoice:${id}`;
  
  // List keys
  static productList = (filters: string) => `product:list:${filters}`;
  static quoteByOpportunity = (oppId: string) => `quote:opp:${oppId}`;
  
  // User-specific keys
  static userSession = (userId: string) => `session:${userId}`;
  static userPermissions = (userId: string) => `permissions:${userId}`;
  
  // Aggregation keys
  static revenueByMonth = (month: string) => `revenue:month:${month}`;
  static dashboardMetrics = (userId: string) => `dashboard:${userId}`;
  
  // Rate limiting keys
  static rateLimit = (userId: string, endpoint: string) => 
    `rate:${userId}:${endpoint}`;
}

// Cache Key Best Practices:
// 1. Use consistent naming: entity:type:id
// 2. Include version for schema changes: product:v2:123
// 3. Use separators: colon (:) or pipe (|)
// 4. Keep keys short but descriptive
// 5. Include tenant ID for multi-tenant: tenant:123:product:456
```

---

## 12. Event Design

### 12.1 Event Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT DRIVEN ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │   Producer   │───▶│    Kafka     │───▶│   Consumer   │         │
│  │  (Service A) │    │   Cluster    │    │  (Service B) │         │
│  ──────────────┘    └──────────────┘    └──────────────┘         │
│         │                    │                    │                │
│         │                    │                    │                │
│         ▼                    ▼                    ▼                │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │              EVENT SCHEMA (Avro/JSON)                     │    │
│  │                                                           │    │
│  │  {                                                        │    │
│  │    "event_id": "uuid",                                   │    │
│  │    "event_type": "QuoteCreated",                         │    │
│  │    "event_version": "1.0",                               │    │
│  │    "timestamp": "2026-06-22T10:30:00Z",                  │    │
│  │    "correlation_id": "abc-123",                          │    │
│  │    "source": "cpq-service",                              │    │
│  │    "tenant_id": "tenant-789",                            │    │
│  │    "payload": { ... }                                    │    │
│  │  }                                                        │    │
│  │                                                           │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.2 Event Schema Definition

```typescript
// Event Base Interface
export interface DomainEvent {
  eventId: string;
  eventType: string;
  eventVersion: string;
  timestamp: string;
  correlationId: string;
  source: string;
  tenantId: string;
  payload: any;
  metadata?: {
    userId?: string;
    ipAddress?: string;
    userAgent?: string;
  };
}

// Specific Events
export class QuoteCreatedEvent implements DomainEvent {
  eventId = uuidv4();
  eventType = 'QuoteCreated';
  eventVersion = '1.0';
  timestamp = new Date().toISOString();
  correlationId: string;
  source = 'cpq-service';
  tenantId: string;
  
  constructor(
    public readonly payload: {
      quoteId: string;
      opportunityId: string;
      name: string;
      totalAmount: number;
      currency: string;
      createdBy: string;
    },
    correlationId: string,
    tenantId: string,
  ) {
    this.correlationId = correlationId;
    this.tenantId = tenantId;
  }
}

export class OrderActivatedEvent implements DomainEvent {
  eventId = uuidv4();
  eventType = 'OrderActivated';
  eventVersion = '1.0';
  timestamp = new Date().toISOString();
  correlationId: string;
  source = 'order-service';
  tenantId: string;
  
  constructor(
    public readonly payload: {
      orderId: string;
      accountId: string;
      activationDate: string;
      totalAmount: number;
      items: Array<{
        productId: string;
        quantity: number;
        unitPrice: number;
      }>;
    },
    correlationId: string,
    tenantId: string,
  ) {
    this.correlationId = correlationId;
    this.tenantId = tenantId;
  }
}

export class InvoiceGeneratedEvent implements DomainEvent {
  eventId = uuidv4();
  eventType = 'InvoiceGenerated';
  eventVersion = '1.0';
  timestamp = new Date().toISOString();
  correlationId: string;
  source = 'billing-service';
  tenantId: string;
  
  constructor(
    public readonly payload: {
      invoiceId: string;
      invoiceNumber: string;
      accountId: string;
      amount: number;
      currency: string;
      dueDate: string;
      lineItems: Array<{
        description: string;
        quantity: number;
        unitPrice: number;
        total: number;
      }>;
    },
    correlationId: string,
    tenantId: string,
  ) {
    this.correlationId = correlationId;
    this.tenantId = tenantId;
  }
}
```

### 12.3 Kafka Configuration

```typescript
// Kafka Producer Service
@Injectable()
export class KafkaProducerService implements OnModuleInit, OnModuleDestroy {
  private producer: Producer;

  constructor(
    @Inject(KAFKA_INSTANCE) private readonly kafka: Kafka,
    private readonly logger: LoggerService,
  ) {}

  async onModuleInit() {
    this.producer = this.kafka.producer();
    await this.producer.connect();
    this.logger.log('Kafka producer connected');
  }

  async onModuleDestroy() {
    await this.producer.disconnect();
  }

  async publish<T>(topic: string, event: T): Promise<void> {
    try {
      await this.producer.send({
        topic,
        messages: [
          {
            key: this.getEventKey(event),
            value: JSON.stringify(event),
            headers: {
              'event-type': (event as any).eventType,
              'correlation-id': (event as any).correlationId,
              'timestamp': (event as any).timestamp,
            },
          },
        ],
      });
      
      this.logger.debug(`Event published to ${topic}`, {
        eventType: (event as any).eventType,
        eventId: (event as any).eventId,
      });
    } catch (error) {
      this.logger.error(`Failed to publish event to ${topic}`, error);
      throw new EventPublishException(error);
    }
  }

  private getEventKey(event: any): string {
    // Use tenant ID or entity ID for partitioning
    return event.tenantId || event.payload?.accountId || 'default';
  }
}

// Kafka Consumer Service
@Injectable()
export class KafkaConsumerService implements OnModuleInit, OnModuleDestroy {
  private consumer: Consumer;

  constructor(
    @Inject(KAFKA_INSTANCE) private readonly kafka: Kafka,
    private readonly logger: LoggerService,
  ) {}

  async onModuleInit() {
    this.consumer = this.kafka.consumer({ 
      groupId: 'revenue-management-group' 
    });
    await this.consumer.connect();
    
    // Subscribe to topics
    await this.consumer.subscribe({ 
      topics: ['quote-events', 'order-events', 'billing-events'],
      fromBeginning: false,
    });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const event = JSON.parse(message.value.toString());
        await this.handleEvent(topic, event);
      },
    });
    
    this.logger.log('Kafka consumer started');
  }

  async onModuleDestroy() {
    await this.consumer.disconnect();
  }

  private async handleEvent(topic: string, event: DomainEvent): Promise<void> {
    const startTime = Date.now();
    
    try {
      this.logger.info(`Processing event ${event.eventType}`, {
        topic,
        eventId: event.eventId,
        correlationId: event.correlationId,
      });

      switch (event.eventType) {
        case 'QuoteCreated':
          await this.handleQuoteCreated(event);
          break;
        case 'OrderActivated':
          await this.handleOrderActivated(event);
          break;
        case 'InvoiceGenerated':
          await this.handleInvoiceGenerated(event);
          break;
        default:
          this.logger.warn(`Unknown event type: ${event.eventType}`);
      }

      const duration = Date.now() - startTime;
      this.logger.info(`Event processed successfully`, {
        eventType: event.eventType,
        duration_ms: duration,
      });
    } catch (error) {
      this.logger.error(`Failed to process event`, {
        eventType: event.eventType,
        eventId: event.eventId,
        error: error.message,
      });
      throw error; // Will trigger retry
    }
  }

  private async handleQuoteCreated(event: DomainEvent): Promise<void> {
    // Handle quote created event
  }

  private async handleOrderActivated(event: DomainEvent): Promise<void> {
    // Handle order activated event
  }

  private async handleInvoiceGenerated(event: DomainEvent): Promise<void> {
    // Handle invoice generated event
  }
}
```

### 12.4 Event Topics Design

```yaml
# Kafka Topics Configuration
topics:
  # Core Business Events
  quote-events:
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000  # 7 days
    cleanup_policy: delete
    events:
      - QuoteCreated
      - QuoteUpdated
      - QuoteSubmitted
      - QuoteApproved
      - QuoteRejected
  
  order-events:
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000
    events:
      - OrderCreated
      - OrderActivated
      - OrderDeactivated
      - OrderCompleted
  
  billing-events:
    partitions: 24
    replication_factor: 3
    retention_ms: 2592000000  # 30 days
    events:
      - InvoiceGenerated
      - InvoiceSent
      - InvoicePaid
      - InvoiceOverdue
      - PaymentReceived
      - PaymentFailed
  
  contract-events:
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000
    events:
      - ContractCreated
      - ContractSigned
      - ContractExpired
      - ContractRenewed
  
  usage-events:
    partitions: 48
    replication_factor: 3
    retention_ms: 86400000  # 1 day
    compression_type: lz4
    events:
      - UsageRecorded
      - UsageAggregated
      - RatingCompleted
  
  notification-events:
    partitions: 6
    replication_factor: 3
    retention_ms: 86400000
    events:
      - EmailSent
      - SmsSent
      - PushNotificationSent

# Consumer Groups
consumer_groups:
  billing-consumer:
    topics: [order-events, contract-events]
    concurrency: 10
  
  notification-consumer:
    topics: [quote-events, billing-events, contract-events]
    concurrency: 5
  
  analytics-consumer:
    topics: [quote-events, order-events, billing-events, usage-events]
    concurrency: 20
  
  audit-consumer:
    topics: [quote-events, order-events, billing-events, contract-events]
    concurrency: 10
```

### 12.5 Event Sourcing Pattern

```typescript
// Event Store
@Injectable()
export class EventStoreService {
  constructor(
    private readonly eventRepository: EventRepository,
    private readonly kafkaProducer: KafkaProducerService,
  ) {}

  async appendEvents(
    aggregateId: string,
    aggregateType: string,
    events: DomainEvent[],
  ): Promise<void> {
    // Get current version
    const currentVersion = await this.eventRepository.getLatestVersion(
      aggregateId,
    );

    // Save events with versioning
    for (let i = 0; i < events.length; i++) {
      const event = events[i];
      await this.eventRepository.save({
        ...event,
        aggregateId,
        aggregateType,
        version: currentVersion + i + 1,
      });
    }

    // Publish to Kafka
    for (const event of events) {
      await this.kafkaProducer.publish(
        `${aggregateType.toLowerCase()}-events`,
        event,
      );
    }
  }

  async getAggregateEvents(
    aggregateId: string,
    fromVersion?: number,
  ): Promise<DomainEvent[]> {
    return this.eventRepository.findByAggregateId(
      aggregateId,
      fromVersion,
    );
  }

  async rebuildAggregate<T>(
    aggregateId: string,
    factory: (events: DomainEvent[]) => T,
  ): Promise<T> {
    const events = await this.getAggregateEvents(aggregateId);
    return factory(events);
  }
}

// Aggregate Root with Event Sourcing
export class QuoteAggregate {
  private quote: Quote;
  private uncommittedEvents: DomainEvent[] = [];

  constructor(private readonly eventStore: EventStoreService) {}

  static async load(
    id: string,
    eventStore: EventStoreService,
  ): Promise<QuoteAggregate> {
    const aggregate = new QuoteAggregate(eventStore);
    const events = await eventStore.getAggregateEvents(id);
    
    for (const event of events) {
      aggregate.applyEvent(event);
    }
    
    return aggregate;
  }

  createQuote(data: CreateQuoteData): void {
    const event = new QuoteCreatedEvent(data, uuidv4(), data.tenantId);
    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  submitForApproval(userId: string): void {
    if (this.quote.status !== QuoteStatus.DRAFT) {
      throw new BusinessException('Quote must be in DRAFT status');
    }
    
    const event = new QuoteSubmittedEvent(
      { quoteId: this.quote.id, submittedBy: userId },
      uuidv4(),
      this.quote.tenantId,
    );
    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  async save(): Promise<void> {
    if (this.uncommittedEvents.length === 0) return;
    
    await this.eventStore.appendEvents(
      this.quote.id,
      'Quote',
      this.uncommittedEvents,
    );
    
    this.uncommittedEvents = [];
  }

  private applyEvent(event: DomainEvent): void {
    switch (event.eventType) {
      case 'QuoteCreated':
        this.quote = Quote.fromEvent(event);
        break;
      case 'QuoteSubmitted':
        this.quote.status = QuoteStatus.PENDING_APPROVAL;
        break;
      case 'QuoteApproved':
        this.quote.status = QuoteStatus.APPROVED;
        break;
      // ... other events
    }
  }
}
```

### 12.6 Event Retry & Dead Letter Queue

```typescript
// Dead Letter Queue Handler
@Injectable()
export class DeadLetterQueueService {
  constructor(
    private readonly kafkaProducer: KafkaProducerService,
    private readonly logger: LoggerService,
    private readonly eventRepository: EventRepository,
  ) {}

  async moveToDLQ(
    originalTopic: string,
    event: DomainEvent,
    error: Error,
    retryCount: number,
  ): Promise<void> {
    const dlqEvent = {
      ...event,
      metadata: {
        ...event.metadata,
        originalTopic,
        error: error.message,
        errorStack: error.stack,
        retryCount,
        failedAt: new Date().toISOString(),
      },
    };

    await this.kafkaProducer.publish('dead-letter-queue', dlqEvent);
    await this.eventRepository.saveFailedEvent(dlqEvent);

    this.logger.error('Event moved to DLQ', {
      eventType: event.eventType,
      eventId: event.eventId,
      error: error.message,
      retryCount,
    });
  }

  async retryFromDLQ(eventId: string): Promise<void> {
    const failedEvent = await this.eventRepository.findFailedEvent(eventId);
    if (!failedEvent) {
      throw new NotFoundException('FailedEvent', eventId);
    }

    // Remove metadata added by DLQ
    const { originalTopic, ...originalEvent } = failedEvent.metadata;
    const event = { ...failedEvent, metadata: originalEvent };

    await this.kafkaProducer.publish(originalTopic, event);
    await this.eventRepository.markEventAsRetried(eventId);

    this.logger.info('Event retried from DLQ', {
      eventId,
      originalTopic,
    });
  }
}

// Retry Policy
export class RetryPolicy {
  static readonly MAX_RETRIES = 3;
  static readonly INITIAL_DELAY_MS = 1000;
  static readonly MAX_DELAY_MS = 30000;

  static async executeWithRetry<T>(
    operation: () => Promise<T>,
    context: string,
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= RetryPolicy.MAX_RETRIES; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt < RetryPolicy.MAX_RETRIES) {
          const delay = Math.min(
            RetryPolicy.INITIAL_DELAY_MS * Math.pow(2, attempt - 1),
            RetryPolicy.MAX_DELAY_MS,
          );
          
          console.warn(
            `${context} failed (attempt ${attempt}/${RetryPolicy.MAX_RETRIES}), ` +
            `retrying in ${delay}ms`,
          );
          
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError;
  }
}
```

---

## ภาคผนวก

### ภาคผนวก A: Technology Decision Records

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Frontend Framework | React + TypeScript | Large ecosystem, TypeScript support, component reusability |
| Backend Framework | NestJS | Modular architecture, TypeScript-first, enterprise-ready |
| High-Performance Services | Go | Concurrency, performance, low memory footprint |
| AI/ML Services | Python | Rich ML ecosystem, FastAPI, data processing libraries |
| Cache | Redis | In-memory, data structures, pub/sub, clustering |
| Event Streaming | Kafka | High throughput, durability, event sourcing support |
| Database | PostgreSQL | ACID compliance, JSON support, extensions, reliability |

### ภาคผนวก B: Development Environment Setup

```bash
# Prerequisites
- Node.js 18+
- Go 1.21+
- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Kafka 3.5+
- Docker & Docker Compose

# Quick Start
git clone <repository>
cd revenue-management

# Start infrastructure
docker-compose up -d postgres redis kafka

# Install dependencies
cd frontend && npm install
cd ../api && npm install
cd ../services/billing && go mod download
cd ../services/ai && pip install -r requirements.txt

# Run migrations
cd ../api && npm run migration:run

# Start services
npm run start:dev  # API
go run cmd/billing-service/main.go  # Billing
python src/ai_service/main.py  # AI Service
npm run start  # Frontend
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการพัฒนาระบบ Revenue Management ระดับ Developer**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review
