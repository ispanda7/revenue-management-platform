# Database Design

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**เอกสารที่เกี่ยวข้อง:** `05-Kubernetes-Deployment-Design.md`, `03-Software-Design-Specification.md`  
**กลุ่มเป้าหมาย:** Database Administrators, Backend Developers, Data Architects

---

## 📋 สารบัญ

1. [Database Selection](#1-database-selection)
2. [ER Diagram](#2-er-diagram)
3. [Domain Model](#3-domain-model)
4. [Tables](#4-tables)
5. [Indexes](#5-indexes)
6. [Constraints](#6-constraints)
7. [Partition Strategy](#7-partition-strategy)
8. [Replication Strategy](#8-replication-strategy)
9. [Backup Strategy](#9-backup-strategy)
10. [Data Retention](#10-data-retention)
11. [Security](#11-security)
12. [RLS Policy](#12-rls-policy)

---

## 1. Database Selection

### 1.1 Multi-Database Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATABASE SELECTION MATRIX                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PostgreSQL (Primary - Relational Data)                          │
│  ├─ Use Case: Transactional data, financial records              │
│  ├─ Data: Opportunities, Quotes, Orders, Invoices, Contracts    │
│  ├─ Why: ACID compliance, strong consistency, JSON support      │
│  ├─ Version: PostgreSQL 15.4 (Aurora)                           │
│  └─ Scale: Vertical + Read Replicas                             │
│                                                                  │
│  MongoDB (Document Store - Flexible Schema)                      │
│  ├─ Use Case: Product catalog, usage records, audit logs        │
│  ├─ Data: Product configurations, usage events, document store  │
│  ├─ Why: Schema flexibility, horizontal scaling, JSON native    │
│  ├─ Version: MongoDB 7.0 (Atlas)                                │
│  └─ Scale: Sharding + Replica Sets                              │
│                                                                  │
│  Redis (In-Memory Cache)                                         │
│  ├─ Use Case: Session cache, rate limiting, real-time data      │
│  ├─ Data: User sessions, pricing cache, rate limit counters     │
│  ├─ Why: Sub-millisecond latency, data structures, pub/sub      │
│  ├─ Version: Redis 7.2 (ElastiCache)                            │
│  └─ Scale: Cluster mode, read replicas                          │
│                                                                  │
│  SQL Server (Optional - Enterprise Integration)                  │
│  ├─ Use Case: Legacy system integration, reporting              │
│  ├─ Data: Synced data from legacy ERP systems                   │
│  ├─ Why: Enterprise features, SSIS, SSRS integration            │
│  ├─ Version: SQL Server 2022                                    │
│  └─ Scale: Always On Availability Groups                        │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 1.2 Database Selection Criteria

| Criteria | PostgreSQL | MongoDB | Redis | SQL Server |
|----------|-----------|---------|-------|------------|
| **ACID Compliance** | ✅ Full | ✅ (4.0+) | ❌ (Best effort) | ✅ Full |
| **Schema Flexibility** | ️ JSONB | ✅ Native | ❌ | ⚠️ JSON |
| **Horizontal Scaling** | ⚠️ Citus | ✅ Sharding | ✅ Cluster | ⚠️ Limited |
| **Query Complexity** | ✅ Complex | ⚠️ Moderate |  Simple | ✅ Complex |
| **Transaction Support** | ✅ Full | ✅ Multi-doc |  | ✅ Full |
| **Full-Text Search** | ✅ Built-in | ✅ Atlas Search | ❌ | ✅ Full |
| **Geospatial** | ✅ PostGIS | ✅ 2dsphere |  | ✅ |
| **Cost** |  Low | 💰💰 Medium | 💰 Low | 💰💰💰 High |
| **Community** | ✅ Large | ✅ Large | ✅ Large | ⚠️ Vendor |

### 1.3 Data Distribution Strategy

```yaml
# PostgreSQL Tables (Transactional)
- opportunities
- quotes
- quote_line_items
- orders
- order_items
- invoices
- invoice_line_items
- contracts
- assets
- billing_schedules
- payments
- customers
- products
- price_books

# MongoDB Collections (Flexible/High-Volume)
- product_catalog (flexible attributes)
- usage_records (high volume, time-series)
- audit_logs (append-only, flexible schema)
- document_store (contracts, proposals)
- feature_flags (dynamic configuration)
- user_preferences (flexible structure)

# Redis Keys (Cache/Real-time)
- session:{user_id} (user sessions)
- pricing:{product_id} (price cache)
- rate_limit:{ip}:{endpoint} (rate limiting)
- quote:{id}:lock (distributed locks)
- dashboard:{user_id}:metrics (real-time metrics)
- pubsub:invoice-events (event streaming)

# SQL Server (Legacy Integration)
- legacy_customers (synced from old system)
- legacy_products (ERP integration)
- historical_data (archived data)
```

---

## 2. ER Diagram

### 2.1 Core Entities Relationship

```
─────────────────────────────────────────────────────────────────┐
│                    CORE ENTITIES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ──────────────┐     1:N     ┌──────────────┐                 │
│  │   Account    │────────────▶│ Opportunity  │                 │
│  │              │             │              │                 │
│  │ id (PK)      │             │ id (PK)      │                 │
│  │ name         │             │ account_id   │                 │
│  │ industry     │             │ name         │                 │
│  │ type         │             │ stage        │                 │
│  │ owner_id     │             │ amount       │                 │
│  └──────────────┘             │ close_date   │                 │
│         │                     │ probability  │                 │
│         │                     └──────────────┘                 │
│         │ 1:N                          │ 1:N                   │
│         ▼                              ▼                       │
│  ┌──────────────┐             ┌──────────────┐                 │
│  │   Contact    │             │    Quote     │                 │
│  │              │             │              │                 │
│  │ id (PK)      │             │ id (PK)      │                 │
│  │ account_id   │             │ opportunity  │                 │
│  │ name         │             │ name         │                 │
│  │ email        │             │ status       │                 │
│  │ phone        │             │ total_amount │                 │
│  │ role         │             │ expiration   │                 │
│  └──────────────┘             └──────────────                 │
│                                          │ 1:N                  │
│                                          ▼                      │
│                                 ┌──────────────┐               │
│                                 │ QuoteLineItem│               │
│                                 │              │               │
│                                 │ id (PK)      │               │
│                                 │ quote_id     │               │
│                                 │ product_id   │               │
│                                 │ quantity     │               │
│                                 │ unit_price   │               │
│                                 │ discount     │               │
│                                 │ total_price  │               │
│                                 └──────────────┘               │
│                                          │                      │
│                                          │ Approved             │
│                                          ▼                      │
│                                 ┌──────────────               │
│                                 │    Order     │               │
│                                 │              │               │
│                                 │ id (PK)      │               │
│                                 │ quote_id     │               │
│                                 │ account_id   │               │
│                                 │ status       │               │
│                                 │ total_amount │               │
│                                 │ order_date   │               │
│                                 └──────────────┘               │
│                                          │ 1:N                  │
│                                          ▼                      │
│                                 ┌──────────────┐               │
│                                 │  OrderItem   │               │
│                                 │              │               │
│                                 │ id (PK)      │               │
│                                 │ order_id     │               │
│                                 │ product_id   │               │
│                                 │ quantity     │               │
│                                 │ unit_price   │               │
│                                 └──────────────┘               │
│                                          │                      │
│                              ┌─────────────┼─────────────┐     │
│                              │             │             │     │
│                              ▼             ▼             ▼     │
│                     ┌──────────────┐ ┌──────────┐ ┌──────────┐│
│                     │   Invoice    │ │  Asset   │ │ Contract ││
│                     │              │ │          │ │          ││
│                     │ id (PK)      │ │ id (PK)  │ │ id (PK)  ││
│                     │ order_id     │ │ order_id │ │ order_id ││
│                     │ invoice_num  │ │ product  │ │ terms    ││
│                     │ status       │ │ status   │ │ status   ││
│                     │ total_amount │ │ lifespan │ │ signed   ││
│                     │ due_date     │ └──────────┘ └──────────┘│
│                     └──────────────┘                           │
│                              │ 1:N                             │
│                              ▼                                 │
│                     ┌──────────────┐                           │
│                     │InvoiceLine   │                           │
│                     │              │                           │
│                     │ id (PK)      │                           │
│                     │ invoice_id   │                           │
│                     │ description  │                           │
│                     │ quantity     │                           │
│                     │ unit_price   │                           │
│                     │ total        │                           │
│                     └──────────────┘                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Supporting Entities

```
┌─────────────────────────────────────────────────────────────────┐
│                  SUPPORTING ENTITIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     1:N     ┌──────────────┐                 │
│  │   Product    │────────────▶│  PriceBook   │                 │
│  │              │             │              │                 │
│  │ id (PK)      │             │ id (PK)      │                 │
│  │ name         │             │ product_id   │                 │
│  │ sku          │             │ pricebook_id │                 │
│  │ category     │             │ unit_price   │                 │
│  │ description  │             │ currency     │                 │
│  │ is_active    │             │ effective    │                 │
│  └──────────────┘             └──────────────┘                 │
│                                                                  │
│  ┌──────────────┐     1:N     ┌──────────────┐                 │
│  │  Customer    │────────────▶│ Subscription │                 │
│  │              │             │              │                 │
│  │ id (PK)      │             │ id (PK)      │                 │
│  │ account_id   │             │ customer_id  │                 │
│  │ name         │             │ product_id   │                 │
│  │ email        │             │ status       │                 │
│  │ tier         │             │ start_date   │                 │
│  │ credit_limit │             │ end_date     │                 │
│  └──────────────┘             │ billing_freq │                 │
│                               └──────────────┘                 │
│                                          │ 1:N                  │
│                                          ▼                      │
│                                 ┌──────────────┐               │
│                                 │ UsageRecord  │               │
│                                 │ (MongoDB)    │               │
│                                 │              │               │
│                                 │ id (PK)      │               │
│                                 │ subscription │               │
│                                 │ timestamp    │               │
│                                 │ quantity     │               │
│                                 │ unit         │               │
│                                 │ metadata     │               │
│                                 └──────────────┘               │
│                                                                  │
│  ──────────────┐     1:N     ┌──────────────┐                 │
│  │    User      │────────────▶│    Audit     │                 │
│  │              │             │    Log       │                 │
│  │ id (PK)      │             │ (MongoDB)    │                 │
│  │ username     │             │              │                 │
│  │ email        │             │ id (PK)      │                 │
│  │ role         │             │ user_id      │                 │
│  │ tenant_id    │             │ action       │                 │
│  │ is_active    │             │ entity_type  │                 │
│  └──────────────┘             │ entity_id    │                 │
│                               │ timestamp    │                 │
│                               │ changes      │                 │
│                               ──────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Domain Model

### 3.1 Domain-Driven Design (DDD) Approach

```typescript
// Domain Entities (TypeScript Example)

// Aggregate Root: Opportunity
class Opportunity {
  private constructor(
    public readonly id: string,
    public readonly accountId: string,
    public name: string,
    public stage: OpportunityStage,
    public amount: Money,
    public closeDate: Date,
    public probability: number,
    public ownerId: string,
    private quotes: Quote[],
    private createdAt: Date,
    private updatedAt: Date,
  ) {}

  // Factory Method
  static create(props: CreateOpportunityProps): Opportunity {
    // Validation
    if (props.amount.amount <= 0) {
      throw new DomainError('Amount must be positive');
    }
    if (props.probability < 0 || props.probability > 100) {
      throw new DomainError('Probability must be between 0 and 100');
    }

    return new Opportunity(
      uuidv4(),
      props.accountId,
      props.name,
      OpportunityStage.QUALIFICATION,
      props.amount,
      props.closeDate,
      props.probability,
      props.ownerId,
      [],
      new Date(),
      new Date(),
    );
  }

  // Domain Methods
  addQuote(quote: Quote): void {
    if (this.stage === OpportunityStage.CLOSED_WON) {
      throw new DomainError('Cannot add quote to closed opportunity');
    }
    this.quotes.push(quote);
    this.updatedAt = new Date();
  }

  advanceStage(newStage: OpportunityStage): void {
    if (!this.canTransitionTo(newStage)) {
      throw new DomainError(`Cannot transition to ${newStage}`);
    }
    this.stage = newStage;
    this.updatedAt = new Date();
  }

  private canTransitionTo(stage: OpportunityStage): boolean {
    const transitions: Record<OpportunityStage, OpportunityStage[]> = {
      [OpportunityStage.QUALIFICATION]: [OpportunityStage.DISCOVERY],
      [OpportunityStage.DISCOVERY]: [OpportunityStage.PROPOSAL],
      [OpportunityStage.PROPOSAL]: [OpportunityStage.NEGOTIATION],
      [OpportunityStage.NEGOTIATION]: [
        OpportunityStage.CLOSED_WON,
        OpportunityStage.CLOSED_LOST,
      ],
      [OpportunityStage.CLOSED_WON]: [],
      [OpportunityStage.CLOSED_LOST]: [],
    };
    return transitions[this.stage].includes(stage);
  }
}

// Value Object: Money
class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string,
  ) {
    if (amount < 0) {
      throw new DomainError('Amount cannot be negative');
    }
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new DomainError('Cannot add money with different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }
}

// Entity: Quote
class Quote {
  constructor(
    public readonly id: string,
    public readonly opportunityId: string,
    public name: string,
    public status: QuoteStatus,
    public lineItems: QuoteLineItem[],
    public totalAmount: Money,
    public expirationDate: Date,
    public createdBy: string,
    public createdAt: Date,
    public updatedAt: Date,
  ) {}

  // Domain Methods
  addLineItem(item: QuoteLineItem): void {
    if (this.status !== QuoteStatus.DRAFT) {
      throw new DomainError('Can only add items to draft quotes');
    }
    this.lineItems.push(item);
    this.recalculateTotal();
    this.updatedAt = new Date();
  }

  submitForApproval(): void {
    if (this.lineItems.length === 0) {
      throw new DomainError('Quote must have at least one line item');
    }
    this.status = QuoteStatus.PENDING_APPROVAL;
    this.updatedAt = new Date();
  }

  approve(approverId: string): void {
    if (this.status !== QuoteStatus.PENDING_APPROVAL) {
      throw new DomainError('Quote must be pending approval');
    }
    this.status = QuoteStatus.APPROVED;
    this.updatedAt = new Date();
  }

  private recalculateTotal(): void {
    this.totalAmount = this.lineItems.reduce(
      (sum, item) => sum.add(item.totalPrice),
      new Money(0, this.lineItems[0]?.unitPrice.currency || 'USD'),
    );
  }
}
```

### 3.2 Domain Events

```typescript
// Domain Events
class OpportunityCreated {
  constructor(
    public readonly opportunityId: string,
    public readonly accountId: string,
    public readonly amount: Money,
    public readonly timestamp: Date,
  ) {}
}

class QuoteSubmitted {
  constructor(
    public readonly quoteId: string,
    public readonly opportunityId: string,
    public readonly totalAmount: Money,
    public readonly submittedBy: string,
    public readonly timestamp: Date,
  ) {}
}

class OrderActivated {
  constructor(
    public readonly orderId: string,
    public readonly accountId: string,
    public readonly activationDate: Date,
    public readonly timestamp: Date,
  ) {}
}

class InvoiceGenerated {
  constructor(
    public readonly invoiceId: string,
    public readonly orderId: string,
    public readonly amount: Money,
    public readonly dueDate: Date,
    public readonly timestamp: Date,
  ) {}
}
```

---

## 4. Tables

### 4.1 PostgreSQL Schema Design

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "citext";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Schema for Revenue Management
CREATE SCHEMA IF NOT EXISTS revenue_mgmt;
SET search_path TO revenue_mgmt;

-- ============================================================
-- ACCOUNTS TABLE
-- ============================================================
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    industry VARCHAR(100),
    type VARCHAR(50) CHECK (type IN ('customer', 'prospect', 'partner')),
    owner_id UUID NOT NULL,
    credit_limit DECIMAL(15,2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    is_active BOOLEAN DEFAULT true,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_credit_limit CHECK (credit_limit >= 0),
    CONSTRAINT chk_currency CHECK (currency IN ('USD', 'THB', 'EUR', 'GBP', 'SGD'))
);

COMMENT ON TABLE accounts IS 'Customer and prospect accounts';
COMMENT ON COLUMN accounts.metadata IS 'Flexible metadata in JSON format';

-- ============================================================
-- OPPORTUNITIES TABLE
-- ============================================================
CREATE TABLE opportunities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    stage VARCHAR(50) NOT NULL DEFAULT 'qualification'
        CHECK (stage IN ('qualification', 'discovery', 'proposal', 'negotiation', 'closed_won', 'closed_lost')),
    amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    close_date DATE NOT NULL,
    probability INTEGER CHECK (probability BETWEEN 0 AND 100),
    owner_id UUID NOT NULL,
    description TEXT,
    is_deleted BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_amount CHECK (amount >= 0),
    CONSTRAINT chk_probability CHECK (probability BETWEEN 0 AND 100)
);

COMMENT ON TABLE opportunities IS 'Sales opportunities';
COMMENT ON COLUMN opportunities.stage IS 'Current stage in sales pipeline';

-- ============================================================
-- QUOTES TABLE
-- ============================================================
CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    opportunity_id UUID NOT NULL REFERENCES opportunities(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'pending_approval', 'approved', 'rejected', 'expired', 'converted')),
    total_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    expiration_date DATE NOT NULL,
    created_by UUID NOT NULL,
    approved_by UUID,
    approved_at TIMESTAMP WITH TIME ZONE,
    rejection_reason TEXT,
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_total_amount CHECK (total_amount >= 0)
);

COMMENT ON TABLE quotes IS 'Price quotes for opportunities';

-- ============================================================
-- QUOTE LINE ITEMS TABLE
-- ============================================================
CREATE TABLE quote_line_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    quote_id UUID NOT NULL REFERENCES quotes(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity DECIMAL(10,2) NOT NULL DEFAULT 1,
    unit_price DECIMAL(15,2) NOT NULL,
    discount_percent DECIMAL(5,2) DEFAULT 0,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    tax_percent DECIMAL(5,2) DEFAULT 0,
    total_price DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    configuration JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_quantity CHECK (quantity > 0),
    CONSTRAINT chk_unit_price CHECK (unit_price >= 0),
    CONSTRAINT chk_discount_percent CHECK (discount_percent BETWEEN 0 AND 100),
    CONSTRAINT chk_discount_amount CHECK (discount_amount >= 0),
    CONSTRAINT chk_tax_percent CHECK (tax_percent BETWEEN 0 AND 100),
    CONSTRAINT chk_total_price CHECK (total_price >= 0)
);

COMMENT ON TABLE quote_line_items IS 'Line items in quotes';
COMMENT ON COLUMN quote_line_items.configuration IS 'Product configuration in JSON format';

-- ============================================================
-- ORDERS TABLE
-- ============================================================
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    quote_id UUID REFERENCES quotes(id) ON DELETE SET NULL,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'activated', 'fulfilled', 'cancelled', 'expired')),
    total_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    order_date DATE NOT NULL DEFAULT CURRENT_DATE,
    activation_date TIMESTAMP WITH TIME ZONE,
    fulfillment_date TIMESTAMP WITH TIME ZONE,
    cancellation_reason TEXT,
    notes TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_order_total CHECK (total_amount >= 0)
);

COMMENT ON TABLE orders IS 'Customer orders';
COMMENT ON COLUMN orders.order_number IS 'Human-readable order number';

-- ============================================================
-- ORDER ITEMS TABLE
-- ============================================================
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity DECIMAL(10,2) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    total_price DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    billing_type VARCHAR(50) DEFAULT 'one_time'
        CHECK (billing_type IN ('one_time', 'recurring', 'usage_based')),
    billing_frequency VARCHAR(50)
        CHECK (billing_frequency IN ('monthly', 'quarterly', 'annual')),
    subscription_start_date DATE,
    subscription_end_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_order_item_quantity CHECK (quantity > 0),
    CONSTRAINT chk_order_item_unit_price CHECK (unit_price >= 0),
    CONSTRAINT chk_order_item_total CHECK (total_price >= 0)
);

COMMENT ON TABLE order_items IS 'Line items in orders';

-- ============================================================
-- INVOICES TABLE
-- ============================================================
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID REFERENCES orders(id) ON DELETE SET NULL,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    invoice_number VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'issued', 'sent', 'paid', 'overdue', 'cancelled', 'void')),
    subtotal DECIMAL(15,2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    total_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    amount_paid DECIMAL(15,2) DEFAULT 0,
    amount_due DECIMAL(15,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    issue_date DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date DATE NOT NULL,
    paid_date TIMESTAMP WITH TIME ZONE,
    notes TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_invoice_subtotal CHECK (subtotal >= 0),
    CONSTRAINT chk_invoice_tax CHECK (tax_amount >= 0),
    CONSTRAINT chk_invoice_discount CHECK (discount_amount >= 0),
    CONSTRAINT chk_invoice_total CHECK (total_amount >= 0),
    CONSTRAINT chk_invoice_paid CHECK (amount_paid >= 0),
    CONSTRAINT chk_invoice_due CHECK (amount_due >= 0)
);

COMMENT ON TABLE invoices IS 'Customer invoices';

-- ============================================================
-- INVOICE LINE ITEMS TABLE
-- ============================================================
CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    order_item_id UUID REFERENCES order_items(id) ON DELETE SET NULL,
    description TEXT NOT NULL,
    quantity DECIMAL(10,2) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    tax_percent DECIMAL(5,2) DEFAULT 0,
    total DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_inv_line_quantity CHECK (quantity > 0),
    CONSTRAINT chk_inv_line_unit_price CHECK (unit_price >= 0),
    CONSTRAINT chk_inv_line_total CHECK (total >= 0)
);

COMMENT ON TABLE invoice_line_items IS 'Line items in invoices';

-- ============================================================
-- CONTRACTS TABLE
-- ============================================================
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID REFERENCES orders(id) ON DELETE SET NULL,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    contract_number VARCHAR(50) UNIQUE NOT NULL,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'negotiation', 'pending_signature', 'active', 'expired', 'terminated')),
    contract_type VARCHAR(50) NOT NULL
        CHECK (contract_type IN ('sales', 'service', 'partnership', 'nda', 'subscription')),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    auto_renewal BOOLEAN DEFAULT false,
    renewal_notice_days INTEGER DEFAULT 30,
    total_value DECIMAL(15,2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    signed_date TIMESTAMP WITH TIME ZONE,
    signed_by UUID,
    document_url VARCHAR(500),
    terms_and_conditions TEXT,
    notes TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_contract_dates CHECK (end_date > start_date),
    CONSTRAINT chk_contract_value CHECK (total_value >= 0),
    CONSTRAINT chk_renewal_notice CHECK (renewal_notice_days >= 0)
);

COMMENT ON TABLE contracts IS 'Customer contracts';

-- ============================================================
-- ASSETS TABLE
-- ============================================================
CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
    order_item_id UUID REFERENCES order_items(id) ON DELETE SET NULL,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    product_id UUID NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    serial_number VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'suspended', 'expired', 'cancelled')),
    activation_date TIMESTAMP WITH TIME ZONE NOT NULL,
    expiration_date TIMESTAMP WITH TIME ZONE,
    quantity DECIMAL(10,2) DEFAULT 1,
    entitlements JSONB DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_asset_quantity CHECK (quantity > 0)
);

COMMENT ON TABLE assets IS 'Customer assets and entitlements';
COMMENT ON COLUMN assets.entitlements IS 'Entitlement details in JSON format';

-- ============================================================
-- BILLING SCHEDULES TABLE
-- ============================================================
CREATE TABLE billing_schedules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    order_item_id UUID REFERENCES order_items(id) ON DELETE SET NULL,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    billing_type VARCHAR(50) NOT NULL
        CHECK (billing_type IN ('one_time', 'recurring', 'usage_based', 'milestone')),
    billing_frequency VARCHAR(50)
        CHECK (billing_frequency IN ('monthly', 'quarterly', 'annual', 'custom')),
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    start_date DATE NOT NULL,
    end_date DATE,
    next_billing_date DATE NOT NULL,
    billing_day INTEGER CHECK (billing_day BETWEEN 1 AND 31),
    status VARCHAR(50) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'paused', 'completed', 'cancelled')),
    invoice_id UUID REFERENCES invoices(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_billing_amount CHECK (amount >= 0),
    CONSTRAINT chk_billing_day CHECK (billing_day BETWEEN 1 AND 31)
);

COMMENT ON TABLE billing_schedules IS 'Billing schedules for recurring charges';

-- ============================================================
-- PAYMENTS TABLE
-- ============================================================
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    payment_number VARCHAR(50) UNIQUE NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    payment_method VARCHAR(50) NOT NULL
        CHECK (payment_method IN ('credit_card', 'bank_transfer', 'check', 'cash', 'other')),
    payment_date TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) NOT NULL DEFAULT 'completed'
        CHECK (status IN ('pending', 'completed', 'failed', 'refunded', 'cancelled')),
    reference_number VARCHAR(100),
    notes TEXT,
    processed_by UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_payment_amount CHECK (amount > 0)
);

COMMENT ON TABLE payments IS 'Customer payments';

-- ============================================================
-- PRODUCTS TABLE
-- ============================================================
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    subcategory VARCHAR(100),
    product_type VARCHAR(50) NOT NULL
        CHECK (product_type IN ('physical', 'digital', 'service', 'subscription')),
    is_active BOOLEAN DEFAULT true,
    is_taxable BOOLEAN DEFAULT true,
    default_price DECIMAL(15,2),
    default_currency VARCHAR(3) DEFAULT 'USD',
    attributes JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE products IS 'Product catalog';
COMMENT ON COLUMN products.attributes IS 'Product attributes in JSON format';

-- ============================================================
-- PRICE BOOKS TABLE
-- ============================================================
CREATE TABLE price_books (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    currency VARCHAR(3) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    is_standard BOOLEAN DEFAULT false,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_price_book_dates CHECK (expiration_date IS NULL OR expiration_date > effective_date)
);

COMMENT ON TABLE price_books IS 'Price books for different pricing tiers';

-- ============================================================
-- PRICE BOOK ENTRIES TABLE
-- ============================================================
CREATE TABLE price_book_entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    price_book_id UUID NOT NULL REFERENCES price_books(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    unit_price DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    discount_percent DECIMAL(5,2) DEFAULT 0,
    min_quantity DECIMAL(10,2) DEFAULT 1,
    max_quantity DECIMAL(10,2),
    effective_date DATE NOT NULL,
    expiration_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_pbe_unit_price CHECK (unit_price >= 0),
    CONSTRAINT chk_pbe_discount CHECK (discount_percent BETWEEN 0 AND 100),
    CONSTRAINT chk_pbe_quantity CHECK (min_quantity > 0),
    CONSTRAINT chk_pbe_dates CHECK (expiration_date IS NULL OR expiration_date > effective_date),
    UNIQUE(price_book_id, product_id, effective_date)
);

COMMENT ON TABLE price_book_entries IS 'Product prices in price books';

-- ============================================================
-- USERS TABLE
-- ============================================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    role VARCHAR(50) NOT NULL
        CHECK (role IN ('admin', 'sales_rep', 'sales_manager', 'billing_specialist', 'contract_manager', 'viewer')),
    tenant_id UUID NOT NULL,
    is_active BOOLEAN DEFAULT true,
    last_login TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE users IS 'System users';

-- ============================================================
-- AUDIT LOGS TABLE (PostgreSQL for recent logs)
-- ============================================================
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE audit_logs IS 'Recent audit logs (archived to MongoDB)';

-- ============================================================
-- INDEXES (See Section 5 for details)
-- ============================================================
-- Created in Section 5

-- ============================================================
-- TRIGGERS FOR UPDATED_AT
-- ============================================================
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to all tables with updated_at
CREATE TRIGGER update_accounts_updated_at BEFORE UPDATE ON accounts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_opportunities_updated_at BEFORE UPDATE ON opportunities
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_quotes_updated_at BEFORE UPDATE ON quotes
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_orders_updated_at BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_invoices_updated_at BEFORE UPDATE ON invoices
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_contracts_updated_at BEFORE UPDATE ON contracts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_assets_updated_at BEFORE UPDATE ON assets
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_billing_schedules_updated_at BEFORE UPDATE ON billing_schedules
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_payments_updated_at BEFORE UPDATE ON payments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_price_books_updated_at BEFORE UPDATE ON price_books
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_price_book_entries_updated_at BEFORE UPDATE ON price_book_entries
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 4.2 MongoDB Schema Design

```javascript
// MongoDB Collections

// 1. Product Catalog (Flexible Schema)
db.createCollection('product_catalog', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['sku', 'name', 'category', 'createdAt'],
      properties: {
        sku: { bsonType: 'string', description: 'SKU must be a string' },
        name: { bsonType: 'string', description: 'Name must be a string' },
        category: { bsonType: 'string' },
        subcategory: { bsonType: 'string' },
        attributes: { bsonType: 'object' },
        pricing: {
          bsonType: 'object',
          properties: {
            basePrice: { bsonType: 'number' },
            currency: { bsonType: 'string' },
            tiers: {
              bsonType: 'array',
              items: {
                bsonType: 'object',
                properties: {
                  minQuantity: { bsonType: 'number' },
                  maxQuantity: { bsonType: ['number', 'null'] },
                  price: { bsonType: 'number' }
                }
              }
            }
          }
        },
        isActive: { bsonType: 'bool' },
        createdAt: { bsonType: 'date' },
        updatedAt: { bsonType: 'date' }
      }
    }
  }
});

// 2. Usage Records (Time-Series Collection)
db.createCollection('usage_records', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'metadata',
    granularity: 'hours'
  },
  expireAfterSeconds: 2592000 // 30 days retention
});

// Example usage record
{
  _id: ObjectId(),
  timestamp: ISODate('2026-06-22T10:30:00Z'),
  metadata: {
    subscriptionId: 'sub-123',
    customerId: 'cust-456',
    productId: 'prod-789'
  },
  quantity: 150.5,
  unit: 'GB',
  type: 'data_transfer',
  metadata: {
    region: 'ap-southeast-1',
    source: 'api'
  }
}

// 3. Audit Logs (Append-Only)
db.createCollection('audit_logs', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['userId', 'action', 'entityType', 'entityId', 'timestamp'],
      properties: {
        userId: { bsonType: 'string' },
        action: { bsonType: 'string' },
        entityType: { bsonType: 'string' },
        entityId: { bsonType: 'string' },
        changes: { bsonType: 'object' },
        ipAddress: { bsonType: 'string' },
        userAgent: { bsonType: 'string' },
        timestamp: { bsonType: 'date' }
      }
    }
  }
});

// 4. Document Store
db.createCollection('documents', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['title', 'type', 'ownerId', 'createdAt'],
      properties: {
        title: { bsonType: 'string' },
        type: { 
          bsonType: 'string',
          enum: ['contract', 'proposal', 'invoice', 'report', 'other']
        },
        ownerId: { bsonType: 'string' },
        accountId: { bsonType: 'string' },
        content: { bsonType: 'object' },
        attachments: {
          bsonType: 'array',
          items: {
            bsonType: 'object',
            properties: {
              filename: { bsonType: 'string' },
              url: { bsonType: 'string' },
              size: { bsonType: 'number' },
              mimeType: { bsonType: 'string' }
            }
          }
        },
        metadata: { bsonType: 'object' },
        createdAt: { bsonType: 'date' },
        updatedAt: { bsonType: 'date' }
      }
    }
  }
});

// 5. Feature Flags
db.createCollection('feature_flags', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['key', 'name', 'enabled', 'createdAt'],
      properties: {
        key: { bsonType: 'string' },
        name: { bsonType: 'string' },
        description: { bsonType: 'string' },
        enabled: { bsonType: 'bool' },
        rollout: {
          bsonType: 'object',
          properties: {
            percentage: { bsonType: 'number' },
            userIds: { bsonType: 'array', items: { bsonType: 'string' } },
            tenantIds: { bsonType: 'array', items: { bsonType: 'string' } }
          }
        },
        createdAt: { bsonType: 'date' },
        updatedAt: { bsonType: 'date' }
      }
    }
  }
});
```

### 4.3 Redis Data Structures

```javascript
// Redis Key Patterns and Data Structures

// 1. User Sessions (Hash)
// Key: session:{session_id}
// TTL: 3600 seconds (1 hour)
HSET session:abc123 user_id "user-456" role "sales_rep" tenant_id "tenant-789" last_active "2026-06-22T10:30:00Z"
EXPIRE session:abc123 3600

// 2. Pricing Cache (Hash)
// Key: pricing:{product_id}:{price_book_id}
// TTL: 3600 seconds
HSET pricing:prod-123:pb-456 unit_price "1500.00" currency "USD" discount "10" effective_date "2026-01-01"
EXPIRE pricing:prod-123:pb-456 3600

// 3. Rate Limiting (String with INCR)
// Key: rate_limit:{ip}:{endpoint}
// TTL: 60 seconds
INCR rate_limit:192.168.1.1:/api/quotes
EXPIRE rate_limit:192.168.1.1:/api/quotes 60

// 4. Distributed Locks (String with SET NX)
// Key: lock:{resource}:{id}
// TTL: 30 seconds
SET lock:quote:abc123 "locked" NX EX 30

// 5. Real-time Metrics (Sorted Set)
// Key: metrics:{user_id}:{metric_type}
// TTL: 86400 seconds (24 hours)
ZADD metrics:user-456:quotes_created 1640000000 "quote-123"
ZADD metrics:user-456:quotes_created 1640000100 "quote-124"
EXPIRE metrics:user-456:quotes_created 86400

// 6. Pub/Sub Channels
// Channel: invoice:events
PUBLISH invoice:events '{"event":"invoice_generated","invoiceId":"inv-123","amount":1500}'

// 7. Dashboard Cache (String with JSON)
// Key: dashboard:{user_id}:summary
// TTL: 300 seconds (5 minutes)
SET dashboard:user-456:summary '{"totalRevenue":150000,"activeQuotes":25,"pendingInvoices":10}' EX 300

// 8. Quote Approval Queue (List)
// Key: queue:quote_approvals
LPUSH queue:quote_approvals '{"quoteId":"quote-123","submittedBy":"user-456","timestamp":"2026-06-22T10:30:00Z"}'
RPOP queue:quote_approvals
```

---

## 5. Indexes

### 5.1 PostgreSQL Indexes

```sql
-- ============================================================
-- PRIMARY KEY INDEXES (Automatically created)
-- ============================================================

-- ============================================================
-- FOREIGN KEY INDEXES
-- ============================================================

-- Accounts
CREATE INDEX idx_accounts_owner_id ON accounts(owner_id);
CREATE INDEX idx_accounts_type ON accounts(type) WHERE is_active = true;

-- Opportunities
CREATE INDEX idx_opportunities_account_id ON opportunities(account_id);
CREATE INDEX idx_opportunities_owner_id ON opportunities(owner_id);
CREATE INDEX idx_opportunities_stage ON opportunities(stage) WHERE is_deleted = false;
CREATE INDEX idx_opportunities_close_date ON opportunities(close_date);
CREATE INDEX idx_opportunities_created_at ON opportunities(created_at DESC);

-- Quotes
CREATE INDEX idx_quotes_opportunity_id ON quotes(opportunity_id);
CREATE INDEX idx_quotes_status ON quotes(status);
CREATE INDEX idx_quotes_expiration_date ON quotes(expiration_date);
CREATE INDEX idx_quotes_created_by ON quotes(created_by);
CREATE INDEX idx_quotes_created_at ON quotes(created_at DESC);

-- Quote Line Items
CREATE INDEX idx_quote_line_items_quote_id ON quote_line_items(quote_id);
CREATE INDEX idx_quote_line_items_product_id ON quote_line_items(product_id);

-- Orders
CREATE INDEX idx_orders_quote_id ON orders(quote_id);
CREATE INDEX idx_orders_account_id ON orders(account_id);
CREATE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_order_date ON orders(order_date);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Order Items
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Invoices
CREATE INDEX idx_invoices_order_id ON invoices(order_id);
CREATE INDEX idx_invoices_account_id ON invoices(account_id);
CREATE INDEX idx_invoices_invoice_number ON invoices(invoice_number);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date);
CREATE INDEX idx_invoices_issue_date ON invoices(issue_date);
CREATE INDEX idx_invoices_created_at ON invoices(created_at DESC);

-- Invoice Line Items
CREATE INDEX idx_invoice_line_items_invoice_id ON invoice_line_items(invoice_id);
CREATE INDEX idx_invoice_line_items_order_item_id ON invoice_line_items(order_item_id);

-- Contracts
CREATE INDEX idx_contracts_order_id ON contracts(order_id);
CREATE INDEX idx_contracts_account_id ON contracts(account_id);
CREATE INDEX idx_contracts_contract_number ON contracts(contract_number);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_start_date ON contracts(start_date);
CREATE INDEX idx_contracts_end_date ON contracts(end_date);
CREATE INDEX idx_contracts_created_at ON contracts(created_at DESC);

-- Assets
CREATE INDEX idx_assets_order_id ON assets(order_id);
CREATE INDEX idx_assets_order_item_id ON assets(order_item_id);
CREATE INDEX idx_assets_account_id ON assets(account_id);
CREATE INDEX idx_assets_product_id ON assets(product_id);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_assets_activation_date ON assets(activation_date);
CREATE INDEX idx_assets_expiration_date ON assets(expiration_date);

-- Billing Schedules
CREATE INDEX idx_billing_schedules_order_id ON billing_schedules(order_id);
CREATE INDEX idx_billing_schedules_order_item_id ON billing_schedules(order_item_id);
CREATE INDEX idx_billing_schedules_account_id ON billing_schedules(account_id);
CREATE INDEX idx_billing_schedules_next_billing_date ON billing_schedules(next_billing_date);
CREATE INDEX idx_billing_schedules_status ON billing_schedules(status);

-- Payments
CREATE INDEX idx_payments_invoice_id ON payments(invoice_id);
CREATE INDEX idx_payments_account_id ON payments(account_id);
CREATE INDEX idx_payments_payment_number ON payments(payment_number);
CREATE INDEX idx_payments_payment_date ON payments(payment_date);
CREATE INDEX idx_payments_status ON payments(status);

-- Products
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_product_type ON products(product_type);
CREATE INDEX idx_products_is_active ON products(is_active) WHERE is_active = true;

-- Price Books
CREATE INDEX idx_price_books_currency ON price_books(currency);
CREATE INDEX idx_price_books_is_active ON price_books(is_active) WHERE is_active = true;

-- Price Book Entries
CREATE INDEX idx_price_book_entries_price_book_id ON price_book_entries(price_book_id);
CREATE INDEX idx_price_book_entries_product_id ON price_book_entries(product_id);
CREATE INDEX idx_price_book_entries_effective_date ON price_book_entries(effective_date);

-- Users
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_is_active ON users(is_active) WHERE is_active = true;

-- Audit Logs
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_entity_type ON audit_logs(entity_type);
CREATE INDEX idx_audit_logs_entity_id ON audit_logs(entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- ============================================================
-- COMPOSITE INDEXES
-- ============================================================

-- Opportunities: Common query patterns
CREATE INDEX idx_opportunities_account_stage ON opportunities(account_id, stage) WHERE is_deleted = false;
CREATE INDEX idx_opportunities_owner_stage ON opportunities(owner_id, stage) WHERE is_deleted = false;
CREATE INDEX idx_opportunities_stage_close_date ON opportunities(stage, close_date) WHERE is_deleted = false;

-- Quotes: Common query patterns
CREATE INDEX idx_quotes_opportunity_status ON quotes(opportunity_id, status);
CREATE INDEX idx_quotes_status_expiration ON quotes(status, expiration_date);

-- Orders: Common query patterns
CREATE INDEX idx_orders_account_status ON orders(account_id, status);
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- Invoices: Common query patterns
CREATE INDEX idx_invoices_account_status ON invoices(account_id, status);
CREATE INDEX idx_invoices_status_due_date ON invoices(status, due_date);
CREATE INDEX idx_invoices_due_date_status ON invoices(due_date, status) WHERE status IN ('issued', 'sent', 'overdue');

-- Contracts: Common query patterns
CREATE INDEX idx_contracts_account_status ON contracts(account_id, status);
CREATE INDEX idx_contracts_status_dates ON contracts(status, start_date, end_date);

-- Assets: Common query patterns
CREATE INDEX idx_assets_account_product ON assets(account_id, product_id);
CREATE INDEX idx_assets_status_expiration ON assets(status, expiration_date);

-- Billing Schedules: Common query patterns
CREATE INDEX idx_billing_schedules_account_next_date ON billing_schedules(account_id, next_billing_date);
CREATE INDEX idx_billing_schedules_status_next_date ON billing_schedules(status, next_billing_date) WHERE status = 'active';

-- Payments: Common query patterns
CREATE INDEX idx_payments_invoice_status ON payments(invoice_id, status);
CREATE INDEX idx_payments_account_date ON payments(account_id, payment_date DESC);

-- Audit Logs: Common query patterns
CREATE INDEX idx_audit_logs_entity_type_id ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_user_action ON audit_logs(user_id, action);

-- ============================================================
-- PARTIAL INDEXES (Filtered)
-- ============================================================

-- Active opportunities only
CREATE INDEX idx_opportunities_active ON opportunities(account_id, stage, close_date) 
    WHERE is_deleted = false AND stage NOT IN ('closed_won', 'closed_lost');

-- Overdue invoices only
CREATE INDEX idx_invoices_overdue ON invoices(account_id, due_date, total_amount) 
    WHERE status IN ('issued', 'sent') AND due_date < CURRENT_DATE;

-- Active contracts only
CREATE INDEX idx_contracts_active ON contracts(account_id, end_date) 
    WHERE status = 'active';

-- Active assets only
CREATE INDEX idx_assets_active ON assets(account_id, product_id, expiration_date) 
    WHERE status = 'active';

-- Active billing schedules only
CREATE INDEX idx_billing_schedules_active ON billing_schedules(account_id, next_billing_date, amount) 
    WHERE status = 'active';

-- ============================================================
-- FULL-TEXT SEARCH INDEXES
-- ============================================================

-- Products: Full-text search
CREATE INDEX idx_products_search ON products 
    USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '') || ' ' || COALESCE(sku, '')));

-- Opportunities: Full-text search
CREATE INDEX idx_opportunities_search ON opportunities 
    USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- Accounts: Full-text search
CREATE INDEX idx_accounts_search ON accounts 
    USING gin(to_tsvector('english', name || ' ' || COALESCE(industry, '')));

-- ============================================================
-- JSONB INDEXES
-- ============================================================

-- Products: Index on JSONB attributes
CREATE INDEX idx_products_attributes ON products USING gin(attributes);
CREATE INDEX idx_products_attributes_category ON products 
    USING gin((attributes->>'category'));

-- Accounts: Index on JSONB metadata
CREATE INDEX idx_accounts_metadata ON accounts USING gin(metadata);

-- Quote Line Items: Index on JSONB configuration
CREATE INDEX idx_quote_line_items_config ON quote_line_items USING gin(configuration);

-- Assets: Index on JSONB entitlements
CREATE INDEX idx_assets_entitlements ON assets USING gin(entitlements);
CREATE INDEX idx_assets_metadata ON assets USING gin(metadata);

-- ============================================================
-- EXPRESSION INDEXES
-- ============================================================

-- Invoices: Amount due calculation
CREATE INDEX idx_invoices_amount_due ON invoices((total_amount - amount_paid)) 
    WHERE status IN ('issued', 'sent', 'overdue');

-- Opportunities: Weighted amount (amount * probability / 100)
CREATE INDEX idx_opportunities_weighted_amount ON opportunities((amount * probability / 100)) 
    WHERE is_deleted = false AND stage NOT IN ('closed_won', 'closed_lost');

-- ============================================================
-- COVERING INDEXES (INCLUDE)
-- ============================================================

-- Opportunities: Covering index for common queries
CREATE INDEX idx_opportunities_covering ON opportunities(account_id, stage, close_date) 
    INCLUDE (name, amount, probability) 
    WHERE is_deleted = false;

-- Invoices: Covering index for overdue queries
CREATE INDEX idx_invoices_overdue_covering ON invoices(due_date, status) 
    INCLUDE (invoice_number, account_id, total_amount, amount_due) 
    WHERE status IN ('issued', 'sent') AND due_date < CURRENT_DATE;

-- Orders: Covering index for status queries
CREATE INDEX idx_orders_status_covering ON orders(status, order_date) 
    INCLUDE (order_number, account_id, total_amount) 
    WHERE status IN ('pending', 'activated');
```

### 5.2 MongoDB Indexes

```javascript
// MongoDB Indexes

// 1. Product Catalog Indexes
db.product_catalog.createIndex({ sku: 1 }, { unique: true });
db.product_catalog.createIndex({ name: 1 });
db.product_catalog.createIndex({ category: 1, subcategory: 1 });
db.product_catalog.createIndex({ isActive: 1 });
db.product_catalog.createIndex({ 'pricing.basePrice': 1 });
db.product_catalog.createIndex({ 'attributes.brand': 1 });
db.product_catalog.createIndex({ createdAt: -1 });

// Text index for search
db.product_catalog.createIndex({ 
  name: 'text', 
  description: 'text', 
  sku: 'text' 
});

// 2. Usage Records Indexes (Time-Series)
db.usage_records.createIndex({ 'metadata.subscriptionId': 1, timestamp: -1 });
db.usage_records.createIndex({ 'metadata.customerId': 1, timestamp: -1 });
db.usage_records.createIndex({ 'metadata.productId': 1, timestamp: -1 });
db.usage_records.createIndex({ type: 1, timestamp: -1 });

// 3. Audit Logs Indexes
db.audit_logs.createIndex({ userId: 1, timestamp: -1 });
db.audit_logs.createIndex({ entityType: 1, entityId: 1, timestamp: -1 });
db.audit_logs.createIndex({ action: 1, timestamp: -1 });
db.audit_logs.createIndex({ timestamp: -1 });

// 4. Documents Indexes
db.documents.createIndex({ ownerId: 1, type: 1 });
db.documents.createIndex({ accountId: 1, type: 1 });
db.documents.createIndex({ type: 1, createdAt: -1 });
db.documents.createIndex({ 'metadata.tags': 1 });

// 5. Feature Flags Indexes
db.feature_flags.createIndex({ key: 1 }, { unique: true });
db.feature_flags.createIndex({ enabled: 1 });
db.feature_flags.createIndex({ 'rollout.userIds': 1 });
db.feature_flags.createIndex({ 'rollout.tenantIds': 1 });
```

### 5.3 Redis Index Patterns

```javascript
// Redis doesn't have traditional indexes, but uses key patterns

// Pattern-based access:
// - session:* - All sessions
// - pricing:* - All pricing cache
// - rate_limit:* - All rate limit counters
// - lock:* - All distributed locks
// - metrics:* - All metrics
// - dashboard:* - All dashboard caches
// - queue:* - All queues

// Use SCAN for pattern matching (not KEYS)
// SCAN 0 MATCH session:* COUNT 100
```

---

## 6. Constraints

### 6.1 Data Integrity Constraints

```sql
-- ============================================================
-- CHECK CONSTRAINTS
-- ============================================================

-- Monetary values must be non-negative
ALTER TABLE opportunities ADD CONSTRAINT chk_opportunities_amount 
    CHECK (amount >= 0);

ALTER TABLE quotes ADD CONSTRAINT chk_quotes_total_amount 
    CHECK (total_amount >= 0);

ALTER TABLE orders ADD CONSTRAINT chk_orders_total_amount 
    CHECK (total_amount >= 0);

ALTER TABLE invoices ADD CONSTRAINT chk_invoices_total_amount 
    CHECK (total_amount >= 0);

ALTER TABLE payments ADD CONSTRAINT chk_payments_amount 
    CHECK (amount > 0);

-- Percentages must be between 0 and 100
ALTER TABLE quote_line_items ADD CONSTRAINT chk_qli_discount_percent 
    CHECK (discount_percent BETWEEN 0 AND 100);

ALTER TABLE quote_line_items ADD CONSTRAINT chk_qli_tax_percent 
    CHECK (tax_percent BETWEEN 0 AND 100);

-- Dates must be logical
ALTER TABLE contracts ADD CONSTRAINT chk_contracts_dates 
    CHECK (end_date > start_date);

ALTER TABLE price_books ADD CONSTRAINT chk_price_books_dates 
    CHECK (expiration_date IS NULL OR expiration_date > effective_date);

-- Quantities must be positive
ALTER TABLE quote_line_items ADD CONSTRAINT chk_qli_quantity 
    CHECK (quantity > 0);

ALTER TABLE order_items ADD CONSTRAINT chk_oi_quantity 
    CHECK (quantity > 0);

-- ============================================================
-- UNIQUE CONSTRAINTS
-- ============================================================

-- Business keys
ALTER TABLE orders ADD CONSTRAINT uq_orders_order_number 
    UNIQUE (order_number);

ALTER TABLE invoices ADD CONSTRAINT uq_invoices_invoice_number 
    UNIQUE (invoice_number);

ALTER TABLE contracts ADD CONSTRAINT uq_contracts_contract_number 
    UNIQUE (contract_number);

ALTER TABLE payments ADD CONSTRAINT uq_payments_payment_number 
    UNIQUE (payment_number);

ALTER TABLE products ADD CONSTRAINT uq_products_sku 
    UNIQUE (sku);

ALTER TABLE users ADD CONSTRAINT uq_users_username 
    UNIQUE (username);

ALTER TABLE users ADD CONSTRAINT uq_users_email 
    UNIQUE (email);

-- ============================================================
-- FOREIGN KEY CONSTRAINTS
-- ============================================================

-- Cascade rules
-- ON DELETE CASCADE: Child records deleted when parent deleted
-- ON DELETE RESTRICT: Prevent deletion if child records exist
-- ON DELETE SET NULL: Set foreign key to NULL when parent deleted

-- Quotes depend on Opportunities (restrict deletion)
ALTER TABLE quotes ADD CONSTRAINT fk_quotes_opportunity 
    FOREIGN KEY (opportunity_id) REFERENCES opportunities(id) ON DELETE RESTRICT;

-- Quote line items cascade with quotes
ALTER TABLE quote_line_items ADD CONSTRAINT fk_qli_quote 
    FOREIGN KEY (quote_id) REFERENCES quotes(id) ON DELETE CASCADE;

-- Orders can exist without quotes (set null)
ALTER TABLE orders ADD CONSTRAINT fk_orders_quote 
    FOREIGN KEY (quote_id) REFERENCES quotes(id) ON DELETE SET NULL;

-- Order items cascade with orders
ALTER TABLE order_items ADD CONSTRAINT fk_oi_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;

-- Invoices can exist without orders (set null)
ALTER TABLE invoices ADD CONSTRAINT fk_invoices_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE SET NULL;

-- Invoice line items cascade with invoices
ALTER TABLE invoice_line_items ADD CONSTRAINT fk_ili_invoice 
    FOREIGN KEY (invoice_id) REFERENCES invoices(id) ON DELETE CASCADE;

-- Contracts can exist without orders (set null)
ALTER TABLE contracts ADD CONSTRAINT fk_contracts_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE SET NULL;

-- Assets depend on orders (restrict deletion)
ALTER TABLE assets ADD CONSTRAINT fk_assets_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE RESTRICT;

-- Billing schedules cascade with orders
ALTER TABLE billing_schedules ADD CONSTRAINT fk_bs_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;

-- Payments depend on invoices (restrict deletion)
ALTER TABLE payments ADD CONSTRAINT fk_payments_invoice 
    FOREIGN KEY (invoice_id) REFERENCES invoices(id) ON DELETE RESTRICT;

-- ============================================================
-- NOT NULL CONSTRAINTS
-- ============================================================

-- Critical fields that must always have values
ALTER TABLE opportunities ALTER COLUMN account_id SET NOT NULL;
ALTER TABLE opportunities ALTER COLUMN name SET NOT NULL;
ALTER TABLE opportunities ALTER COLUMN stage SET NOT NULL;
ALTER TABLE opportunities ALTER COLUMN close_date SET NOT NULL;

ALTER TABLE quotes ALTER COLUMN opportunity_id SET NOT NULL;
ALTER TABLE quotes ALTER COLUMN name SET NOT NULL;
ALTER TABLE quotes ALTER COLUMN status SET NOT NULL;
ALTER TABLE quotes ALTER COLUMN expiration_date SET NOT NULL;

ALTER TABLE orders ALTER COLUMN account_id SET NOT NULL;
ALTER TABLE orders ALTER COLUMN order_number SET NOT NULL;
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
ALTER TABLE orders ALTER COLUMN order_date SET NOT NULL;

ALTER TABLE invoices ALTER COLUMN account_id SET NOT NULL;
ALTER TABLE invoices ALTER COLUMN invoice_number SET NOT NULL;
ALTER TABLE invoices ALTER COLUMN status SET NOT NULL;
ALTER TABLE invoices ALTER COLUMN due_date SET NOT NULL;

ALTER TABLE contracts ALTER COLUMN account_id SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN contract_number SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN title SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN status SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN contract_type SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN start_date SET NOT NULL;
ALTER TABLE contracts ALTER COLUMN end_date SET NOT NULL;

-- ============================================================
-- DEFAULT CONSTRAINTS
-- ============================================================

-- Default values for common fields
ALTER TABLE accounts ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE accounts ALTER COLUMN is_active SET DEFAULT true;

ALTER TABLE opportunities ALTER COLUMN stage SET DEFAULT 'qualification';
ALTER TABLE opportunities ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE opportunities ALTER COLUMN is_deleted SET DEFAULT false;

ALTER TABLE quotes ALTER COLUMN status SET DEFAULT 'draft';
ALTER TABLE quotes ALTER COLUMN currency SET DEFAULT 'USD';

ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';
ALTER TABLE orders ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE orders ALTER COLUMN order_date SET DEFAULT CURRENT_DATE;

ALTER TABLE invoices ALTER COLUMN status SET DEFAULT 'draft';
ALTER TABLE invoices ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE invoices ALTER COLUMN issue_date SET DEFAULT CURRENT_DATE;

ALTER TABLE contracts ALTER COLUMN status SET DEFAULT 'draft';
ALTER TABLE contracts ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE contracts ALTER COLUMN auto_renewal SET DEFAULT false;
ALTER TABLE contracts ALTER COLUMN renewal_notice_days SET DEFAULT 30;

ALTER TABLE assets ALTER COLUMN status SET DEFAULT 'active';
ALTER TABLE assets ALTER COLUMN quantity SET DEFAULT 1;

ALTER TABLE billing_schedules ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE billing_schedules ALTER COLUMN status SET DEFAULT 'active';

ALTER TABLE payments ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE payments ALTER COLUMN payment_date SET DEFAULT CURRENT_TIMESTAMP;
ALTER TABLE payments ALTER COLUMN status SET DEFAULT 'completed';

ALTER TABLE products ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE products ALTER COLUMN is_active SET DEFAULT true;
ALTER TABLE products ALTER COLUMN is_taxable SET DEFAULT true;

ALTER TABLE price_books ALTER COLUMN currency SET DEFAULT 'USD';
ALTER TABLE price_books ALTER COLUMN is_active SET DEFAULT true;
ALTER TABLE price_books ALTER COLUMN is_standard SET DEFAULT false;

ALTER TABLE users ALTER COLUMN is_active SET DEFAULT true;

-- ============================================================
-- EXCLUSION CONSTRAINTS (Prevent overlapping ranges)
-- ============================================================

-- Prevent overlapping price book entries for same product
ALTER TABLE price_book_entries ADD CONSTRAINT excl_pbe_overlap 
    EXCLUDE USING gist (
        price_book_id WITH =,
        product_id WITH =,
        daterange(effective_date, COALESCE(expiration_date, 'infinity')) WITH &&
    );

-- Prevent overlapping contracts for same account (optional business rule)
-- ALTER TABLE contracts ADD CONSTRAINT excl_contracts_overlap 
--     EXCLUDE USING gist (
--         account_id WITH =,
--         daterange(start_date, end_date) WITH &&
--     ) WHERE (status = 'active');
```

### 6.2 MongoDB Validation Constraints

```javascript
// MongoDB JSON Schema Validation

// Product Catalog Validation
db.createCollection('product_catalog', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['sku', 'name', 'category', 'createdAt'],
      additionalProperties: false,
      properties: {
        sku: { 
          bsonType: 'string',
          minLength: 1,
          maxLength: 100,
          description: 'SKU must be a non-empty string'
        },
        name: { 
          bsonType: 'string',
          minLength: 1,
          maxLength: 255,
          description: 'Name must be a non-empty string'
        },
        category: { 
          bsonType: 'string',
          enum: ['software', 'hardware', 'service', 'subscription', 'other']
        },
        pricing: {
          bsonType: 'object',
          required: ['basePrice', 'currency'],
          properties: {
            basePrice: { 
              bsonType: 'number',
              minimum: 0
            },
            currency: { 
              bsonType: 'string',
              enum: ['USD', 'THB', 'EUR', 'GBP', 'SGD']
            },
            tiers: {
              bsonType: 'array',
              items: {
                bsonType: 'object',
                required: ['minQuantity', 'price'],
                properties: {
                  minQuantity: { bsonType: 'number', minimum: 1 },
                  maxQuantity: { bsonType: ['number', 'null'], minimum: 1 },
                  price: { bsonType: 'number', minimum: 0 }
                }
              }
            }
          }
        },
        isActive: { bsonType: 'bool' },
        createdAt: { bsonType: 'date' },
        updatedAt: { bsonType: 'date' }
      }
    }
  }
});

// Usage Records Validation
db.createCollection('usage_records', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['timestamp', 'metadata', 'quantity', 'unit'],
      properties: {
        timestamp: { bsonType: 'date' },
        metadata: {
          bsonType: 'object',
          required: ['subscriptionId', 'customerId', 'productId'],
          properties: {
            subscriptionId: { bsonType: 'string' },
            customerId: { bsonType: 'string' },
            productId: { bsonType: 'string' }
          }
        },
        quantity: { 
          bsonType: 'number',
          minimum: 0
        },
        unit: { 
          bsonType: 'string',
          enum: ['GB', 'hours', 'requests', 'units', 'minutes']
        },
        type: { 
          bsonType: 'string',
          enum: ['data_transfer', 'compute', 'api_calls', 'storage', 'other']
        }
      }
    }
  }
});
```

---

## 7. Partition Strategy

### 7.1 Table Partitioning (PostgreSQL)

```sql
-- ============================================================
-- PARTITIONING STRATEGY
-- ============================================================

-- Partitioning improves query performance and manageability for large tables
-- Common strategies: Range, List, Hash

-- ============================================================
-- 1. RANGE PARTITIONING: Invoices by Date
-- ============================================================

-- Create partitioned table
CREATE TABLE invoices_partitioned (
    id UUID NOT NULL DEFAULT uuid_generate_v4(),
    order_id UUID,
    account_id UUID NOT NULL,
    invoice_number VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    subtotal DECIMAL(15,2) NOT NULL,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    total_amount DECIMAL(15,2) NOT NULL,
    amount_paid DECIMAL(15,2) DEFAULT 0,
    amount_due DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    issue_date DATE NOT NULL,
    due_date DATE NOT NULL,
    paid_date TIMESTAMP WITH TIME ZONE,
    notes TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, issue_date)
) PARTITION BY RANGE (issue_date);

-- Create partitions by year
CREATE TABLE invoices_2024 PARTITION OF invoices_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE invoices_2025 PARTITION OF invoices_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE invoices_2026 PARTITION OF invoices_partitioned
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE TABLE invoices_2027 PARTITION OF invoices_partitioned
    FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');

-- Default partition for future dates
CREATE TABLE invoices_default PARTITION OF invoices_partitioned
    DEFAULT;

-- Indexes on partitioned table
CREATE INDEX idx_invoices_part_account ON invoices_partitioned(account_id, issue_date);
CREATE INDEX idx_invoices_part_status ON invoices_partitioned(status, issue_date);
CREATE INDEX idx_invoices_part_due_date ON invoices_partitioned(due_date);

-- ============================================================
-- 2. RANGE PARTITIONING: Audit Logs by Month
-- ============================================================

CREATE TABLE audit_logs_partitioned (
    id UUID NOT NULL DEFAULT uuid_generate_v4(),
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create partitions by month
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs_partitioned
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs_partitioned
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE audit_logs_2026_03 PARTITION OF audit_logs_partitioned
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Continue for other months...

-- Default partition
CREATE TABLE audit_logs_default PARTITION OF audit_logs_partitioned
    DEFAULT;

-- ============================================================
-- 3. LIST PARTITIONING: Orders by Status
-- ============================================================

CREATE TABLE orders_partitioned (
    id UUID NOT NULL DEFAULT uuid_generate_v4(),
    quote_id UUID,
    account_id UUID NOT NULL,
    order_number VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    order_date DATE NOT NULL,
    activation_date TIMESTAMP WITH TIME ZONE,
    fulfillment_date TIMESTAMP WITH TIME ZONE,
    cancellation_reason TEXT,
    notes TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, status)
) PARTITION BY LIST (status);

-- Create partitions by status
CREATE TABLE orders_pending PARTITION OF orders_partitioned
    FOR VALUES IN ('pending');

CREATE TABLE orders_activated PARTITION OF orders_partitioned
    FOR VALUES IN ('activated');

CREATE TABLE orders_fulfilled PARTITION OF orders_partitioned
    FOR VALUES IN ('fulfilled');

CREATE TABLE orders_cancelled PARTITION OF orders_partitioned
    FOR VALUES IN ('cancelled', 'expired');

-- Default partition
CREATE TABLE orders_default PARTITION OF orders_partitioned
    DEFAULT;

-- ============================================================
-- 4. HASH PARTITIONING: Usage Records (MongoDB Sharding Alternative)
-- ============================================================

-- For very high-volume tables, use hash partitioning
CREATE TABLE usage_records_hash (
    id UUID NOT NULL DEFAULT uuid_generate_v4(),
    subscription_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    product_id UUID NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    quantity DECIMAL(10,2) NOT NULL,
    unit VARCHAR(50) NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, subscription_id)
) PARTITION BY HASH (subscription_id);

-- Create 8 hash partitions
CREATE TABLE usage_records_hash_0 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);

CREATE TABLE usage_records_hash_1 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);

CREATE TABLE usage_records_hash_2 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 2);

CREATE TABLE usage_records_hash_3 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 3);

CREATE TABLE usage_records_hash_4 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 4);

CREATE TABLE usage_records_hash_5 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 5);

CREATE TABLE usage_records_hash_6 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 6);

CREATE TABLE usage_records_hash_7 PARTITION OF usage_records_hash
    FOR VALUES WITH (MODULUS 8, REMAINDER 7);

-- ============================================================
-- PARTITION MANAGEMENT
-- ============================================================

-- Function to create new partitions automatically
CREATE OR REPLACE FUNCTION create_monthly_partitions()
RETURNS void AS $$
DECLARE
    next_month_start DATE;
    next_month_end DATE;
    partition_name TEXT;
BEGIN
    -- Calculate next month
    next_month_start := date_trunc('month', CURRENT_DATE + interval '1 month');
    next_month_end := next_month_start + interval '1 month';
    partition_name := 'audit_logs_' || to_char(next_month_start, 'YYYY_MM');
    
    -- Check if partition exists
    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = partition_name
    ) THEN
        -- Create partition
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF audit_logs_partitioned FOR VALUES FROM (%L) TO (%L)',
            partition_name, next_month_start, next_month_end
        );
        
        -- Create indexes on new partition
        EXECUTE format(
            'CREATE INDEX idx_%I_user ON %I(user_id)',
            partition_name, partition_name
        );
        EXECUTE format(
            'CREATE INDEX idx_%I_entity ON %I(entity_type, entity_id)',
            partition_name, partition_name
        );
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Schedule partition creation (run monthly)
-- SELECT create_monthly_partitions();

-- Function to drop old partitions (data retention)
CREATE OR REPLACE FUNCTION drop_old_partitions(retention_months INTEGER DEFAULT 12)
RETURNS void AS $$
DECLARE
    cutoff_date DATE;
    partition_record RECORD;
BEGIN
    cutoff_date := date_trunc('month', CURRENT_DATE - (retention_months || ' months')::interval);
    
    FOR partition_record IN 
        SELECT inhrelid::regclass::text AS partition_name
        FROM pg_inherits
        WHERE inhparent = 'audit_logs_partitioned'::regclass
        AND inhrelid::regclass::text LIKE 'audit_logs_%'
    LOOP
        -- Check if partition is older than retention period
        IF partition_record.partition_name < 'audit_logs_' || to_char(cutoff_date, 'YYYY_MM') THEN
            -- Detach partition
            EXECUTE format(
                'ALTER TABLE audit_logs_partitioned DETACH PARTITION %I',
                partition_record.partition_name
            );
            
            -- Drop partition
            EXECUTE format('DROP TABLE %I', partition_record.partition_name);
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Schedule old partition cleanup (run monthly)
-- SELECT drop_old_partitions(12);
```

### 7.2 MongoDB Sharding Strategy

```javascript
// MongoDB Sharding Configuration

// Enable sharding on database
sh.enableSharding('revenue_mgmt');

// 1. Shard Usage Records by subscriptionId (High volume)
sh.shardCollection('revenue_mgmt.usage_records', {
  'metadata.subscriptionId': 'hashed'
});

// 2. Shard Audit Logs by timestamp (Time-series)
sh.shardCollection('revenue_mgmt.audit_logs', {
  'timestamp': 1,
  '_id': 1
});

// 3. Shard Product Catalog by category (Read-heavy)
sh.shardCollection('revenue_mgmt.product_catalog', {
  'category': 1,
  '_id': 1
});

// Add shard key indexes
db.usage_records.createIndex({ 'metadata.subscriptionId': 'hashed' });
db.audit_logs.createIndex({ 'timestamp': 1, '_id': 1 });
db.product_catalog.createIndex({ 'category': 1, '_id': 1 });
```

---

## 8. Replication Strategy

### 8.1 PostgreSQL Replication (Aurora)

```
┌─────────────────────────────────────────────────────────────────┐
│              AURORA REPLICATION ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIMARY CLUSTER (Singapore - ap-southeast-1)                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ──────────────┐    ┌──────────────┐    ┌──────────┐  │  │
│  │  │   Writer     │───▶│   Reader 1   │    │ Reader 2 │  │  │
│  │  │   Instance   │    │   Instance   │    │ Instance │  │  │
│  │  │              │    │              │    │          │  │  │
│  │  │ - Read/Write │    │ - Read Only  │    │ - Read   │  │  │
│  │  │ - Primary    │    │ - Replica    │    │ - Replica│  │  │
│  │  └──────────────┘    └──────────────┘    ──────────┘  │  │
│  │                                                          │  │
│  │  Shared Storage (Aurora Cluster Volume)                  │  │
│  │  - 6 copies across 3 AZs                                 │  │
│  │  - Automatic failover                                    │  │
│  ──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  GLOBAL CLUSTER (Cross-Region DR)                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ┌──────────────┐    ┌──────────────┐                   │  │
│  │  │   Reader 1   │    │   Reader 2   │                   │  │
│  │  │   (Tokyo)    │    │   (Tokyo)    │                   │  │
│  │  │              │    │              │                   │  │
│  │  │ - Read Only  │    │ - Read Only  │                   │  │
│  │  │ - DR         │    │ - DR         │                   │  │
│  │  └──────────────┘    └──────────────┘                   │  │
│  │                                                          │  │
│  │  Replication Lag: < 1 second (typically)                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  REPLICATION MODES                                               │
│  ├─ Synchronous: Writer waits for replica acknowledgment        │
│  ├─ Asynchronous: Writer doesn't wait (default)                 │
│  └─ Semi-Synchronous: Writer waits for at least one replica     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Aurora Configuration

```hcl
# Terraform: Aurora PostgreSQL Cluster
resource "aws_rds_cluster" "revenue_mgmt" {
  cluster_identifier     = "revenue-mgmt-aurora"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  database_name          = "revenue_mgmt"
  master_username        = "revenue_admin"
  master_password        = var.db_password
  backup_retention_period = 35
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "revenue-mgmt-final"

  # Enable global cluster for cross-region DR
  global_cluster_identifier = "revenue-mgmt-global"

  vpc_security_group_ids = [aws_security_group.aurora.id]
  db_subnet_group_name   = aws_db_subnet_group.aurora.name

  # Enable CloudWatch logs
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  # Performance Insights
  performance_insights_enabled = true
  performance_insights_retention_period = 7

  tags = {
    Name        = "revenue-mgmt-aurora"
    Environment = "production"
  }
}

# Writer Instance
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "revenue-mgmt-writer"
  cluster_identifier = aws_rds_cluster.revenue_mgmt.id
  instance_class     = "db.r5.2xlarge"
  engine             = aws_rds_cluster.revenue_mgmt.engine
  engine_version     = aws_rds_cluster.revenue_mgmt.engine_version
  
  performance_insights_enabled = true
  
  tags = {
    Name = "revenue-mgmt-writer"
    Role = "writer"
  }
}

# Reader Instances
resource "aws_rds_cluster_instance" "reader" {
  count              = 2
  identifier         = "revenue-mgmt-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.revenue_mgmt.id
  instance_class     = "db.r5.xlarge"
  engine             = aws_rds_cluster.revenue_mgmt.engine
  engine_version     = aws_rds_cluster.revenue_mgmt.engine_version
  
  performance_insights_enabled = true
  
  tags = {
    Name = "revenue-mgmt-reader-${count.index}"
    Role = "reader"
  }
}

# Global Cluster (Cross-Region)
resource "aws_rds_global_cluster" "revenue_mgmt" {
  global_cluster_identifier = "revenue-mgmt-global"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  deletion_protection       = true
}

# Secondary Cluster (Tokyo)
resource "aws_rds_cluster" "revenue_mgmt_dr" {
  cluster_identifier     = "revenue-mgmt-aurora-dr"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  global_cluster_identifier = aws_rds_global_cluster.revenue_mgmt.id
  
  # Note: master_username and master_password not needed for secondary
  
  vpc_security_group_ids = [aws_security_group.aurora_dr.id]
  db_subnet_group_name   = aws_db_subnet_group.aurora_dr.name
  
  tags = {
    Name        = "revenue-mgmt-aurora-dr"
    Environment = "production-dr"
  }
}
```

### 8.3 Read Replica Routing

```typescript
// NestJS: Database Connection with Read Replica Routing

// database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        // Primary (Writer)
        host: configService.get('DB_HOST'),
        port: configService.get('DB_PORT'),
        username: configService.get('DB_USERNAME'),
        password: configService.get('DB_PASSWORD'),
        database: configService.get('DB_DATABASE'),
        
        // Read Replicas
        replication: {
          master: {
            host: configService.get('DB_HOST'),
            port: configService.get('DB_PORT'),
            username: configService.get('DB_USERNAME'),
            password: configService.get('DB_PASSWORD'),
            database: configService.get('DB_DATABASE'),
          },
          slaves: [
            {
              host: configService.get('DB_READER_1_HOST'),
              port: configService.get('DB_PORT'),
              username: configService.get('DB_USERNAME'),
              password: configService.get('DB_PASSWORD'),
              database: configService.get('DB_DATABASE'),
            },
            {
              host: configService.get('DB_READER_2_HOST'),
              port: configService.get('DB_PORT'),
              username: configService.get('DB_USERNAME'),
              password: configService.get('DB_PASSWORD'),
              database: configService.get('DB_DATABASE'),
            },
          ],
        },
        
        // Connection Pool
        extra: {
          max: 20,
          min: 5,
          idleTimeoutMillis: 30000,
          connectionTimeoutMillis: 2000,
        },
        
        // Logging
        logging: configService.get('NODE_ENV') === 'development',
      }),
    }),
  ],
})
export class DatabaseModule {}

// read-replica.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const USE_READ_REPLICA = 'use_read_replica';
export const UseReadReplica = () => SetMetadata(USE_READ_REPLICA, true);

// read-replica.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Observable } from 'rxjs';
import { getRepository } from 'typeorm';

@Injectable()
export class ReadReplicaInterceptor implements NestInterceptor {
  constructor(private reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const useReadReplica = this.reflector.get<boolean>(
      USE_READ_REPLICA,
      context.getHandler(),
    );

    if (useReadReplica) {
      // Route to read replica
      const request = context.switchToHttp().getRequest();
      request.readReplica = true;
    }

    return next.handle();
  }
}

// Usage in controller
@Controller('opportunities')
export class OpportunitiesController {
  @Get()
  @UseReadReplica() // Route to read replica
  async findAll() {
    return this.opportunitiesService.findAll();
  }

  @Post()
  async create(@Body() dto: CreateOpportunityDto) {
    return this.opportunitiesService.create(dto); // Route to writer
  }
}
```

### 8.4 MongoDB Replica Set

```javascript
// MongoDB Replica Set Configuration

// Replica Set Members:
// - 1 Primary (Read/Write)
// - 2 Secondaries (Read Only)
// - 1 Arbiter (Voting only, no data)

// Connection String with Replica Set
const connectionString = 'mongodb://primary:27017,secondary1:27017,secondary2:27017/revenue_mgmt?replicaSet=rs0&readPreference=secondaryPreferred';

// Read Preferences:
// - primary: Read from primary only
// - primaryPreferred: Read from primary, fallback to secondary
// - secondary: Read from secondary only
// - secondaryPreferred: Read from secondary, fallback to primary
// - nearest: Read from nearest member (lowest latency)

// Write Concern:
// - w: 1 (acknowledged by primary)
// - w: majority (acknowledged by majority)
// - w: 0 (unacknowledged - not recommended)

// Journal:
// - j: true (wait for journal commit)
// - j: false (don't wait)

// Example with Mongoose
const mongoose = require('mongoose');

mongoose.connect(connectionString, {
  replicaSet: 'rs0',
  readPreference: 'secondaryPreferred',
  writeConcern: {
    w: 'majority',
    j: true,
    wtimeout: 5000
  },
  poolSize: 10,
  maxPoolSize: 50,
  minPoolSize: 5,
  socketTimeoutMS: 45000,
  connectTimeoutMS: 10000,
  serverSelectionTimeoutMS: 30000
});
```

### 8.5 Redis Cluster

```javascript
// Redis Cluster Configuration

const Redis = require('ioredis');

// Cluster mode with 3 masters and 3 replicas
const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
  { host: 'redis-node-3', port: 6379 },
  { host: 'redis-node-4', port: 6379 },
  { host: 'redis-node-5', port: 6379 },
  { host: 'redis-node-6', port: 6379 },
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    tls: {}, // Enable TLS
  },
  clusterRetryStrategy: (times) => {
    const delay = Math.min(times * 100, 3000);
    return delay;
  },
  maxRedirections: 16,
  retryDelayOnClusterDown: 300,
  retryDelayOnFailover: 300,
  retryDelayOnTryAgain: 300,
});

// Sentinel mode (alternative to cluster)
const sentinel = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 },
  ],
  name: 'revenue-mgmt-redis',
  password: process.env.REDIS_PASSWORD,
  sentinelPassword: process.env.REDIS_SENTINEL_PASSWORD,
  maxRetriesPerRequest: 3,
  retryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
});
```

---

## 9. Backup Strategy

### 9.1 Backup Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP STRATEGY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  POSTGRESQL (Aurora)                                             │
│  ├─ Automated Backups: Every 5 minutes (continuous)             │
│  ├─ Retention: 35 days                                          │
│  ├─ Point-in-Time Recovery: Yes                                 │
│  ├─ Cross-Region Snapshots: Daily to Tokyo                      │
│  ├─ Manual Snapshots: Before major changes                      │
│  └─ Export to S3: Weekly for long-term retention                │
│                                                                  │
│  MONGODB (Atlas)                                                 │
│  ├─ Continuous Backups: Oplog-based                             │
│  ├─ Retention: 7 days (continuous), 35 days (snapshots)         │
│  ├─ Point-in-Time Recovery: Yes                                 │
│  ├─ Cross-Region: Enabled                                       │
│  └─ Manual Snapshots: Before schema changes                     │
│                                                                  │
│  REDIS (ElastiCache)                                             │
│  ├─ Automated Backups: Daily                                    │
│  ├─ Retention: 7 days                                           │
│  ├─ Manual Snapshots: Before maintenance                        │
│  ─ Note: Cache can be regenerated from source                  │
│                                                                  │
│  SQL SERVER (Optional)                                           │
│  ├─ Full Backups: Weekly                                        │
│  ├─ Differential Backups: Daily                                 │
│  ├─ Transaction Log Backups: Every 15 minutes                   │
│  ├─ Retention: 35 days                                          │
│  └─ Cross-Region: Via Azure Blob or S3                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Aurora Backup Configuration

```sql
-- ============================================================
-- BACKUP VERIFICATION QUERIES
-- ============================================================

-- Check backup status
SELECT 
    db_cluster_identifier,
    status,
    earliest_restorable_time,
    latest_restorable_time
FROM aws_rds_db_cluster_snapshots
WHERE db_cluster_identifier = 'revenue-mgmt-aurora';

-- Point-in-Time Recovery Test
-- Restore to specific timestamp
-- aws rds restore-db-cluster-to-point-in-time \
--   --source-db-cluster-identifier revenue-mgmt-aurora \
--   --restore-to-time 2026-06-22T10:30:00Z \
--   --use-latest-restorable-time false \
--   --db-cluster-identifier revenue-mgmt-restore-test

-- ============================================================
-- BACKUP MONITORING
-- ============================================================

-- Create backup monitoring table
CREATE TABLE backup_monitoring (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    backup_type VARCHAR(50) NOT NULL,
    backup_start TIMESTAMP WITH TIME ZONE NOT NULL,
    backup_end TIMESTAMP WITH TIME ZONE,
    status VARCHAR(50) NOT NULL,
    size_bytes BIGINT,
    retention_days INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Function to log backup events
CREATE OR REPLACE FUNCTION log_backup_event(
    p_backup_type VARCHAR,
    p_backup_start TIMESTAMP WITH TIME ZONE,
    p_backup_end TIMESTAMP WITH TIME ZONE,
    p_status VARCHAR,
    p_size_bytes BIGINT,
    p_retention_days INTEGER
)
RETURNS void AS $$
BEGIN
    INSERT INTO backup_monitoring (
        backup_type, backup_start, backup_end, status, size_bytes, retention_days
    ) VALUES (
        p_backup_type, p_backup_start, p_backup_end, p_status, p_size_bytes, p_retention_days
    );
END;
$$ LANGUAGE plpgsql;

-- ============================================================
-- BACKUP RETENTION POLICY
-- ============================================================

-- Function to clean up old backups
CREATE OR REPLACE FUNCTION cleanup_old_backups(retention_days INTEGER DEFAULT 35)
RETURNS void AS $$
BEGIN
    DELETE FROM backup_monitoring
    WHERE backup_start < CURRENT_TIMESTAMP - (retention_days || ' days')::interval;
END;
$$ LANGUAGE plpgsql;
```

### 9.3 MongoDB Backup Configuration

```javascript
// MongoDB Backup Strategy

// 1. Automated Backups (Atlas)
// - Continuous backups using oplog
// - Daily snapshots
// - Retention: 7 days continuous, 35 days snapshots

// 2. Manual Backups
// Before schema changes or major operations
db.adminCommand({
  createBackup: 1,
  backupDir: '/backup/mongodb',
  compression: 'gzip'
});

// 3. Point-in-Time Recovery
// Restore to specific timestamp
// mongorestore --oplogReplay --oplogFile oplog.bson --ts <timestamp>

// 4. Backup Verification
// Verify backup integrity
db.adminCommand({
  validate: {
    collection: 'product_catalog',
    full: true
  }
});

// 5. Backup Monitoring
// Create backup log collection
db.createCollection('backup_logs', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['backupType', 'startTime', 'status'],
      properties: {
        backupType: { bsonType: 'string', enum: ['automated', 'manual', 'snapshot'] },
        startTime: { bsonType: 'date' },
        endTime: { bsonType: 'date' },
        status: { bsonType: 'string', enum: ['success', 'failed', 'in_progress'] },
        sizeBytes: { bsonType: 'number' },
        retentionDays: { bsonType: 'number' }
      }
    }
  }
});

// Log backup event
db.backup_logs.insertOne({
  backupType: 'manual',
  startTime: new Date(),
  endTime: new Date(),
  status: 'success',
  sizeBytes: 1073741824, // 1GB
  retentionDays: 35
});
```

### 9.4 Redis Backup Configuration

```javascript
// Redis Backup Strategy

// 1. Automated Backups (ElastiCache)
// - Daily automated backups
// - Retention: 7 days
// - Backup window: 03:00-04:00 UTC

// 2. Manual Backups
// Create snapshot
// aws elasticache create-snapshot \
//   --cache-cluster-id revenue-mgmt-redis \
//   --snapshot-name revenue-mgmt-manual-backup

// 3. Backup Verification
// Verify snapshot
// aws elasticache describe-snapshots \
//   --snapshot-name revenue-mgmt-manual-backup

// 4. Restore from Backup
// Restore cluster from snapshot
// aws elasticache create-cache-cluster \
//   --cache-cluster-id revenue-mgmt-redis-restore \
//   --cache-node-type cache.r5.large \
//   --engine redis \
//   --snapshot-name revenue-mgmt-manual-backup

// 5. Note: Redis is cache, data can be regenerated
// - Focus on backup for disaster recovery
// - Primary data source is PostgreSQL/MongoDB
```

### 9.5 Backup Schedule

```yaml
# Backup Schedule

PostgreSQL (Aurora):
  Continuous:
    - Frequency: Every 5 minutes
    - Retention: 35 days
    - Type: Transaction log backups
    - PITR: Enabled
  
  Daily Snapshot:
    - Time: 03:00 UTC
    - Retention: 35 days
    - Cross-Region: Tokyo
  
  Weekly Export:
    - Day: Sunday
    - Time: 04:00 UTC
    - Destination: S3 (Glacier)
    - Retention: 7 years

MongoDB (Atlas):
  Continuous:
    - Frequency: Oplog-based
    - Retention: 7 days
    - PITR: Enabled
  
  Daily Snapshot:
    - Time: 03:30 UTC
    - Retention: 35 days
    - Cross-Region: Enabled

Redis (ElastiCache):
  Daily Backup:
    - Time: 03:00 UTC
    - Retention: 7 days
    - Note: Cache can be regenerated

SQL Server (Optional):
  Full Backup:
    - Frequency: Weekly (Sunday)
    - Time: 02:00 UTC
    - Retention: 35 days
  
  Differential Backup:
    - Frequency: Daily
    - Time: 02:30 UTC
    - Retention: 35 days
  
  Transaction Log Backup:
    - Frequency: Every 15 minutes
    - Retention: 7 days
```

---

## 10. Data Retention

### 10.1 Retention Policy

```
┌─────────────────────────────────────────────────────────────────
│                  DATA RETENTION POLICY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRANSACTIONAL DATA (PostgreSQL)                                 │
│  ├─ Opportunities: 7 years (financial records)                  │
│  ├─ Quotes: 7 years                                            │
│  ├─ Orders: 10 years (legal requirement)                        │
│  ├─ Invoices: 10 years (tax records)                            │
│  ├─ Payments: 10 years (financial records)                      │
│  ├─ Contracts: 10 years after expiration                        │
│  ├─ Assets: 7 years after expiration                            │
│  └─ Billing Schedules: 7 years after completion                 │
│                                                                  │
│  OPERATIONAL DATA (PostgreSQL)                                   │
│  ├─ Audit Logs: 1 year (then archive to MongoDB)                │
│  ├─ Users: Indefinite (while active)                            │
│  ├─ Products: Indefinite (while active)                         │
│  └─ Price Books: 7 years after expiration                       │
│                                                                  │
│  HIGH-VOLUME DATA (MongoDB)                                      │
│  ├─ Usage Records: 30 days (TTL index)                          │
│  ├─ Audit Logs: 1 year (then delete)                            │
│  ├─ Documents: 10 years (business records)                      │
│  └─ Feature Flags: Indefinite                                   │
│                                                                  │
│  CACHE DATA (Redis)                                              │
│  ├─ Sessions: 1 hour (TTL)                                      │
│  ├─ Pricing Cache: 1 hour (TTL)                                 │
│  ├─ Rate Limits: 1 minute (TTL)                                 │
│  ├─ Distributed Locks: 30 seconds (TTL)                         │
│  ├─ Metrics: 24 hours (TTL)                                     │
│  └─ Dashboard Cache: 5 minutes (TTL)                            │
│                                                                  │
│  BACKUP DATA                                                     │
│  ├─ PostgreSQL: 35 days (automated), 7 years (S3)               │
│  ├─ MongoDB: 35 days (automated)                                │
│  ├─ Redis: 7 days (automated)                                   │
│  └─ SQL Server: 35 days (automated)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Data Retention Implementation

```sql
-- ============================================================
-- DATA RETENTION FUNCTIONS
-- ============================================================

-- Function to archive old audit logs to MongoDB
CREATE OR REPLACE FUNCTION archive_old_audit_logs(retention_days INTEGER DEFAULT 365)
RETURNS INTEGER AS $$
DECLARE
    archived_count INTEGER;
    cutoff_date TIMESTAMP WITH TIME ZONE;
BEGIN
    cutoff_date := CURRENT_TIMESTAMP - (retention_days || ' days')::interval;
    
    -- Get count of records to archive
    SELECT COUNT(*) INTO archived_count
    FROM audit_logs
    WHERE created_at < cutoff_date;
    
    -- Note: Actual archiving to MongoDB would be done via application
    -- This function just marks records for archival
    
    -- Delete archived records
    DELETE FROM audit_logs
    WHERE created_at < cutoff_date;
    
    RETURN archived_count;
END;
$$ LANGUAGE plpgsql;

-- Function to clean up expired quotes
CREATE OR REPLACE FUNCTION cleanup_expired_quotes()
RETURNS INTEGER AS $$
DECLARE
    cleaned_count INTEGER;
BEGIN
    UPDATE quotes
    SET status = 'expired'
    WHERE status = 'draft' 
    AND expiration_date < CURRENT_DATE;
    
    GET DIAGNOSTICS cleaned_count = ROW_COUNT;
    
    RETURN cleaned_count;
END;
$$ LANGUAGE plpgsql;

-- Function to clean up expired contracts
CREATE OR REPLACE FUNCTION cleanup_expired_contracts()
RETURNS INTEGER AS $$
DECLARE
    cleaned_count INTEGER;
BEGIN
    UPDATE contracts
    SET status = 'expired'
    WHERE status = 'active'
    AND end_date < CURRENT_DATE
    AND auto_renewal = false;
    
    GET DIAGNOSTICS cleaned_count = ROW_COUNT;
    
    RETURN cleaned_count;
END;
$$ LANGUAGE plpgsql;

-- Function to archive old invoices (soft delete)
CREATE OR REPLACE FUNCTION archive_old_invoices(retention_years INTEGER DEFAULT 7)
RETURNS INTEGER AS $$
DECLARE
    archived_count INTEGER;
    cutoff_date DATE;
BEGIN
    cutoff_date := CURRENT_DATE - (retention_years || ' years')::interval;
    
    -- Mark old paid invoices as archived
    UPDATE invoices
    SET status = 'archived'
    WHERE status = 'paid'
    AND paid_date < cutoff_date;
    
    GET DIAGNOSTICS archived_count = ROW_COUNT;
    
    RETURN archived_count;
END;
$$ LANGUAGE plpgsql;

-- ============================================================
-- SCHEDULED JOBS (pg_cron)
-- ============================================================

-- Enable pg_cron extension
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Schedule daily cleanup jobs
SELECT cron.schedule('cleanup-expired-quotes', '0 2 * * *', 
    $$SELECT cleanup_expired_quotes()$$);

SELECT cron.schedule('cleanup-expired-contracts', '0 3 * * *', 
    $$SELECT cleanup_expired_contracts()$$);

SELECT cron.schedule('archive-old-audit-logs', '0 4 * * 0', 
    $$SELECT archive_old_audit_logs(365)$$);

SELECT cron.schedule('archive-old-invoices', '0 5 * * 0', 
    $$SELECT archive_old_invoices(7)$$);
```

### 10.3 MongoDB TTL Indexes

```javascript
// MongoDB TTL Indexes for Automatic Data Expiration

// Usage Records: Expire after 30 days
db.usage_records.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 } // 30 days
);

// Audit Logs: Expire after 1 year
db.audit_logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 31536000 } // 1 year
);

// Session Data: Expire after 1 hour
db.sessions.createIndex(
  { lastActive: 1 },
  { expireAfterSeconds: 3600 } // 1 hour
);

// Temporary Documents: Expire after 7 days
db.documents.createIndex(
  { 'metadata.temporary': 1, createdAt: 1 },
  { 
    expireAfterSeconds: 604800, // 7 days
    partialFilterExpression: { 'metadata.temporary': true }
  }
);

// Rate Limit Counters: Expire after 1 minute
db.rate_limits.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 60 } // 1 minute
);
```

---

## 11. Security

### 11.1 Database Security Architecture

```
┌─────────────────────────────────────────────────────────────────
│                  DATABASE SECURITY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AUTHENTICATION                                                  │
│  ├─ PostgreSQL: IAM Authentication (Aurora)                     │
│  ├─ MongoDB: SCRAM-SHA-256 + X.509                              │
│  ├─ Redis: AUTH command + ACL                                   │
│  └─ SQL Server: Windows Authentication / SQL Auth               │
│                                                                  │
│  AUTHORIZATION                                                   │
│  ├─ Role-Based Access Control (RBAC)                            │
│  ├─ Row-Level Security (RLS)                                    │
│  ├─ Column-Level Security                                       │
│  └─ Application-Level Permissions                               │
│                                                                  │
│  ENCRYPTION                                                      │
│  ├─ At Rest: AES-256 (TDE, KMS)                                │
│  ├─ In Transit: TLS 1.2+                                        │
│  ├─ Column-Level: Application encryption                        │
│  └─ Backup Encryption: KMS                                      │
│                                                                  │
│  AUDITING                                                        │
│  ├─ Query Logging: All DDL and DML                              │
│  ├─ Access Logging: Login attempts                              │
│  ├─ Change Tracking: Data modifications                         │
│  └─ Compliance: SOC2, GDPR, PCI-DSS                             │
│                                                                  │
│  NETWORK SECURITY                                                │
│  ├─ VPC Isolation: Private subnets only                         │
│  ├─ Security Groups: Restrictive rules                          │
│  ├─ VPC Endpoints: PrivateLink for AWS services                 │
│  └─ No Public Access: Never expose to internet                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 PostgreSQL Security Configuration

```sql
-- ============================================================
-- ROLES AND PERMISSIONS
-- ============================================================

-- Create application roles
CREATE ROLE revenue_app_readonly;
CREATE ROLE revenue_app_readwrite;
CREATE ROLE revenue_app_admin;
CREATE ROLE revenue_app_billing;
CREATE ROLE revenue_app_reporting;

-- Grant permissions to readonly role
GRANT CONNECT ON DATABASE revenue_mgmt TO revenue_app_readonly;
GRANT USAGE ON SCHEMA revenue_mgmt TO revenue_app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA revenue_mgmt TO revenue_app_readonly;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA revenue_mgmt TO revenue_app_readonly;

-- Grant permissions to readwrite role
GRANT CONNECT ON DATABASE revenue_mgmt TO revenue_app_readwrite;
GRANT USAGE ON SCHEMA revenue_mgmt TO revenue_app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA revenue_mgmt TO revenue_app_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA revenue_mgmt TO revenue_app_readwrite;

-- Grant permissions to admin role
GRANT CONNECT ON DATABASE revenue_mgmt TO revenue_app_admin;
GRANT ALL ON SCHEMA revenue_mgmt TO revenue_app_admin;
GRANT ALL ON ALL TABLES IN SCHEMA revenue_mgmt TO revenue_app_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA revenue_mgmt TO revenue_app_admin;

-- Grant permissions to billing role (specific tables)
GRANT CONNECT ON DATABASE revenue_mgmt TO revenue_app_billing;
GRANT USAGE ON SCHEMA revenue_mgmt TO revenue_app_billing;
GRANT SELECT, INSERT, UPDATE ON invoices, invoice_line_items, payments, billing_schedules TO revenue_app_billing;
GRANT SELECT ON orders, order_items, accounts TO revenue_app_billing;

-- Grant permissions to reporting role
GRANT CONNECT ON DATABASE revenue_mgmt TO revenue_app_reporting;
GRANT USAGE ON SCHEMA revenue_mgmt TO revenue_app_reporting;
GRANT SELECT ON ALL TABLES IN SCHEMA revenue_mgmt TO revenue_app_reporting;

-- ============================================================
-- CREATE APPLICATION USERS
-- ============================================================

-- Create users for different services
CREATE USER nestjs_api WITH PASSWORD '***' IN ROLE revenue_app_readwrite;
CREATE USER go_billing WITH PASSWORD '***' IN ROLE revenue_app_billing;
CREATE USER python_analytics WITH PASSWORD '***' IN ROLE revenue_app_readonly;
CREATE USER reporting_user WITH PASSWORD '***' IN ROLE revenue_app_reporting;

-- ============================================================
-- ENABLE ROW-LEVEL SECURITY
-- ============================================================

-- Enable RLS on all tables
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE opportunities ENABLE ROW LEVEL SECURITY;
ALTER TABLE quotes ENABLE ROW LEVEL SECURITY;
ALTER TABLE quote_line_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoice_line_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE contracts ENABLE ROW LEVEL SECURITY;
ALTER TABLE assets ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing_schedules ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE price_books ENABLE ROW LEVEL SECURITY;
ALTER TABLE price_book_entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- ============================================================
-- ENCRYPTION AT REST
-- ============================================================

-- Enable encryption at rest (Aurora)
-- This is configured at the cluster level via AWS KMS
-- All data, backups, and snapshots are encrypted

-- ============================================================
-- ENCRYPTION IN TRANSIT
-- ============================================================

-- Force SSL connections
ALTER SYSTEM SET ssl = 'on';
ALTER SYSTEM SET ssl_cert_file = '/rdsdbdata/rds-metadata/server-cert.pem';
ALTER SYSTEM SET ssl_key_file = '/rdsdbdata/rds-metadata/server-key.pem';

-- Require SSL for all connections
ALTER SYSTEM SET rds.force_ssl = 'on';

-- ============================================================
-- AUDIT LOGGING
-- ============================================================

-- Enable pgAudit extension
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure pgAudit
ALTER SYSTEM SET pgaudit.log = 'write, ddl';
ALTER SYSTEM SET pgaudit.log_catalog = 'off';
ALTER SYSTEM SET pgaudit.log_parameter = 'on';
ALTER SYSTEM SET pgaudit.log_statement_once = 'off';

-- Reload configuration
SELECT pg_reload_conf();

-- ============================================================
-- PASSWORD POLICY
-- ============================================================

-- Set password encryption method
ALTER SYSTEM SET password_encryption = 'scram-sha-256';

-- Set password expiration (90 days)
ALTER USER nestjs_api VALID UNTIL '2026-09-22';
ALTER USER go_billing VALID UNTIL '2026-09-22';
ALTER USER python_analytics VALID UNTIL '2026-09-22';

-- ============================================================
-- CONNECTION LIMITS
-- ============================================================

-- Set connection limits per user
ALTER USER nestjs_api CONNECTION LIMIT 50;
ALTER USER go_billing CONNECTION LIMIT 20;
ALTER USER python_analytics CONNECTION LIMIT 10;
ALTER USER reporting_user CONNECTION LIMIT 5;
```

### 11.3 MongoDB Security Configuration

```javascript
// MongoDB Security Configuration

// 1. Enable Authentication
// mongod --auth --config /etc/mongod.conf

// 2. Create Admin User
use admin
db.createUser({
  user: 'admin',
  pwd: '***',
  roles: [
    { role: 'root', db: 'admin' }
  ]
});

// 3. Create Application Users
use revenue_mgmt

// Read-only user
db.createUser({
  user: 'analytics_user',
  pwd: '***',
  roles: [
    { role: 'read', db: 'revenue_mgmt' }
  ]
});

// Read-write user
db.createUser({
  user: 'app_user',
  pwd: '***',
  roles: [
    { role: 'readWrite', db: 'revenue_mgmt' }
  ]
});

// 4. Enable TLS/SSL
// mongod --sslMode requireSSL --sslPEMKeyFile /path/to/cert.pem

// 5. Enable Audit Logging
// mongod --auditLog destination=file --auditLogFormat json --auditLogPath /var/log/mongodb/audit.log

// 6. Network Security
// Bind to specific interfaces
// mongod --bind_ip 10.0.0.0/16

// 7. Encryption at Rest
// Enable via MongoDB Enterprise or Atlas
// db.adminCommand({
//   setParameter: 1,
//   enableEncryption: true,
//   encryptionKeyFile: '/path/to/keyfile'
// });

// 8. Role-Based Access Control
// Create custom roles
db.createRole({
  role: 'billing_role',
  privileges: [
    {
      resource: { db: 'revenue_mgmt', collection: 'invoices' },
      actions: ['find', 'insert', 'update']
    },
    {
      resource: { db: 'revenue_mgmt', collection: 'payments' },
      actions: ['find', 'insert', 'update']
    }
  ],
  roles: []
});
```

### 11.4 Redis Security Configuration

```javascript
// Redis Security Configuration

// 1. Enable Authentication
// redis-server --requirepass your-strong-password

// 2. ACL (Access Control Lists) - Redis 6+
// Create users with specific permissions
ACL SETUSER nestjs_api on >password ~session:* ~pricing:* +@read +@write +@connection
ACL SETUSER go_billing on >password ~lock:* ~queue:* +@read +@write
ACL SETUSER analytics on >password ~metrics:* ~dashboard:* +@read

// 3. Enable TLS
// redis-server --tls-port 6379 --port 0 --tls-cert-file /path/to/cert.pem --tls-key-file /path/to/key.pem

// 4. Rename Dangerous Commands
// redis-server --rename-command FLUSHDB "" --rename-command FLUSHALL "" --rename-command CONFIG "b840fc02d524045429941cc15f59e41cb7be6c52"

// 5. Network Security
// Bind to specific interfaces
// redis-server --bind 10.0.0.0/16

// 6. Disable Dangerous Commands
// CONFIG SET protected-mode yes

// 7. Set Memory Limit
// CONFIG SET maxmemory 2gb
// CONFIG SET maxmemory-policy allkeys-lru
```

---

## 12. RLS Policy

### 12.1 Row-Level Security (RLS) Policies

```sql
-- ============================================================
-- ROW-LEVEL SECURITY POLICIES
-- ============================================================

-- RLS ensures users can only access data they're authorized to see
-- Policies are evaluated for every query

-- ============================================================
-- 1. ACCOUNTS TABLE
-- ============================================================

-- Policy: Users can only see accounts in their tenant
CREATE POLICY accounts_tenant_isolation ON accounts
    FOR ALL
    USING (
        -- Get tenant_id from current session
        EXISTS (
            SELECT 1 FROM users 
            WHERE users.id = current_setting('app.current_user_id')::uuid
            AND users.tenant_id = accounts.tenant_id
        )
    );

-- Policy: Sales reps can only see their own accounts
CREATE POLICY accounts_owner_restriction ON accounts
    FOR SELECT
    USING (
        current_setting('app.current_user_role') = 'admin'
        OR current_setting('app.current_user_role') = 'sales_manager'
        OR owner_id = current_setting('app.current_user_id')::uuid
    );

-- ============================================================
-- 2. OPPORTUNITIES TABLE
-- ============================================================

-- Policy: Users can only see opportunities in their tenant
CREATE POLICY opportunities_tenant_isolation ON opportunities
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts 
            WHERE accounts.id = opportunities.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- Policy: Sales reps can only see their own opportunities
CREATE POLICY opportunities_owner_restriction ON opportunities
    FOR SELECT
    USING (
        current_setting('app.current_user_role') IN ('admin', 'sales_manager')
        OR owner_id = current_setting('app.current_user_id')::uuid
    );

-- Policy: Only admins and sales managers can delete opportunities
CREATE POLICY opportunities_delete_restriction ON opportunities
    FOR DELETE
    USING (
        current_setting('app.current_user_role') IN ('admin', 'sales_manager')
    );

-- ============================================================
-- 3. QUOTES TABLE
-- ============================================================

-- Policy: Users can only see quotes for their opportunities
CREATE POLICY quotes_opportunity_access ON quotes
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM opportunities
            WHERE opportunities.id = quotes.opportunity_id
            AND opportunities.owner_id = current_setting('app.current_user_id')::uuid
        )
        OR current_setting('app.current_user_role') IN ('admin', 'sales_manager')
    );

-- Policy: Only approved quotes can be converted to orders
CREATE POLICY quotes_approval_required ON quotes
    FOR UPDATE
    USING (
        status = 'approved'
        OR current_setting('app.current_user_role') IN ('admin', 'sales_manager')
    );

-- ============================================================
-- 4. ORDERS TABLE
-- ============================================================

-- Policy: Users can only see orders for their accounts
CREATE POLICY orders_account_access ON orders
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts
            WHERE accounts.id = orders.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- Policy: Only billing specialists can modify order status
CREATE POLICY orders_status_modification ON orders
    FOR UPDATE
    USING (
        current_setting('app.current_user_role') IN ('admin', 'billing_specialist')
        OR (
            current_setting('app.current_user_role') = 'sales_manager'
            AND status IN ('pending', 'cancelled')
        )
    );

-- ============================================================
-- 5. INVOICES TABLE
-- ============================================================

-- Policy: Users can only see invoices for their accounts
CREATE POLICY invoices_account_access ON invoices
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts
            WHERE accounts.id = invoices.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- Policy: Only billing specialists can create/modify invoices
CREATE POLICY invoices_billing_access ON invoices
    FOR INSERT
    WITH CHECK (
        current_setting('app.current_user_role') IN ('admin', 'billing_specialist')
    );

CREATE POLICY invoices_billing_update ON invoices
    FOR UPDATE
    USING (
        current_setting('app.current_user_role') IN ('admin', 'billing_specialist')
    );

-- Policy: Only admins can delete invoices
CREATE POLICY invoices_delete_restriction ON invoices
    FOR DELETE
    USING (
        current_setting('app.current_user_role') = 'admin'
    );

-- ============================================================
-- 6. CONTRACTS TABLE
-- ============================================================

-- Policy: Users can only see contracts for their accounts
CREATE POLICY contracts_account_access ON contracts
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts
            WHERE accounts.id = contracts.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- Policy: Only contract managers can modify contracts
CREATE POLICY contracts_manager_access ON contracts
    FOR UPDATE
    USING (
        current_setting('app.current_user_role') IN ('admin', 'contract_manager')
    );

-- ============================================================
-- 7. ASSETS TABLE
-- ============================================================

-- Policy: Users can only see assets for their accounts
CREATE POLICY assets_account_access ON assets
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts
            WHERE accounts.id = assets.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- ============================================================
-- 8. PAYMENTS TABLE
-- ============================================================

-- Policy: Users can only see payments for their accounts
CREATE POLICY payments_account_access ON payments
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM accounts
            WHERE accounts.id = payments.account_id
            AND accounts.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
    );

-- Policy: Only billing specialists can record payments
CREATE POLICY payments_billing_access ON payments
    FOR INSERT
    WITH CHECK (
        current_setting('app.current_user_role') IN ('admin', 'billing_specialist')
    );

-- ============================================================
-- 9. USERS TABLE
-- ============================================================

-- Policy: Users can only see users in their tenant
CREATE POLICY users_tenant_isolation ON users
    FOR ALL
    USING (
        tenant_id = (
            SELECT tenant_id FROM users 
            WHERE users.id = current_setting('app.current_user_id')::uuid
        )
    );

-- Policy: Users can only modify their own profile (except admins)
CREATE POLICY users_self_modification ON users
    FOR UPDATE
    USING (
        id = current_setting('app.current_user_id')::uuid
        OR current_setting('app.current_user_role') = 'admin'
    );

-- ============================================================
-- 10. AUDIT LOGS TABLE
-- ============================================================

-- Policy: Users can only see audit logs for their tenant
CREATE POLICY audit_logs_tenant_isolation ON audit_logs
    FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM users
            WHERE users.id = audit_logs.user_id
            AND users.tenant_id = (
                SELECT tenant_id FROM users 
                WHERE users.id = current_setting('app.current_user_id')::uuid
            )
        )
        OR current_setting('app.current_user_role') = 'admin'
    );

-- Policy: Only system can insert audit logs
CREATE POLICY audit_logs_system_insert ON audit_logs
    FOR INSERT
    WITH CHECK (
        current_setting('app.current_user_role') = 'system'
    );

-- Policy: No one can delete audit logs
CREATE POLICY audit_logs_no_delete ON audit_logs
    FOR DELETE
    USING (false);

-- ============================================================
-- RLS HELPER FUNCTIONS
-- ============================================================

-- Function to set current user context
CREATE OR REPLACE FUNCTION set_current_user_context(
    p_user_id UUID,
    p_user_role VARCHAR,
    p_tenant_id UUID
)
RETURNS void AS $$
BEGIN
    PERFORM set_config('app.current_user_id', p_user_id::text, false);
    PERFORM set_config('app.current_user_role', p_user_role, false);
    PERFORM set_config('app.current_tenant_id', p_tenant_id::text, false);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Function to get current user info
CREATE OR REPLACE FUNCTION get_current_user_info()
RETURNS TABLE(
    user_id UUID,
    user_role VARCHAR,
    tenant_id UUID
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        current_setting('app.current_user_id')::uuid AS user_id,
        current_setting('app.current_user_role') AS user_role,
        current_setting('app.current_tenant_id')::uuid AS tenant_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- ============================================================
-- RLS TESTING
-- ============================================================

-- Test RLS policies
-- Set user context
SELECT set_current_user_context(
    'user-uuid-here'::uuid,
    'sales_rep',
    'tenant-uuid-here'::uuid
);

-- Query should only return authorized records
SELECT * FROM opportunities;
SELECT * FROM invoices;
SELECT * FROM contracts;

-- Clear user context
SELECT set_config('app.current_user_id', NULL, false);
SELECT set_config('app.current_user_role', NULL, false);
SELECT set_config('app.current_tenant_id', NULL, false);
```

### 12.2 RLS Best Practices

```yaml
# RLS Best Practices

1. Always Enable RLS
   - Enable on all tables with sensitive data
   - Test thoroughly before production deployment
   - Monitor performance impact

2. Use Session Variables
   - Set user context at connection start
   - Use set_config() for session variables
   - Clear context on connection close

3. Keep Policies Simple
   - Avoid complex subqueries in policies
   - Use indexed columns in policy conditions
   - Test policy performance

4. Combine with Application Logic
   - RLS is defense-in-depth
   - Still validate permissions in application
   - Don't rely solely on RLS

5. Monitor RLS Performance
   - Check query plans for policy evaluation
   - Monitor slow queries
   - Optimize policy conditions

6. Document Policies
   - Document each policy's purpose
   - Review policies regularly
   - Update when business rules change

7. Test RLS Thoroughly
   - Test with different user roles
   - Test edge cases
   - Verify no data leakage
```

---

## ภาคผนวก

### ภาคผนวก A: Database Migration Strategy

```yaml
# Migration Strategy

Tools:
  - Flyway (PostgreSQL)
  - Liquibase (Multi-database)
  - Prisma Migrate (TypeScript/Node.js)
  - MongoDB Migrate (MongoDB)

Process:
  1. Create migration file
  2. Test migration in development
  3. Apply to staging
  4. Validate in staging
  5. Apply to production (during maintenance window)
  6. Monitor for issues
  7. Rollback plan if needed

Best Practices:
  - Always provide rollback scripts
  - Test migrations with production data volume
  - Use feature flags for schema changes
  - Monitor migration performance
  - Document all migrations
```

### ภาคผนวก B: Performance Monitoring Queries

```sql
-- ============================================================
-- PERFORMANCE MONITORING QUERIES
-- ============================================================

-- 1. Slow Query Log
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- 2. Index Usage Statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- 3. Table Bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- 4. Connection Statistics
SELECT 
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit
FROM pg_stat_database;

-- 5. Lock Statistics
SELECT 
    pid,
    usename,
    datname,
    state,
    query,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL;

-- 6. Cache Hit Ratio
SELECT 
    sum(blks_hit) * 100 / nullif(sum(blks_hit) + sum(blks_read),0) AS cache_hit_ratio
FROM pg_stat_database;

-- 7. Replication Lag
SELECT 
    client_addr,
    state,
    sent_lag,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

### ภาคผนวก C: Database Optimization Checklist

```yaml
Pre-Production Checklist:
  Schema:
    - [ ] All tables have primary keys
    - [ ] Foreign keys are properly indexed
    - [ ] Check constraints are in place
    - [ ] Default values are set
    - [ ] NOT NULL constraints applied where needed
  
  Indexes:
    - [ ] Foreign key indexes created
    - [ ] Composite indexes for common queries
    - [ ] Partial indexes for filtered queries
    - [ ] Full-text search indexes (if needed)
    - [ ] JSONB indexes (if using JSONB)
  
  Security:
    - [ ] RLS enabled on sensitive tables
    - [ ] Roles and permissions configured
    - [ ] Encryption at rest enabled
    - [ ] TLS enabled for connections
    - [ ] Audit logging enabled
  
  Performance:
    - [ ] Query plans reviewed
    - [ ] Slow queries optimized
    - [ ] Connection pooling configured
    - [ ] Read replicas configured (if needed)
    - [ ] Partitioning implemented (if needed)
  
  Backup:
    - [ ] Automated backups configured
    - [ ] Backup retention policy set
    - [ ] Point-in-time recovery enabled
    - [ ] Cross-region backups (if needed)
    - [ ] Backup verification tested
  
  Monitoring:
    - [ ] Performance metrics configured
    - [ ] Alert thresholds set
    - [ ] Slow query logging enabled
    - [ ] Connection monitoring enabled
    - [ ] Replication lag monitoring (if applicable)
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการออกแบบฐานข้อมูลสำหรับระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** Database Architecture Team
