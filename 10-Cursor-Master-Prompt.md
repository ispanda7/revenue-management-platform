# 09-Google-Stitch-Prompt.md

# ROLE

Act as a World-Class Product Designer, UX Strategist, UX Architect, UI Designer, Design System Expert, and Frontend Solution Architect.

You will analyze the Product Requirements Document (PRD) for the **Revenue Management System (Order-to-Cash, CPQ, Billing, and Contracts)** and transform it into a complete UX/UI design specification ready for Google Stitch, v0, Figma, and Frontend Development.

---

# CONTEXT: REVENUE MANAGEMENT PRD SUMMARY

**System Overview:** An enterprise-grade Revenue Management platform covering the entire Order-to-Cash lifecycle.
**Core Modules:** 
1. **Catalog & CPQ:** Product management, Configure-Price-Quote, discount approvals.
2. **Orders & Assets:** Order activation, asset generation, entitlement management.
3. **Billing & Invoicing:** Usage rating, invoice generation, batch processing, payment collection.
4. **Contracts:** Contract authoring, negotiation, e-signature, lifecycle management.
5. **AI & Analytics:** Revenue forecasting, AI-driven insights (Agentforce), automated dunning.

**Key Personas:** Sales Representative, Billing Operations Specialist (James), Finance Manager, Contract Manager, System Admin.

---

# OBJECTIVES

Create a world-class UX/UI experience that is:

* **Enterprise Grade:** Handles complex data, high volume, and strict compliance.
* **Mobile First:** Fully responsive, optimized for mobile approvals and quick actions.
* **AI Native:** Seamless integration of AI Copilot, smart suggestions, and automated workflows.
* **Modern SaaS:** Clean, fast, and intuitive, breaking away from legacy enterprise clunkiness.
* **Highly Scalable:** UI architecture that supports thousands of records without performance degradation.
* **Easy to Learn / Easy to Use:** Minimal cognitive load, clear information hierarchy.
* **WCAG 2.2 Accessible:** Fully compliant with enterprise accessibility standards.

**Benchmark against:** Salesforce (functionality), HubSpot (usability), Stripe Dashboard (financial clarity), Linear (speed/aesthetics), and Vercel (modern developer experience).

---

# TASKS

## 1. Analyze PRD
Extract and internalize:
* **Business Objectives:** Accelerate Order-to-Cash, reduce billing errors, automate revenue recognition.
* **User Personas & Roles:** Sales Rep, Billing Specialist, Finance Manager, Contract Manager, Admin.
* **Core Features:** CPQ, Order Management, Invoicing, Contract Lifecycle, Usage Rating.
* **User Workflows:** Opportunity → Quote → Order → Invoice → Payment → Revenue Recognition.

## 2. Information Architecture
Generate:
* **Sitemap:** Logical grouping of modules (Sales, Billing, Finance, Admin).
* **Navigation Structure:** Global sidebar navigation, contextual top bars, command palette (Cmd+K).
* **User Journey:** End-to-end flow from Lead creation to Cash collection.

## 3. Screen Inventory
Identify every required screen. For each screen provide: Screen Name, Purpose, User Goal, Components, Actions, Permissions, Data Displayed.

**Mandatory Screen Categories:**
* **Authentication:** Login, MFA, Forgot Password, SSO.
* **Dashboards:** Executive, Sales Pipeline, Billing Operations, Financial Analytics.
* **Sales & CPQ:** Opportunities (List/Detail/Create), Quotes (List/Detail/Builder/Edit), Approvals.
* **Orders & Fulfillment:** Orders (List/Detail/Activate), Assets (List/Detail), Subscriptions.
* **Billing & Finance:** Invoices (List/Detail/Preview/Print), Payments (List/Detail/Record), Billing Schedules, Usage Records.
* **Contracts:** Contracts (List/Detail/Editor/Signature), Clauses, Templates.
* **Catalog:** Products (List/Detail/Create), Price Books, Pricing Rules.
* **Reports & AI:** Revenue Reports, Forecasting, AI Insights, Custom Report Builder.
* **Administration:** Users, Roles, Tenants, Integrations, Audit Logs, Settings.

## 4. UX Design
For each screen provide:
### Layout Structure
* **Header:** Global search, notifications, AI Copilot toggle, user profile.
* **Sidebar:** Collapsible, module-based navigation with badges for pending tasks.
* **Content Area:** Data-dense but breathable. Use tabs, accordions, and drawers to manage complexity.
* **Filters & Search:** Advanced filtering panels, saved views, global search.
* **Actions:** Contextual action bars (Bulk actions, primary CTA).

### User States
* **Empty State:** Illustrations + clear CTA to create the first record.
* **Loading State:** Skeleton screens (no spinners for main content).
* **Error State:** Inline validation, clear recovery steps.
* **Success State:** Toast notifications, optimistic UI updates.

### Responsive Behavior
* **Desktop:** Multi-column, data tables, side-by-side comparisons.
* **Tablet:** Stacked columns, horizontal scroll for tables, bottom sheets.
* **Mobile:** Single column, card-based lists, full-screen modals, bottom navigation for key tasks.

## 5. Design System
Generate:
### Colors
* **Primary:** Trust Blue (`#2563EB`) for primary actions and links.
* **Neutrals:** Slate scale (`#F8FAFC` to `#0F172A`) for backgrounds and text.
* **Semantic:** 
  * Success (`#16A34A`) for Paid, Won, Active.
  * Warning (`#D97706`) for Pending, Expiring, Overdue.
  * Error (`#DC2626`) for Failed, Rejected, Critical.
  * Info (`#0284C7`) for Draft, Informational.

### Typography
* **Font Family:** Inter (UI text), JetBrains Mono (Financial numbers, IDs, code).
* **Heading Scale:** Clear hierarchy (H1 to H6) using font weight and color, not just size.
* **Body Scale:** 14px base, 16px for reading text.

### Spacing, Radius, Shadows
* **Spacing:** 4px base grid (4, 8, 12, 16, 24, 32, 48).
* **Border Radius:** 8px for cards, 6px for inputs/buttons, full for avatars/badges.
* **Shadows:** Subtle, layered shadows for depth (sm, md, lg, xl).

### Icon System
* Lucide React icons, 20px for UI, 16px for dense tables.

### Dark Mode
* Fully supported. High contrast, reduced glare, semantic colors adjusted for dark backgrounds.

## 6. Component Library
Create reusable, highly polished components:
* **Primitives:** Button, Input, Select, Checkbox, Switch, Badge, Avatar, Tooltip.
* **Data Display:** Data Table (sortable, filterable, selectable), Dashboard Card, KPI Widget, Timeline, Activity Feed.
* **Financial Specific:** `MoneyDisplay` (right-aligned, monospace, handles negative/positive), `StatusBadge` (color-coded by entity type), `DateRangePicker`.
* **Complex Components:** `QuoteLineItemEditor` (inline editing), `ApprovalStepper`, `AI Copilot Widget` (floating or side panel), `Chart` (Recharts integration).

## 7. Dashboard Design
Create:
### Executive Dashboard
* High-level KPIs (MRR, ARR, DSO, Churn).
* Revenue trend charts (Area/Line).
* AI-generated insights and risk alerts.

### Operational Dashboard (Billing/Sales)
* Task queues (Pending Approvals, Overdue Invoices, Expiring Contracts).
* Quick action buttons.
* Real-time monitoring widgets.

### Analytics Dashboard
* Interactive, drill-down capable charts.
* Exportable data views.

## 8. AI Features
Design UX for:
* **AI Copilot:** Floating action button that opens a side panel for natural language queries ("Show me overdue invoices over $10k").
* **AI Recommendations:** Inline suggestions in Quote Builder ("Customers with this profile usually add Premium Support").
* **Smart Search:** Semantic search across all entities.

## 9. Mobile Experience
Design:
* **Mobile Navigation:** Bottom tab bar for primary modules, hamburger for secondary.
* **Quick Actions:** Swipe-to-approve on quotes/invoices.
* **Offline Mode:** Read-only cache with sync indicator.
* **Notifications:** Push notification center.

## 10. Output Format
Generate:
1. UX Specification Document
2. UI Specification Document
3. Design System Tokens (JSON/CSS)
4. Component Library API
5. User Flow Diagrams (Mermaid.js)
6. Screen Flow Diagrams
7. Wireframe Descriptions (ASCII/Text)
8. Responsive Design Rules
9. Figma Ready Specification
10. **Google Stitch / v0 Screen Generation Prompts** (Detailed prompts for each screen to generate code directly).

---

# REQUIREMENTS

- **Mobile First:** Design for mobile constraints first, then scale up to desktop.
- **Light Mode Default:** Primary design in light mode, with dark mode tokens provided.
- **Responsive:** Fluid layouts, no fixed widths.
- **Design Language:** Material Design 3 principles (elevation, state layers, dynamic color) adapted for **Tailwind CSS and shadcn/ui**.

# TECH STACK

- **Framework:** Next.js 14+ (App Router)
- **Styling:** Tailwind CSS
- **UI Library:** shadcn/ui (Radix UI primitives)
- **Icons:** Lucide React
- **Charts:** Recharts / Tremor
- **State Management:** Zustand / TanStack Query

---

# CRITICAL REQUIREMENT

Before generating UI designs:

1. Analyze PRD completely.
2. Identify ALL entities (Opportunity, Quote, Order, Invoice, Contract, Asset, Product, Payment).
3. Identify ALL workflows (Quote-to-Order, Order-to-Cash, Contract-to-Asset).
4. Identify ALL CRUD operations for every entity.
5. Identify ALL reports and dashboards.
6. Identify ALL administration and settings functions.
7. Identify ALL approval workflows (Quote approval, Discount approval, Contract approval).
8. Identify ALL AI features and touchpoints.
9. **Generate COMPLETE SCREEN INVENTORY first.**

**DO NOT SKIP ANY SCREEN.**

Every core entity (Opportunity, Quote, Order, Invoice, Contract, Asset, Product) MUST have:
- List Screen (with advanced filters, bulk actions, saved views)
- Detail Screen (comprehensive view, related records, timeline)
- Create Screen (Wizard or comprehensive form)
- Edit Screen (Inline or modal)
- Delete Confirmation (Destructive action modal)
- History/Audit Log Screen (Version history)
- Activity Feed (Contextual timeline)
- Permission/Access Screen (If applicable)

**Generate UI for ALL screens.**

**Expected Output:** 
50-100+ distinct screen layouts and component variations depending on system complexity. Provide detailed visual descriptions and exact Tailwind/shadcn class structures for the Google Stitch/v0 generation prompts.
