# System Architecture Document (SAD)
## ระบบจัดการ Revenue Management

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  

---

## 📋 สารบัญ
1. [บทนำ](#1-บทนำ)
2. [ภาพรวมสถาปัตยกรรมระบบ](#2-ภาพรวมสถาปัตยกรรมระบบ)
3. [สถาปัตยกรรมเชิงตรรกะ](#3-สถาปัตยกรรมเชิงตรรกะ)
4. [สถาปัตยกรรมข้อมูล](#4-สถาปัตยกรรมข้อมูล)
5. [สถาปัตยกรรมส่วนต่อประสาน](#5-สถาปัตยกรรมส่วนต่อประสาน)
6. [สถาปัตยกรรมความปลอดภัย](#6-สถาปัตยกรรมความปลอดภัย)
7. [สถาปัตยกรรมโครงสร้างพื้นฐาน](#7-สถาปัตยกรรมโครงสร้างพื้นฐาน)
8. [สถาปัตยกรรม Integration](#8-สถาปัตยกรรม-integration)
9. [สถาปัตยกรรม Deployment](#9-สถาปัตยกรรม-deployment)
10. [การจัดการ Performance และ Scalability](#10-การจัดการ-performance-และ-scalability)
11. [การสำรองข้อมูลและ Disaster Recovery](#11-การสำรองข้อมูลและ-disaster-recovery)
12. [Monitoring และ Logging](#12-monitoring-และ-logging)

---

## 1. บทนำ

### 1.1 วัตถุประสงค์ของเอกสาร
เอกสาร System Architecture Document (SAD) ฉบับนี้มีวัตถุประสงค์เพื่ออธิบายสถาปัตยกรรมทางเทคนิคของระบบจัดการ Revenue Management อย่างละเอียด ครอบคลุมการออกแบบระบบ โครงสร้างพื้นฐาน การจัดการข้อมูล การเชื่อมต่อระบบภายนอก และมาตรฐานด้านความปลอดภัย

### 1.2 ขอบเขต
เอกสารนี้ครอบคลุม:
- สถาปัตยกรรมระบบทั้งหมด
- การออกแบบ Components และ Modules
- Data Model และ Database Design
- Integration Patterns และ APIs
- Security Architecture
- Infrastructure และ Deployment Strategy
- Performance และ Scalability Considerations
- Disaster Recovery และ Business Continuity

### 1.4 คำจำกัดความและคำย่อ

| คำย่อ | ความหมาย |
|-------|----------|
| SAD | System Architecture Document |
| PRD | Product Requirements Document |
| API | Application Programming Interface |
| CPQ | Configure, Price, Quote |
| CRM | Customer Relationship Management |
| ERP | Enterprise Resource Planning |
| SSO | Single Sign-On |
| MFA | Multi-Factor Authentication |
| RBAC | Role-Based Access Control |
| SLA | Service Level Agreement |
| RTO | Recovery Time Objective |
| RPO | Recovery Point Objective |

---

## 2. ภาพรวมสถาปัตยกรรมระบบ

### 2.1 สถาปัตยกรรมระดับองค์กร (Enterprise Architecture)

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Web    │  │  Mobile  │  │  Tablet  │  │   API    │       │
│  │  Portal  │  │   App    │  │   App    │  │ Gateway  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────       │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER (Agentforce)                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BUSINESS LOGIC TIER                          │  │
│  │  ┌────────┐ ─────┐ ──────────┐ ────────┐ ────────┐ │  │
│  │  │Catalog │ │ CPQ │ │Contracts │ │ Orders │ │ Assets │ │  │
│  │  └────────┘ └─────┘ └────────── └────────┘ └────────┘ │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │                  Billing Engine                     │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │         AI Agents (Agentforce Intelligence)         │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    DATA & TRUST LAYER                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Salesforce │  │   Platform   │  │   Security   │         │
│  │   Database   │  │   Services   │  │   Services   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                 INTEGRATION LAYER (MuleSoft)                     │
│  ┌──────────  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │    ERP   │  │ Payment  │  │E-Signature│  │ External │       │
│  │  Systems │  │ Gateways │  │ Services │  │   APIs   │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 สถาปัตยกรรมแบบ Microservices

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Lightweight)                     │
│                     Rate Limiting | Auth | Routing               │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   CPQ Service │     │  Billing      │     │  Contract     │
│               │     │  Service      │     │  Service      │
│ - Configure   │     │               │     │               │
│ - Price       │     │ - Invoicing   │     │ - Authoring   │
│ - Quote       │     │ - Rating      │     │ - Negotiation │
│ - Validation  │     │ - Scheduling  │     │ - Approval    │
└───────────────┘     └───────────────┘     └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
                    ┌───────────────┐
                    │   Shared      │
                    │   Services    │
                    │               │
                    │ - Notification│
                    │ - Document    │
                    │ - Workflow    │
                    │ - Analytics   │
                    └───────────────┘
```

### 2.3 เทคโนโลยีหลัก (Technology Stack)

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend** | Salesforce Lightning Web Components (LWC) | Latest | UI Components |
| | Visualforce | Latest | Legacy UI Support |
| | JavaScript/TypeScript | ES2020+ | Client-side Logic |
| **Backend** | Apex | Latest | Server-side Logic |
| | Salesforce Flows | Latest | Process Automation |
| | Batch Apex | Latest | Batch Processing |
| **Database** | Salesforce Multi-Tenant Database | N/A | Primary Data Store |
| | Big Objects | Latest | Large Data Volume |
| | Platform Cache | Latest | Caching Layer |
| **Integration** | MuleSoft Anypoint Platform | Latest | Integration Hub |
| | Salesforce APIs | v58.0+ | System Integration |
| | Platform Events | Latest | Event-Driven Architecture |
| **AI/ML** | Einstein AI | Latest | Predictive Analytics |
| | Agentforce | Latest | AI Agents |
| **DevOps** | Salesforce DevOps Center | Latest | CI/CD |
| | Git | 2.x | Version Control |
| | Jenkins/GitHub Actions | Latest | Automation |

---

## 3. สถาปัตยกรรมเชิงตรรกะ

### 3.1 Logical Components

#### 3.1.1 Presentation Tier Components

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION TIER                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           Lightning Web Components (LWC)            │    │
│  │                                                     │    │
│  │  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │ Opportunity  │  │    Quote     │               │    │
│  │  │  Components  │  │  Components  │               │    │
│  │  └──────────────┘  └──────────────┘               │    │
│  │  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │    Order     │  │   Billing    │               │    │
│  │  │  Components  │  │  Components  │               │    │
│  │  └──────────────┘  └──────────────┘               │    │
│  │  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │   Contract   │  │   Asset      │               │    │
│  │  │  Components  │  │  Components  │               │    │
│  │  └──────────────┘  └──────────────┘               │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Visualforce Pages (Legacy)             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Salesforce Mobile App                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.1.2 Application Tier Components

```
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION TIER                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              SERVICE LAYER (Apex Classes)           │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Catalog Service                             │ │    │
│  │  │  - ProductCatalogService.cls                 │ │    │
│  │  │  - PriceBookService.cls                      │ │    │
│  │  │  - ProductHierarchyService.cls               │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  CPQ Service                                 │ │    │
│  │  │  - QuoteService.cls                          │ │    │
│  │  │  - QuoteLineService.cls                      │ │    │
│  │  │  - PricingEngineService.cls                  │ │    │
│  │  │  - DiscountApprovalService.cls               │ │    │
│  │  │  - ConfigurationValidatorService.cls         │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Contract Service                            │ │    │
│  │  │  - ContractService.cls                       │ │    │
│  │  │  - ContractTemplateService.cls               │ │    │
│  │  │  - ClauseLibraryService.cls                  │ │    │
│  │  │  - ContractApprovalService.cls               │ │    │
│  │  │  - eSignatureService.cls                     │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Order Service                               │ │    │
│  │  │  - OrderService.cls                          │ │    │
│  │  │  - OrderItemService.cls                      │ │    │
│  │  │  - OrderActivationService.cls                │ │    │
│  │  │  - OrderDecompositionService.cls             │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Billing Service                             │ │    │
│  │  │  - BillingService.cls                        │ │    │
│  │  │  - InvoiceService.cls                        │ │    │
│  │  │  - InvoiceBatchService.cls                   │ │    │
│  │  │  - BillingScheduleService.cls                │ │    │
│  │  │  - RatingEngineService.cls                   │ │    │
│  │  │  - PaymentService.cls                        │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Asset Service                               │ │    │
│  │  │  - AssetService.cls                          │ │    │
│  │  │  - AssetLifecycleService.cls                 │ │    │
│  │  │  - EntitlementService.cls                    │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────────┐ │    │
│  │  │  Subscription Service                        │ │    │
│  │  │  - SubscriptionService.cls                   │ │    │
│  │  │  - UsageService.cls                          │ │    │
│  │  │  - ConsumptionService.cls                    │ │    │
│  │  └──────────────────────────────────────────────┘ │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            PROCESS AUTOMATION LAYER                 │    │
│  │                                                     │    │
│  │  - Salesforce Flows (Record-Triggered, Scheduled)  │    │
│  │  - Process Builder (Legacy)                        │    │
│  │  - Workflow Rules (Legacy)                         │    │
│  │  - Approval Processes                              │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              BATCH PROCESSING LAYER                 │    │
│  │                                                     │    │
│  │  - Batch Apex Classes                              │    │
│  │  - Scheduled Apex Jobs                             │    │
│  │  - Queueable Apex Chains                           │    │
│  │  - Future Methods (Async)                          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              AI & INTELLIGENCE LAYER                │    │
│  │                                                     │    │
│  │  - Agentforce AI Agents                            │    │
│  │  - Einstein Prediction Builder                     │    │
│  │  - Einstein Discovery                              │    │
│  │  - AI-Powered Insights                             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.1.3 Data Tier Components

```
┌─────────────────────────────────────────────────────────────┐
│                      DATA TIER                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           STANDARD SALESFORCE OBJECTS               │    │
│  │                                                     │    │
│  │  - Account, Contact, User                          │    │
│  │  - Opportunity, OpportunityLineItem                │    │
│  │  - Quote, QuoteLineItem                            │    │
│  │  - Order, OrderItem                                │    │
│  │  - Contract                                        │    │
│  │  - Asset                                           │    │
│  │  - Invoice, InvoiceLine                            │    │
│  │  - Payment, PaymentLine                            │    │
│  │  - Product2, Pricebook2, PricebookEntry            │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              CUSTOM OBJECTS                         │    │
│  │                                                     │    │
│  │  Revenue Management Core:                          │    │
│  │  - Billing_Schedule__c                             │    │
│  │  - Invoice_Batch_Run__c                            │    │
│  │  - Usage_Record__c                                 │    │
│  │  - Rating_Log__c                                   │    │
│  │  - Entitlement__c                                  │    │
│  │  - Subscription__c                                 │    │
│  │                                                     │    │
│  │  Contract Management:                              │    │
│  │  - Contract_Template__c                            │    │
│  │  - Contract_Clause__c                              │    │
│  │  - Contract_Approval__c                            │    │
│  │  - Contract_Obligation__c                          │    │
│  │                                                     │    │
│  │  CPQ Configuration:                                │    │
│  │  - Product_Configuration__c                        │    │
│  │  - Pricing_Rule__c                                 │    │
│  │  - Discount_Approval__c                            │    │
│  │  - Quote_Validation_Log__c                         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              BIG OBJECTS (Historical Data)          │    │
│  │                                                     │    │
│  │  - Usage_History__b                                │    │
│  │  - Invoice_Archive__b                              │    │
│  │  - Audit_Trail__b                                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              PLATFORM CACHE                         │    │
│  │                                                     │    │
│  │  - Product Catalog Cache                           │    │
│  │  - Price Book Cache                                │    │
│  │  - User Preferences Cache                          │    │
│  │  - Configuration Cache                             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           EXTERNAL DATA SOURCES                     │    │
│  │                                                     │    │
│  │  - ERP System (via External Objects)               │    │
│  │  - Payment Gateway (via Named Credentials)         │    │
│  │  - Document Storage (Files, ContentVersion)        │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow Architecture

#### 3.2.1 Opportunity to Quote Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Opportunity │────▶│    Quote    │────▶│  QuoteLine  │
│   Record    │     │   Record    │     │   Records   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │   Product   │
                  │ Selection   │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │   Pricing   │
                  │  Engine     │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │  Discount   │
                  │  Approval   │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │   Quote     │
                  │  Document   │
                  └─────────────┘
```

#### 3.2.2 Quote to Order Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Quote    │────▶│   Order     │────▶│  OrderItem  │
│  (Approved) │     │   Record    │     │   Records   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │   Asset     │
                  │ Generation  │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │ Entitlement │
                  │ Generation  │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │  Contract   │
                  │  Creation   │
                  └─────────────┘
```

#### 3.2.3 Order to Cash Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Order    │────▶│  Billing    │────▶│   Invoice   │
│  (Active)   │     │  Schedule   │     │ Generation  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │                   │
                           ▼                   ▼
                  ┌─────────────┐     ┌─────────────┐
                  │   Usage     │     │   Invoice   │
                  │ Processing  │     │   Delivery  │
                  └─────────────┘     └─────────────┘
                                             │
                                             ▼
                                    ┌─────────────┐
                                    │  Payment    │
                                    │ Processing  │
                                    └─────────────┘
                                             │
                                             ▼
                                    ┌─────────────┐
                                    │   Revenue   │
                                    │ Recognition │
                                    └─────────────┘
```

---

## 4. สถาปัตยกรรมข้อมูล

### 4.1 Entity Relationship Diagram (ERD) - Core Objects

```
┌──────────────────┐       ┌──────────────────┐
│     Account      │       │  Pricebook2      │
│                  │       │                  │
│  Id              │       │  Id              │
│  Name            │       │  Name            │
│  Type            │       │  IsStandard      │
│  Industry        │       │  CurrencyIsoCode │
└─────────────────┘       └──────────────────┘
         │
         │ 1:N
         ▼
┌──────────────────┐       ┌──────────────────┐
│   Opportunity    │       │    Product2      │
│                  │       │                  │
│  Id              │       │  Id              │
│  Name            │       │  Name            │
│  AccountId       │──────▶│  ProductCode     │
│  CloseDate       │       │  IsActive        │
│  StageName       │       │  Family          │
│  Amount          │       └──────────────────┘
│  Probability     │
│  Pricebook2Id    │──┐
└─────────────────┘  │
         │            │
         │ 1:N        │
         ▼            │
┌──────────────────┐  │
│      Quote       │  │
│                  │  │
│  Id              │  │
│  Name            │  │
│  OpportunityId   │──┘
│  Pricebook2Id    │──┐
│  ExpirationDate  │  │
│  Status          │  │
└────────┬─────────┘  │
         │            │
         │ 1:N        │
         ▼            │
┌──────────────────┐  │
│   QuoteLineItem  │  │
│                  │  │
│  Id              │  │
│  QuoteId         │  │
│  Product2Id      │──┘
│  Quantity        │
│  UnitPrice       │
│  Discount        │
│  TotalPrice      │
└──────────────────┘
```

### 4.2 Data Model - Custom Objects

#### 4.2.1 Billing_Schedule__c

```apex
Object: Billing_Schedule__c
Purpose: กำหนดตารางการเรียกเก็บเงิน

Fields:
├─ Name (Text, Auto-Number)
├─ Order__c (Lookup to Order)
├─ Invoice_Run_Schedule__c (Lookup to Invoice_Batch_Run__c)
├─ Billing_Type__c (Picklist: One-time, Recurring, Usage-based)
├─ Billing_Frequency__c (Picklist: Monthly, Quarterly, Annual)
├─ Billing_Date__c (Date)
├─ Start_Date__c (Date)
├─ End_Date__c (Date)
├─ Amount__c (Currency)
├─ Status__c (Picklist: Pending, Processed, Invoiced, Cancelled)
├─ Invoice__c (Lookup to Invoice)
├─ Description__c (Long Text)
└─ CreatedDate, LastModifiedDate (System)

Relationships:
├─ Order (N:1)
├─ Invoice_Batch_Run__c (N:1)
└─ Invoice (N:1)
```

#### 4.2.2 Invoice_Batch_Run__c

```apex
Object: Invoice_Batch_Run__c
Purpose: บันทึกการรันใบแจ้งหนี้แบบ Batch

Fields:
├─ Name (Text, Auto-Number)
├─ Batch_Number__c (Text, Unique)
├─ Scheduled_Date__c (Date/Time)
├─ Start_Time__c (Date/Time)
├─ End_Time__c (Date/Time)
├─ Status__c (Picklist: Scheduled, Running, Completed, Failed)
├─ Total_Invoices__c (Number)
├─ Successful_Invoices__c (Number)
├─ Failed_Invoices__c (Number)
├─ Total_Amount__c (Currency)
├─ Error_Log__c (Long Text)
├─ Processed_By__c (Lookup to User)
└─ CreatedDate, LastModifiedDate (System)

Relationships:
├─ Billing_Schedule__c (1:N)
└─ User (N:1)
```

#### 4.2.3 Usage_Record__c

```apex
Object: Usage_Record__c
Purpose: บันทึกข้อมูลการใช้งาน

Fields:
├─ Name (Text, Auto-Number)
├─ Subscription__c (Lookup to Subscription__c)
├─ Asset__c (Lookup to Asset)
├─ Usage_Date__c (Date)
├─ Usage_Start_Time__c (Date/Time)
├─ Usage_End_Time__c (Date/Time)
├─ Quantity__c (Number)
├─ Unit_of_Measure__c (Picklist: Hours, GB, Units, Requests)
├─ Rate__c (Currency)
├─ Calculated_Amount__c (Currency)
├─ Status__c (Picklist: Pending, Rated, Invoiced)
├─ Rating_Log__c (Lookup to Rating_Log__c)
└─ CreatedDate, LastModifiedDate (System)

Relationships:
├─ Subscription__c (N:1)
├─ Asset (N:1)
└─ Rating_Log__c (N:1)
```

#### 4.2.4 Contract_Template__c

```apex
Object: Contract_Template__c
Purpose: เทมเพลตสัญญา

Fields:
├─ Name (Text)
├─ Template_Code__c (Text, Unique)
├─ Description__c (Long Text)
├─ Category__c (Picklist: Sales, Service, Partnership, NDA)
├─ Version__c (Text)
├─ Is_Active__c (Checkbox)
├─ Document_Template_Id__c (Text) - MS 365 Template ID
├─ Approval_Required__c (Checkbox)
├─ Default_Term_Months__c (Number)
├─ Auto_Renewal__c (Checkbox)
├─ Created_By__c (Lookup to User)
└─ CreatedDate, LastModifiedDate (System)

Relationships:
├─ User (N:1)
└─ Contract_Clause__c (1:N)
```

### 4.3 Data Volume Estimates

| Object | Estimated Records (Year 1) | Growth Rate/Year | Storage Estimate |
|--------|---------------------------|------------------|------------------|
| Account | 10,000 | 20% | 50 MB |
| Opportunity | 50,000 | 30% | 250 MB |
| Quote | 100,000 | 40% | 500 MB |
| QuoteLineItem | 500,000 | 50% | 2.5 GB |
| Order | 75,000 | 35% | 375 MB |
| OrderItem | 375,000 | 45% | 1.9 GB |
| Invoice | 200,000 | 40% | 1 GB |
| InvoiceLine | 1,000,000 | 50% | 5 GB |
| Billing_Schedule__c | 150,000 | 40% | 750 MB |
| Usage_Record__c | 5,000,000 | 60% | 25 GB (Big Object) |
| Contract__c | 25,000 | 25% | 125 MB |
| Asset | 200,000 | 30% | 1 GB |
| **Total** | **~7.7M** | **-** | **~38 GB** |

### 4.4 Data Archival Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA LIFECYCLE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ACTIVE DATA (0-12 months)                                  │
│  ├─ Stored in Standard Objects                              │
│  ├─ Full Indexing                                           │
│  ├─ Real-time Access                                        │
│  └─ Daily Backup                                            │
│                                                              │
│  WARM DATA (13-24 months)                                   │
│  ├─ Archived to Big Objects                                 │
│  ├─ Partial Indexing                                        │
│  ├─ Query via Async SOQL                                    │
│  └─ Weekly Backup                                             │
│                                                              │
│  COLD DATA (25+ months)                                     │
│  ├─ Exported to External Storage (AWS S3/Azure Blob)        │
│  ├─ Deleted from Salesforce                                   │
│  ├─ Access via External Objects                               │
│  └─ Monthly Backup                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Archival Jobs:
├─ Daily: Archive records older than 12 months
├─ Weekly: Export to external storage
├─ Monthly: Purge exported records
└─ Quarterly: Validate archival integrity
```

### 4.5 Data Security & Compliance

#### 4.5.1 Data Classification

```
┌─────────────────────────────────────────────────────────────┐
│                 DATA CLASSIFICATION                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PUBLIC                                                     │
│  ├─ Product Catalog                                         │
│  ├─ Price Books (Public)                                    │
│  └─ Marketing Materials                                     │
│                                                              │
│  INTERNAL                                                   │
│  ├─ Opportunity Data                                        │
│  ├─ Quote Information                                       │
│  ├─ Contract Terms                                          │
│  └─ Billing Schedules                                       │
│                                                              │
│  CONFIDENTIAL                                               │
│  ├─ Customer PII                                            │
│  ├─ Payment Information                                     │
│  ├─ Pricing Discounts                                       │
│  └─ Revenue Data                                            │
│                                                              │
│  RESTRICTED                                                 │
│  ├─ Credit Card Data (Tokenized)                            │
│  ├─ Bank Account Details                                    │
│  └─ Tax Identification Numbers                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 4.5.2 Data Retention Policy

| Data Type | Retention Period | Legal Hold | Deletion Method |
|-----------|-----------------|------------|-----------------|
| Financial Records | 7 years | Yes | Secure Delete |
| Contracts | 10 years | Yes | Archive + Secure Delete |
| Invoices | 7 years | Yes | Archive + Secure Delete |
| Usage Records | 3 years | No | Automated Purge |
| Audit Logs | 5 years | Yes | Archive |
| Customer PII | As per consent | Yes | Anonymize |
| System Logs | 1 year | No | Automated Purge |

---

## 5. สถาปัตยกรรมส่วนต่อประสาน

### 5.1 API Architecture

#### 5.1.1 API Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Salesforce API Gateway                   │  │
│  │                                                       │  │
│  │  - REST API (v58.0+)                                 │  │
│  │  - SOAP API (Enterprise, Partner)                    │  │
│  │  - Bulk API 2.0                                      │  │
│  │  - Streaming API                                     │  │
│  │  - Metadata API                                      │  │
│  │  - Tooling API                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              MuleSoft API Gateway                     │  │
│  │                                                       │  │
│  │  - External API Exposure                              │  │
│  │  - Rate Limiting & Throttling                         │  │
│  │  - API Analytics                                      │  │
│  │  - OAuth 2.0 / JWT Authentication                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 5.1.2 REST API Endpoints

```yaml
# Opportunity APIs
/opportunities:
  get:
    summary: List opportunities
    parameters:
      - accountId
      - stage
      - closeDate
  post:
    summary: Create opportunity
    body: OpportunityCreateRequest

/opportunities/{id}:
  get:
    summary: Get opportunity details
  patch:
    summary: Update opportunity
  delete:
    summary: Delete opportunity

# Quote APIs
/quotes:
  get:
    summary: List quotes
  post:
    summary: Create quote
    body: QuoteCreateRequest

/quotes/{id}/clone:
  post:
    summary: Clone quote

/quotes/{id}/submit-for-approval:
  post:
    summary: Submit quote for approval

# Order APIs
/orders:
  post:
    summary: Create order from quote
    body: OrderCreateRequest

/orders/{id}/activate:
  post:
    summary: Activate order

# Billing APIs
/billing-schedules:
  get:
    summary: List billing schedules
  post:
    summary: Create billing schedule

/invoice-batch-runs:
  post:
    summary: Trigger invoice batch run
  get:
    summary: Get batch run status

/invoices:
  get:
    summary: List invoices
  post:
    summary: Generate invoice

# Contract APIs
/contracts:
  get:
    summary: List contracts
  post:
    summary: Create contract

/contracts/{id}/send-for-signature:
  post:
    summary: Send contract for e-signature

# Usage APIs
/usage-records:
  post:
    summary: Record usage
    body: UsageRecordRequest
  get:
    summary: Query usage records
```

#### 5.1.3 Integration Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                 INTEGRATION PATTERNS                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. REQUEST-RESPONSE (Synchronous)                          │
│     ─────────────────────────                                │
│     Use Case: Real-time data retrieval                       │
│     Pattern: REST/SOAP API                                   │
│     Examples:                                                │
│     ├─ Get Product Price                                     │
│     ├─ Validate Customer Credit                              │
│     └─ Check Inventory Availability                          │
│                                                              │
│  2. FIRE-AND-FORGET (Asynchronous)                          │
│     ──────────────────────────                               │
│     Use Case: Non-blocking operations                        │
│     Pattern: Platform Events / Queueable Apex                │
│     Examples:                                                │
│     ├─ Send Notification Email                               │
│     ├─ Log Audit Trail                                       │
│     └─ Update External System                                │
│                                                              │
│  3. BATCH PROCESSING                                        │
│     ────────────────                                         │
│     Use Case: Large volume data processing                   │
│     Pattern: Bulk API / Batch Apex                           │
│     Examples:                                                │
│     ├─ Invoice Generation                                    │
│     ├─ Usage Rating                                          │
│     └─ Data Synchronization                                  │
│                                                              │
│  4. EVENT-DRIVEN                                            │
│     ─────────────                                            │
│     Use Case: Real-time event propagation                    │
│     Pattern: Platform Events / Change Data Capture           │
│     Examples:                                                │
│     ├─ Order Created Event                                   │
│     ├─ Payment Received Event                                │
│     └─ Contract Signed Event                                 │
│                                                              │
│  5. SCHEDULED INTEGRATION                                   │
│     ──────────────────────                                   │
│     Use Case: Periodic data sync                             │
│     Pattern: Scheduled Apex / MuleSoft Scheduler             │
│     Examples:                                                │
│     ├─ Daily ERP Sync                                        │
│     ├─ Monthly Revenue Recognition                           │
│     └─ Weekly Report Generation                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 User Interface Architecture

#### 5.2.1 Lightning Web Components Structure

```
force-app/main/default/lwc/
├── opportunityManagement/
│   ├── opportunityListView/
│   │   ├── opportunityListView.js
│   │   ├── opportunityListView.html
│   │   └── opportunityListView.css
│   ├── opportunityDetail/
│   │   ├── opportunityDetail.js
│   │   └── opportunityDetail.html
│   └── opportunityRelatedLists/
│       └── opportunityRelatedLists.js
│
├── quoteManagement/
│   ├── quoteBuilder/
│   │   ├── quoteBuilder.js
│   │   ├── quoteBuilder.html
│   │   └── quoteBuilder.css
│   ├── quoteLineEditor/
│   │   ├── quoteLineEditor.js
│   │   └── quoteLineEditor.html
│   └── quoteApproval/
│       └── quoteApproval.js
│
├── billingOperations/
│   ├── billingScheduleView/
│   │   └── billingScheduleView.js
│   ├── invoiceBatchRun/
│   │   └── invoiceBatchRun.js
│   └── invoiceViewer/
│       └── invoiceViewer.js
│
├── contractManagement/
│   ├── contractAuthor/
│   │   └── contractAuthor.js
│   ├── contractNegotiation/
│   │   └── contractNegotiation.js
│   └── contractAnalytics/
│       └── contractAnalytics.js
│
└── shared/
    ├── dataTable/
    │   └── dataTable.js
    ├── modal/
    │   └── modal.js
    ├── toast/
    │   └── toast.js
    └── spinner/
        └── spinner.js
```

#### 5.2.2 Component Communication Pattern

```javascript
// Parent Component
import { LightningElement, wire } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class OpportunityDetail extends NavigationMixin(LightningElement) {
    opportunityId;
    opportunity;
    
    // Event handling
    handleQuoteCreated(event) {
        const quoteId = event.detail.quoteId;
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: {
                recordId: quoteId,
                actionName: 'view'
            }
        });
    }
    
    // Wire service
    @wire(getOpportunity, { opportunityId: '$opportunityId' })
    wiredOpportunity({ error, data }) {
        if (data) {
            this.opportunity = data;
        } else if (error) {
            this.dispatchEvent(new ShowToastEvent({
                title: 'Error',
                message: error.body.message,
                variant: 'error'
            }));
        }
    }
}

// Child Component
import { LightningElement, api } from 'lwc';

export default class QuoteLineEditor extends LightningElement {
    @api quoteId;
    
    handleSave() {
        // Save logic
        this.dispatchEvent(new CustomEvent('quotesaved', {
            detail: { quoteId: this.quoteId }
        }));
    }
}
```

---

## 6. สถาปัตยกรรมความปลอดภัย

### 6.1 Security Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                 SECURITY LAYERS                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  NETWORK SECURITY                                    │  │
│  │  ├─ Firewall (Salesforce Managed)                    │  │
│  │  ├─ DDoS Protection                                  │  │
│  │  ├─ IP Whitelisting                                  │  │
│  │  └─ TLS 1.2+ Encryption                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  APPLICATION SECURITY                                │  │
│  │  ├─ Authentication (SSO, MFA)                        │  │
│  │  ├─ Authorization (RBAC, Profiles, Permission Sets)  │  │
│  │  ├─ Session Management                               │  │
│  │  └─ Input Validation                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  DATA SECURITY                                       │  │
│  │  ├─ Encryption at Rest (AES-256)                     │  │
│  │  ├─ Encryption in Transit (TLS)                      │  │
│  │  ├─ Field-Level Security                             │  │
│  │  ├─ Data Masking                                     │  │
│  │  └─ Secure Key Management                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  OPERATIONAL SECURITY                                │  │
│  │  ├─ Audit Logging                                    │  │
│  │  ├─ Monitoring & Alerting                            │  │
│  │  ├─ Incident Response                                │  │
│  │  └─ Security Training                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Authentication & Authorization

#### 6.2.1 Authentication Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    User     │────▶│  Identity   │────▶│ Salesforce  │
│  Browser    │     │   Provider  │     │   Platform  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │  SSO (SAML  │
                  │  / OIDC)    │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │   MFA       │
                  │  (Optional) │
                  └─────────────┘
                           │
                           ▼
                  ┌─────────────┐
                  │  Session    │
                  │  Created    │
                  └─────────────┘
```

#### 6.2.2 Role Hierarchy & Sharing Model

```
┌─────────────────────────────────────────────────────────────┐
│              ROLE HIERARCHY                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    CEO / CFO                                 │
│                       │                                      │
│          ┌────────────┴────────────┐                        │
│          │                         │                        │
│   VP of Sales            VP of Finance                      │
│          │                         │                        │
│    ┌─────┴─────┐            ┌─────┴─────┐                  │
│    │           │            │           │                  │
│ Sales      Sales       Billing      Accounting             │
│ Director   Director    Manager       Manager               │
│    │           │            │           │                  │
│    │           │            │           │                  │
│ Sales Rep  Sales Rep   Billing      Accountant             │
│                        Specialist                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Sharing Model:
├─ Opportunity: Private (Role Hierarchy + Sharing Rules)
├─ Quote: Controlled by Parent (Opportunity)
├─ Order: Private (Role Hierarchy)
├─ Invoice: Private (Role Hierarchy + Manual Sharing)
├─ Contract: Private (Role Hierarchy)
└─ Account: Public Read Only (Role Hierarchy for Write)
```

#### 6.2.3 Permission Sets

```yaml
Permission Sets:

RevenueManagement_Admin:
  Description: Full administrative access
  Permissions:
    - ModifyAllData: true
    - ManageRevenueManagement: true
    - ViewAllReports: true
    - ManageUsers: true

RevenueManagement_Manager:
  Description: Management level access
  Permissions:
    - ViewAllRevenueData: true
    - ApproveQuotes: true
    - ApproveContracts: true
    - ManageBilling: true
    - RunReports: true

RevenueManagement_SalesRep:
  Description: Sales representative access
  Permissions:
    - CreateEditOpportunities: true
    - CreateEditQuotes: true
    - ViewOwnRecords: true
    - SubmitForApproval: true

RevenueManagement_BillingSpecialist:
  Description: Billing operations
  Permissions:
    - ViewInvoices: true
    - CreateInvoices: true
    - ProcessPayments: true
    - ViewBillingSchedules: true
    - RunInvoiceBatch: true

RevenueManagement_ContractManager:
  Description: Contract management
  Permissions:
    - CreateEditContracts: true
    - SendForSignature: true
    - ViewContractAnalytics: true
    - ManageObligations: true

RevenueManagement_ReadOnly:
  Description: Read-only access
  Permissions:
    - ViewRevenueData: true
    - ViewReports: true
    - ExportReports: false
```

### 6.3 Data Encryption

#### 6.3.1 Encryption Strategy

```
┌─────────────────────────────────────────────────────────────┐
│              ENCRYPTION IMPLEMENTATION                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ENCRYPTION AT REST                                         │
│  ├─ Salesforce Platform Encryption (Shield)                 │
│  ├─ Algorithm: AES-256                                      │
│  ├─ Key Management: Salesforce Managed Keys / BYOK          │
│  └─ Encrypted Fields:                                       │
│     ├─ Payment Card Numbers                                 │
│     ├─ Bank Account Numbers                                 │
│     ├─ Tax ID Numbers                                       │
│     ├─ Salary/Compensation Data                             │
│     └─ Discount Percentages                                 │
│                                                              │
│  ENCRYPTION IN TRANSIT                                       │
│  ├─ TLS 1.2+ for all communications                         │
│  ├─ HTTPS Only                                              │
│  ├─ Certificate Pinning (Mobile)                            │
│  └─ Perfect Forward Secrecy                                 │
│                                                              │
│  FIELD-LEVEL ENCRYPTION                                     │
│  ├─ Platform Encryption for Sensitive Fields                │
│  ├─ Deterministic Encryption (for filtering)                │
│  └─ Probabilistic Encryption (for maximum security)         │
│                                                              │
│  KEY MANAGEMENT                                             │
│  ├─ Key Rotation: Every 90 days                             │
│  ├─ Key Backup: Geo-redundant                               │
│  ├─ Key Access: Role-based                                  │
│  └─ Audit Trail: All key operations logged                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.4 Compliance & Auditing

#### 6.4.1 Compliance Requirements

| Standard | Requirement | Implementation |
|----------|-------------|----------------|
| **GDPR** | Data Privacy | Consent Management, Right to Erasure, Data Portability |
| **SOC 2** | Security Controls | Access Controls, Encryption, Audit Logging |
| **PCI DSS** | Payment Security | Tokenization, Network Segmentation, Regular Scanning |
| **ISO 27001** | ISMS | Security Policies, Risk Assessment, Incident Management |
| **ASC 606** | Revenue Recognition | Audit Trail, Revenue Schedules, Compliance Reports |

#### 6.4.2 Audit Trail Implementation

```apex
// Custom Audit Trail Object
Object: Audit_Trail__b (Big Object)
Fields:
├─ Timestamp__c (Date/Time, Indexed)
├─ User_Id__c (Text, Indexed)
├─ User_Name__c (Text)
├─ Object_Type__c (Text, Indexed)
├─ Record_Id__c (Text, Indexed)
├─ Action__c (Text) - Create, Update, Delete, View
├─ Field_Changed__c (Text)
├─ Old_Value__c (Long Text)
├─ New_Value__c (Long Text)
├─ IP_Address__c (Text)
├─ Session_Id__c (Text)
└─ Additional_Info__c (Long Text)

// Apex Trigger for Audit Trail
trigger OpportunityAuditTrail on Opportunity (after insert, after update, after delete) {
    List<Audit_Trail__b> auditRecords = new List<Audit_Trail__b>();
    
    if (Trigger.isAfter) {
        if (Trigger.isInsert) {
            for (Opportunity opp : Trigger.new) {
                auditRecords.add(new Audit_Trail__b(
                    Timestamp__c = System.now(),
                    User_Id__c = UserInfo.getUserId(),
                    User_Name__c = UserInfo.getName(),
                    Object_Type__c = 'Opportunity',
                    Record_Id__c = opp.Id,
                    Action__c = 'CREATE',
                    IP_Address__c = UserInfo.getSessionId()
                ));
            }
        }
        
        if (Trigger.isUpdate) {
            for (Opportunity newOpp : Trigger.new) {
                Opportunity oldOpp = Trigger.oldMap.get(newOpp.Id);
                // Compare fields and log changes
                if (oldOpp.Amount != newOpp.Amount) {
                    auditRecords.add(new Audit_Trail__b(
                        Timestamp__c = System.now(),
                        User_Id__c = UserInfo.getUserId(),
                        User_Name__c = UserInfo.getName(),
                        Object_Type__c = 'Opportunity',
                        Record_Id__c = newOpp.Id,
                        Action__c = 'UPDATE',
                        Field_Changed__c = 'Amount',
                        Old_Value__c = String.valueOf(oldOpp.Amount),
                        New_Value__c = String.valueOf(newOpp.Amount)
                    ));
                }
            }
        }
    }
    
    if (!auditRecords.isEmpty()) {
        insert auditRecords;
    }
}
```

---

## 7. สถาปัตยกรรมโครงสร้างพื้นฐาน

### 7.1 Salesforce Platform Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│           SALESFORCE MULTI-TENANT ARCHITECTURE               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              PRESENTATION TIER                        │  │
│  │                                                       │  │
│  │  - Edge Servers (Global CDN)                         │  │
│  │  - Load Balancers                                     │  │
│  │  - Web Servers (Static Content)                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              APPLICATION TIER                         │  │
│  │                                                       │  │
│  │  - Apex Runtime Engine                               │  │
│  │  - Flow Engine                                        │  │
│  │  - API Servers                                        │  │
│  │  - Batch Processing Engine                           │  │
│  │  - Async Processing (Queueable, Future)              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              DATA TIER                                │  │
│  │                                                       │  │
│  │  - Multi-Tenant Database (Oracle/PostgreSQL)         │  │
│  │  - Database Sharding                                  │  │
│  │  - Replication (Read Replicas)                       │  │
│  │  - Caching Layer (Redis)                             │  │
│  │  - File Storage (Blob Storage)                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              METADATA TIER                            │  │
│  │                                                       │  │
│  │  - Metadata Repository                               │  │
│  │  - Configuration Store                               │  │
│  │  - Custom Object Definitions                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Platform Limits & Governance

#### 7.2.1 Apex Governor Limits

| Limit | Value | Mitigation Strategy |
|-------|-------|-------------------|
| SOQL Queries | 100 per transaction | Use selective queries, Batch Apex |
| DML Statements | 150 per transaction | Bulkify operations |
| CPU Time | 10,000 ms | Optimize code, Async processing |
| Heap Size | 6 MB (sync), 12 MB (async) | Efficient data structures |
| Callouts | 100 per transaction | Batch callouts, Queueable |
| Future Methods | 50 per 24h per org | Use Queueable instead |
| Batch Jobs | 5 concurrent | Schedule strategically |

#### 7.2.2 API Limits

| Edition | API Calls/24h | Recommended Usage |
|---------|--------------|-------------------|
| Enterprise | 100,000 | Core integrations |
| Unlimited | 500,000 | High-volume integrations |
| Performance | 1,000,000 | Enterprise-wide |

### 7.3 Scalability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              SCALABILITY STRATEGIES                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HORIZONTAL SCALING                                         │
│  ├─ Multi-Org Strategy (if needed)                          │
│  ├─ Data Partitioning by Region/Business Unit               │
│  └─ Load Distribution across Salesforce Pods                │
│                                                              │
│  VERTICAL SCALING                                           │
│  ├─ Salesforce Edition Upgrade                              │
│  ├─ Additional Storage                                      │
│  └─ Enhanced API Limits                                     │
│                                                              │
│  CACHING STRATEGY                                           │
│  ├─ Org Cache (100 MB)                                     │
│  │  └─ Product Catalog, Price Books                        │
│  ├─ Session Cache (User-specific)                           │
│  │  └─ User Preferences, Recent Records                    │
│  └─ Platform Cache (Shared)                                 │
│     └─ Configuration Data, Lookup Tables                    │
│                                                              │
│  ASYNC PROCESSING                                           │
│  ├─ Queueable Apex Chains                                   │
│  ├─ Batch Apex for large data volumes                       │
│  ├─ Platform Events for decoupling                          │
│  └─ Scheduled Jobs for off-peak processing                  │
│                                                              │
│  DATABASE OPTIMIZATION                                      │
│  ├─ Selective SOQL Queries                                  │
│  ├─ Custom Indexes on frequently queried fields             │
│  ├─ Skinny Tables for large objects                         │
│  └─ Big Objects for historical data                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.4 Infrastructure Components

#### 7.4.1 Salesforce Editions & Add-ons

```
Core Platform:
├─ Salesforce Unlimited Edition
├─ Revenue Cloud (CPQ + Billing)
├─ Sales Cloud
└─ Service Cloud (Optional)

Add-ons:
├─ Salesforce Shield
│  ├─ Platform Encryption
│  ├─ Event Monitoring
│  └─ Field Audit Trail
├─ Einstein AI
│  ├─ Einstein Prediction Builder
│  └─ Einstein Discovery
├─ MuleSoft Anypoint Platform
├─ Tableau CRM (Analytics)
└─ DocuSign / Adobe Sign (eSignature)

Storage:
├─ Data Storage: 500 GB (included) + Additional
├─ File Storage: 500 GB (included) + Additional
└─ Big Objects Storage: As needed
```

---

## 8. สถาปัตยกรรม Integration

### 8.1 Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              INTEGRATION HUB (MuleSoft)                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │   Salesforce│◀──▶│   MuleSoft  │◀──▶│     ERP     │    │
│  │   Platform  │    │   Runtime   │    │   Systems   │    │
│  └─────────────┘    │   Fabric    │    └─────────────┘    │
│                     │             │                       │
│  ┌─────────────┐    │             │    ┌─────────────┐    │
│  │   Payment   │◀──▶│   API       │◀──▶│   External  │    │
│  │   Gateways  │    │   Gateway   │    │   Services  │    │
│  └─────────────┘    │             │    └─────────────┘    │
│                     │             │                       │
│  ┌─────────────┐    │             │    ┌─────────────┐    │
│  │  E-Signature│◀──▶│   Anypoint  │◀──▶│  Messaging  │    │
│  │  Services   │    │   Platform  │    │  Services   │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 Integration Patterns

#### 8.2.1 ERP Integration

```yaml
Integration: ERP System (SAP/Oracle)
Pattern: Bi-directional Sync
Frequency: Real-time + Scheduled Batch

Data Flows:
  Salesforce to ERP:
    - Customer Master Data
    - Sales Orders
    - Invoices
    - Payments
    
  ERP to Salesforce:
    - Product Master Data
    - Pricing Updates
    - Inventory Levels
    - GL Account Mapping
    
Implementation:
  ├─ MuleSoft APIs for real-time sync
  ├─ Scheduled Batch Jobs (Nightly)
  ├─ Change Data Capture for events
  └─ Error Handling & Retry Logic
```

#### 8.2.2 Payment Gateway Integration

```apex
// Payment Gateway Service Interface
public interface PaymentGatewayService {
    PaymentResult processPayment(PaymentRequest request);
    PaymentResult refundPayment(String transactionId, Decimal amount);
    PaymentStatus checkPaymentStatus(String transactionId);
}

// Implementation Example
public class StripePaymentGateway implements PaymentGatewayService {
    
    @AuraEnabled(cacheable=true)
    public static PaymentResult processPayment(PaymentRequest req) {
        // Callout to Stripe API
        HttpRequest httpRequest = new HttpRequest();
        httpRequest.setEndpoint('callout:Stripe_Gateway/v1/charges');
        httpRequest.setMethod('POST');
        httpRequest.setHeader('Authorization', 'Bearer ' + getApiKey());
        httpRequest.setHeader('Content-Type', 'application/json');
        
        JSONGenerator gen = JSON.createGenerator(true);
        gen.writeStartObject();
        gen.writeNumberField('amount', req.amount * 100); // cents
        gen.writeStringField('currency', req.currency);
        gen.writeStringField('source', req.token);
        gen.writeStringField('description', req.description);
        gen.writeEndObject();
        
        httpRequest.setBody(gen.getAsString());
        
        Http http = new Http();
        HttpResponse response = http.send(httpRequest);
        
        // Parse response and return result
        return parsePaymentResponse(response);
    }
}
```

#### 8.2.3 E-Signature Integration

```
┌─────────────────────────────────────────────────────────────┐
│           E-SIGNATURE INTEGRATION FLOW                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. CONTRACT PREPARATION                                    │
│     ├─ User creates contract in Salesforce                  │
│     ├─ System generates PDF document                        │
│     └─ Document stored in Files/Content                     │
│                                                              │
│  2. SIGNATURE REQUEST                                       │
│     ├─ Apex calls DocuSign/Adobe Sign API                   │
│     ├─ Envelope created with recipients                     │
│     ├─ Signing fields mapped                                │
│     └─ Envelope sent to signers                             │
│                                                              │
│  3. SIGNING PROCESS                                         │
│     ├─ Signers receive email notification                   │
│     ├─ Signers access secure signing portal                 │
│     ├─ Document signed electronically                       │
│     └─ Audit trail captured                                 │
│                                                              │
│  4. COMPLETION & SYNC                                       │
│     ├─ Webhook received from e-signature service            │
│     ├─ Signed PDF downloaded                                │
│     ├─ Attached to Contract record                          │
│     ├─ Contract status updated to "Signed"                  │
│     └─ Downstream processes triggered                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 API Management

#### 8.3.1 Rate Limiting & Throttling

```yaml
Rate Limiting Strategy:

External APIs:
  ├─ Standard Tier: 1,000 requests/hour
  ├─ Premium Tier: 10,000 requests/hour
  └─ Enterprise Tier: 100,000 requests/hour

Internal APIs:
  ├─ Per User: 100 requests/minute
  ├─ Per Org: 10,000 requests/minute
  └─ Burst Allowance: 20% above limit

Implementation:
  ├─ MuleSoft API Gateway Policies
  ├─ Salesforce API Limits Enforcement
  └─ Custom Rate Limiting in Apex
```

---

## 9. สถาปัตยกรรม Deployment

### 9.1 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              DEPLOYMENT PIPELINE                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │Development│───▶│   QA     │───▶│  UAT     │             │
│  │   (Dev)  │    │  (Test)  │    │(Staging) │             │
│  └──────────┘    └──────────┘    └──────────┘             │
│       │               │               │                    │
│       │               │               │                    │
│       ▼               ▼               ▼                    │
│  ┌──────────────────────────────────────────┐             │
│  │           CI/CD Pipeline                 │             │
│  │                                          │             │
│  │  1. Code Commit (Git)                    │             │
│  │  2. Automated Build                      │             │
│  │  3. Unit Tests (80% coverage required)   │             │
│  │  4. Static Code Analysis (PMD, ESLint)   │             │
│  │  5. Security Scan (Checkmarx)            │             │
│  │  6. Deploy to Environment                │             │
│  │  7. Integration Tests                    │             │
│  │  8. Performance Tests                    │             │
│  │  9. UAT Approval                         │             │
│  │  10. Production Deployment               │             │
│  └──────────────────────────────────────────┘             │
│                                                              │
│  ┌──────────────────────────────────────────┐             │
│  │           Production                     │             │
│  │                                          │             │
│  │  ├─ Blue-Green Deployment                │             │
│  │  ├─ Zero-Downtime Updates                │             │
│  │  ├─ Rollback Capability                  │             │
│  │  └─ Post-Deployment Validation           │             │
│  └──────────────────────────────────────────┘             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Environment Strategy

| Environment | Purpose | Refresh Cycle | Data | Users |
|------------|---------|---------------|------|-------|
| **Development** | Active development | On-demand | Synthetic | 50 developers |
| **QA/Test** | Quality assurance | Weekly | Anonymized Prod | 20 QA engineers |
| **UAT/Staging** | User acceptance | Bi-weekly | 50% Prod snapshot | 30 business users |
| **Sandbox (Full)** | Final testing | Monthly | Full Prod copy | All users |
| **Production** | Live operations | N/A | Live data | All users |

### 9.3 Deployment Tools

```yaml
Version Control:
  ├─ Git (GitHub/GitLab/Bitbucket)
  ├─ Branching Strategy: Git Flow
  │  ├─ main (Production)
  │  ├─ develop (Integration)
  │  ├─ feature/* (New features)
  │  ├─ release/* (Release preparation)
  │  └─ hotfix/* (Production fixes)
  └─ Pull Request Reviews (Required: 2 approvers)

CI/CD Tools:
  ├─ Salesforce DevOps Center
  ├─ GitHub Actions
  ├─ Jenkins (Alternative)
  └─ Copado (Enterprise Option)

Deployment Automation:
  ├─ Salesforce CLI (sfdx)
  ├─ Metadata API
  ├─ Source Tracking
  └─ Automated Testing Framework

Quality Gates:
  ├─ Code Coverage: > 80%
  ├─ Static Analysis: 0 Critical Issues
  ├─ Security Scan: Pass
  ├─ Performance Tests: Within SLA
  └─ UAT Sign-off: Required
```

---

## 10. การจัดการ Performance และ Scalability

### 10.1 Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Page Load Time | < 3 seconds | Real User Monitoring |
| API Response Time | < 500 ms (p95) | API Analytics |
| SOQL Query Time | < 2 seconds | Debug Logs |
| Batch Processing | < 4 hours (overnight) | Job Monitoring |
| Concurrent Users | 1,000+ | Load Testing |

### 10.2 Optimization Strategies

```
┌─────────────────────────────────────────────────────────────┐
│              PERFORMANCE OPTIMIZATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  FRONTEND OPTIMIZATION                                      │
│  ├─ Lightning Web Components (Fast rendering)               │
│  ├─ Lazy Loading (On-demand component loading)              │
│  ├─ Caching (Browser + Platform Cache)                      │
│  ├─ Minification (CSS/JS)                                   │
│  └─ CDN for Static Resources                                │
│                                                              │
│  BACKEND OPTIMIZATION                                       │
│  ├─ Selective SOQL (Use indexes)                            │
│  ├─ Bulkification (Process in batches)                      │
│  ├─ Async Processing (Queueable, Batch)                     │
│  ├─ Platform Cache (Frequently accessed data)               │
│  └─ Skinny Tables (Large objects)                           │
│                                                              │
│  DATABASE OPTIMIZATION                                      │
│  ├─ Custom Indexes                                          │
│  ├─ Query Optimization                                      │
│  ├─ Data Archiving                                          │
│  └─ Partitioning (Big Objects)                              │
│                                                              │
│  INTEGRATION OPTIMIZATION                                   │
│  ├─ Batch API Calls (Reduce round trips)                    │
│  ├─ Compression (Gzip)                                      │
│  ├─ Connection Pooling                                      │
│  └─ Caching (External data)                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 10.3 Monitoring & Alerting

```yaml
Performance Monitoring:
  Tools:
    ├─ Salesforce Event Monitoring
    ├─ Lightning Usage App
    ├─ Apex Debug Logs
    ├─ API Usage Reports
    └─ Custom Dashboards
    
  Metrics:
    ├─ Page Load Times
    ├─ API Response Times
    ├─ SOQL Query Performance
    ├─ Governor Limit Usage
    ├─ Concurrent API Requests
    └─ Batch Job Duration

Alerting:
  Critical Alerts (Immediate):
    ├─ System Downtime
    ├─ API Limit > 90%
    ├─ Batch Job Failures
    └─ Security Incidents
    
  Warning Alerts (Within 1 hour):
    ├─ Performance Degradation
    ├─ Storage > 80%
    ├─ Failed Login Attempts
    └─ Integration Errors
    
  Info Alerts (Daily Digest):
    ├─ Usage Statistics
    ├─ License Utilization
    └─ System Health Summary
```

---

## 11. การสำรองข้อมูลและ Disaster Recovery

### 11.1 Backup Strategy

```
┌─────────────────────────────────────────────────────────────┐
│              BACKUP & RECOVERY                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  BACKUP TYPES                                               │
│  ├─ Salesforce Native Backup                                │
│  │  ├─ Automated daily backups                              │
│  │  ├─ Retention: 48 hours (standard)                       │
│  │  └─ Point-in-time recovery                               │
│  │                                                          │
│  ├─ Salesforce Weekly Export                                │
│  │  ├─ Automated CSV export                                 │
│  │  ├─ All objects included                                 │
│  │  └─ Delivered via email/FTP                              │
│  │                                                          │
│  ├─ Third-Party Backup (OwnBackup/Backupify)                │
│  │  ├─ Daily automated backups                              │
│  │  ├─ Retention: 7+ years                                  │
│  │  ├─ Granular restore                                     │
│  │  └─ Compliance archiving                                 │
│  │                                                          │
│  └─ Manual Exports                                          │
│     ├─ Data Loader exports                                  │
│     ├─ Report exports                                       │
│     └─ Metadata backups (Git)                               │
│                                                              │
│  RECOVERY OBJECTIVES                                        │
│  ├─ Recovery Time Objective (RTO): 4 hours                  │
│  ├─ Recovery Point Objective (RPO): 1 hour                  │
│  ├─ Maximum Tolerable Downtime (MTD): 8 hours               │
│  └─ Recovery Time Actual (RTA): Target < RTO                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 Disaster Recovery Plan

```yaml
DR Strategy:
  ├─ Primary: Salesforce Production Org
  ├─ Secondary: Full Copy Sandbox (Warm Standby)
  └─ Tertiary: External Data Backup (Cold Storage)

Failover Process:
  1. Incident Detection
     ├─ Automated Monitoring
     └─ User Reports
     
  2. Assessment & Declaration
     ├─ Impact Analysis
     ├─ DR Team Activation
     └─ Stakeholder Notification
     
  3. Failover Execution
     ├─ Activate Secondary Environment
     ├─ Restore from Backup
     ├─ Validate Data Integrity
     └─ Update DNS/Routing
     
  4. Operations in DR Mode
     ├─ Limited Functionality
     ├─ Manual Processes
     └─ Regular Status Updates
     
  5. Failback to Primary
     ├─ Primary System Restoration
     ├─ Data Synchronization
     ├─ Validation Testing
     └─ Cutover to Primary

Testing Schedule:
  ├─ Quarterly: Tabletop Exercise
  ├─ Bi-annually: Partial Failover Test
  └─ Annually: Full DR Test
```

---

## 12. Monitoring และ Logging

### 12.1 Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              MONITORING STACK                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           APPLICATION MONITORING                      │  │
│  │                                                       │  │
│  │  ├─ Salesforce Health Check                          │  │
│  │  ├─ Event Monitoring                                 │  │
│  │  ├─ Debug Logs                                       │  │
│  │  ├─ Apex Flex Queue Monitoring                       │  │
│  │  └─ Batch Job Monitoring                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           PERFORMANCE MONITORING                      │  │
│  │                                                       │  │
│  │  ├─ Lightning Usage Analytics                        │  │
│  │  ├─ API Usage Reports                                │  │
│  │  ├─ SOQL Performance                                 │  │
│  │  └─ Page Load Times                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SECURITY MONITORING                         │  │
│  │                                                       │  │
│  │  ├─ Login History                                    │  │
│  │  ├─ Session Management                               │  │
│  │  ├─ Field Audit Trail                                │  │
│  │  └─ Event Monitoring (Security Events)               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           INTEGRATION MONITORING                      │  │
│  │                                                       │  │
│  │  ├─ API Callout Monitoring                           │  │
│  │  ├─ MuleSoft Analytics                               │  │
│  │  ├─ Webhook Delivery Status                          │  │
│  │  └─ Integration Error Tracking                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 Logging Strategy

```yaml
Log Levels:
  ERROR:
    ├─ System failures
    ├─ Integration failures
    ├─ Data corruption
    └─ Security violations
    
  WARN:
    ├─ Performance degradation
    ├─ Near-limit conditions
    ├─ Validation failures
    └─ Retry attempts
    
  INFO:
    ├─ Business transactions
    ├─ User actions (critical)
    ├─ Batch job progress
    └─ Integration events
    
  DEBUG:
    ├─ Detailed processing logic
    ├─ Variable states
    ├─ Query execution
    └─ API request/response

Log Storage:
  ├─ Salesforce Debug Logs: 7 days
  ├─ Event Monitoring Logs: 30 days (standard), 1 year (add-on)
  ├─ Custom Audit Logs: 5 years (Big Objects)
  └─ External SIEM: 7 years

Log Aggregation:
  ├─ Salesforce Event Monitoring
  ├─ MuleSoft Anypoint Monitoring
  ├─ External SIEM (Splunk/ELK)
  └─ Custom Dashboards
```

---

## ภาคผนวก

### ภาคผนวก A: Glossary

| Term | Definition |
|------|------------|
| **Apex** | Salesforce proprietary programming language |
| **LWC** | Lightning Web Components |
| **SOQL** | Salesforce Object Query Language |
| **DML** | Data Manipulation Language |
| **Governor Limits** | Salesforce runtime limits |
| **Big Objects** | Salesforce storage for massive data |
| **Platform Events** | Event-driven architecture feature |
| **Change Data Capture** | Real-time data change tracking |

### ภาคผนวก B: References

1. Salesforce Architecture Documentation
2. Salesforce Security Guide
3. Salesforce Platform Limits
4. MuleSoft Documentation
5. Revenue Cloud Best Practices
6. Apex Developer Guide
7. Lightning Web Components Developer Guide

### ภาคผนวก C: Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Chief Architect | | | |
| Technical Lead | | | |
| Security Officer | | | |
| Product Owner | | | |
| IT Director | | | |

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการออกแบบและพัฒนาสถาปัตยกรรมระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review
