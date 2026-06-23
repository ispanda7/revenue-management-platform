# UX/UI Specification

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**กลุ่มเป้าหมาย:** UX/UI Designers, Frontend Developers, AI (Design-to-Code)  
**Tech Stack:** React 18 + TypeScript + Tailwind CSS + Radix UI + shadcn/ui

---

## 📋 สารบัญ

1. [Design System](#1-design-system)
2. [Color Palette](#2-color-palette)
3. [Typography](#3-typography)
4. [Components](#4-components)
5. [Pages](#5-pages)
6. [Responsive Design](#6-responsive-design)
7. [Accessibility](#7-accessibility)
8. [User Flow](#8-user-flow)
9. [Wireframes](#9-wireframes)
10. [Mockups](#10-mockups)

---

## 1. Design System

### 1.1 Design Philosophy

```
┌─────────────────────────────────────────────────────────────────┐
│                  DESIGN PHILOSOPHY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CLARITY FIRST                                                │
│     ├─ ข้อมูลทางการเงินต้องชัดเจน ไม่คลุมเครือ                   │
│     ├─ ตัวเลขสำคัญต้องเด่น (monospace, larger size)              │
│     └─ สถานะ (status) ต้องเห็นได้ทันที                           │
│                                                                  │
│  2. EFFICIENT WORKFLOWS                                          │
│     ├─ ลดจำนวนคลิกในการทำงานซ้ำๆ                                  │
│     ├─ Keyboard shortcuts สำหรับ power users                     │
│     ├─ Bulk actions สำหรับการจัดการจำนวนมาก                      │
│     └─ Quick actions บนทุก record                                │
│                                                                  │
│  3. DATA-DENSITY BALANCE                                         │
│     ├─ Compact mode สำหรับผู้ใช้ขั้นสูง                          │
│     ├─ Comfortable mode สำหรับผู้ใช้ทั่วไป                       │
│     └─ สลับได้ทันที ไม่ต้อง reload                                │
│                                                                  │
│  4. TRUST & PROFESSIONALISM                                      │
│     ├─ สีที่สื่อถึงความน่าเชื่อถือ (น้ำเงิน, เทา)                  │
│     ├─ Animation ที่นุ่มนวล ไม่รบกวนการทำงาน                     │
│     └─ Error messages ที่ช่วยเหลือ ไม่ใช่ขู่                      │
│                                                                  │
│  5. AI-READY                                                     │
│     ├─ Design tokens สำหรับ AI generation                        │
│     ├─ Component API ที่ชัดเจน                                   │
│     └─ Figma variables synced กับ code                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

| Principle | Description | Example |
|-----------|-------------|---------|
| **Progressive Disclosure** | แสดงข้อมูลเท่าที่จำเป็น ซ่อนรายละเอียด | Quote summary → expand to see line items |
| **Consistency** | Pattern เดียวกันทั้งระบบ | ทุก table มี pagination, filter, sort แบบเดียวกัน |
| **Feedback** | ทุก action ต้องมี response | Toast notification, loading state, success animation |
| **Forgiveness** | อนุญาตให้ undo/redo | Undo delete, draft auto-save, confirmation dialogs |
| **Context Awareness** | แสดงข้อมูลที่เกี่ยวข้อง | Sidebar แสดง related records, recent activities |

### 1.3 Design Tokens

```typescript
// tokens.ts - Design Tokens (synced with Figma variables)
export const tokens = {
  // Spacing (4px grid system)
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
    '2xl': '48px',
    '3xl': '64px',
  },
  
  // Border Radius
  radius: {
    none: '0',
    sm: '4px',
    md: '8px',
    lg: '12px',
    xl: '16px',
    full: '9999px',
  },
  
  // Shadows
  shadow: {
    sm: '0 1px 2px rgba(0, 0, 0, 0.05)',
    md: '0 4px 6px -1px rgba(0, 0, 0, 0.1)',
    lg: '0 10px 15px -3px rgba(0, 0, 0, 0.1)',
    xl: '0 20px 25px -5px rgba(0, 0, 0, 0.1)',
  },
  
  // Motion
  motion: {
    duration: {
      instant: '50ms',
      fast: '150ms',
      normal: '250ms',
      slow: '400ms',
    },
    easing: {
      easeIn: 'cubic-bezier(0.4, 0, 1, 1)',
      easeOut: 'cubic-bezier(0, 0, 0.2, 1)',
      easeInOut: 'cubic-bezier(0.4, 0, 0.2, 1)',
      spring: 'cubic-bezier(0.34, 1.56, 0.64, 1)',
    },
  },
  
  // Z-index scale
  zIndex: {
    base: 0,
    dropdown: 100,
    sticky: 200,
    modal: 300,
    popover: 400,
    toast: 500,
    tooltip: 600,
  },
};
```

### 1.4 Icon System

```yaml
Icon Library: Lucide React (400+ icons)
Size Scale: 16px (sm), 20px (md), 24px (lg), 32px (xl)
Stroke Width: 1.5px (default), 2px (emphasis)
Color: Inherit from parent (currentColor)

Icon Usage Guidelines:
  Navigation: 20px, stroke 1.5
  Buttons: 16px, stroke 1.5
  Status Indicators: 16px, filled variant
  Empty States: 64px, stroke 1
  Feature Icons: 24px, stroke 1.5

Common Icons:
  Opportunity: Target
  Quote: FileText
  Order: ShoppingCart
  Invoice: Receipt
  Contract: FileSignature
  Asset: Package
  Payment: CreditCard
  Customer: Users
  Product: Box
  Analytics: TrendingUp
```

---

## 2. Color Palette

### 2.1 Brand Colors

```
┌─────────────────────────────────────────────────────────────────┐
│                  BRAND COLOR SYSTEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIMARY: Trust Blue                                             │
│  ├─ 50:  #EFF6FF  (Backgrounds, hover states)                   │
│  ├─ 100: #DBEAFE  (Light backgrounds)                           │
│  ├─ 200: #BFDBFE  (Borders, dividers)                           │
│  ├─ 300: #93C5FD  (Icons, secondary elements)                   │
│  ├─ 400: #60A5FA  (Links, interactive)                          │
│  ├─ 500: #3B82F6  (Primary buttons, CTAs) ★                     │
│  ├─ 600: #2563EB  (Hover states)                                │
│  ├─ 700: #1D4ED8  (Active states)                               │
│  ├─ 800: #1E40AF  (Dark emphasis)                               │
│  └─ 900: #1E3A8A  (Text on light backgrounds)                   │
│                                                                  │
│  NEUTRAL: Slate                                                  │
│  ├─ 50:  #F8FAFC  (Page background)                             │
│  ├─ 100: #F1F5F9  (Card background)                             │
│  ├─ 200: #E2E8F0  (Borders)                                     │
│  ├─ 300: #CBD5E1  (Disabled states)                             │
│  ├─ 400: #94A3B8  (Placeholder text)                            │
│  ├─ 500: #64748B  (Secondary text)                              │
│  ├─ 600: #475569  (Body text)                                   │
│  ├─ 700: #334155  (Headings)                                    │
│  ├─ 800: #1E293B  (Primary text)                                │
│  └─ 900: #0F172A  (Strong emphasis)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Semantic Colors

```
┌─────────────────────────────────────────────────────────────────┐
│                  SEMANTIC COLORS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SUCCESS (Growth, Positive)                                      │
│  ├─ 50:  #F0FDF4  100: #DCFCE7  200: #BBF7D0                   │
│  ├─ 500: #22C55E ★ (Success messages, positive changes)         │
│  ├─ 600: #16A34A  700: #15803D                                  │
│  Use: Completed orders, paid invoices, won deals                │
│                                                                  │
│  WARNING (Caution, Attention)                                    │
│  ├─ 50:  #FFFBEB  100: #FEF3C7  200: #FDE68A                   │
│  ├─ 500: #F59E0B ★ (Warnings, pending actions)                  │
│  ├─ 600: #D97706  700: #B45309                                  │
│  Use: Pending approvals, expiring contracts, low stock          │
│                                                                  │
│  ERROR (Critical, Negative)                                      │
│  ├─ 50:  #FEF2F2  100: #FEE2E2  200: #FECACA                   │
│  ├─ 500: #EF4444 ★ (Errors, deletions, failures)                │
│  ├─ 600: #DC2626  700: #B91C1C                                  │
│  Use: Failed payments, overdue invoices, validation errors      │
│                                                                  │
│  INFO (Neutral, Informational)                                   │
│  ├─ 50:  #EFF6FF  100: #DBEAFE  200: #BFDBFE                   │
│  ├─ 500: #3B82F6 ★ (Information, tips, help)                    │
│  ├─ 600: #2563EB  700: #1D4ED8                                  │
│  Use: Tips, feature announcements, system messages              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Business-Specific Colors

```yaml
# Status-Specific Colors
Status Colors:
  Opportunity:
    qualification: #94A3B8 (slate-400)
    discovery: #60A5FA (blue-400)
    proposal: #3B82F6 (blue-500)
    negotiation: #F59E0B (amber-500)
    closed_won: #22C55E (green-500)
    closed_lost: #EF4444 (red-500)
  
  Quote:
    draft: #94A3B8 (slate-400)
    pending_approval: #F59E0B (amber-500)
    approved: #22C55E (green-500)
    rejected: #EF4444 (red-500)
    expired: #64748B (slate-500)
    converted: #3B82F6 (blue-500)
  
  Order:
    pending: #F59E0B (amber-500)
    activated: #3B82F6 (blue-500)
    fulfilled: #22C55E (green-500)
    cancelled: #EF4444 (red-500)
    expired: #64748B (slate-500)
  
  Invoice:
    draft: #94A3B8 (slate-400)
    issued: #3B82F6 (blue-500)
    sent: #60A5FA (blue-400)
    paid: #22C55E (green-500)
    overdue: #EF4444 (red-500)
    cancelled: #64748B (slate-500)
  
  Contract:
    draft: #94A3B8 (slate-400)
    negotiation: #F59E0B (amber-500)
    pending_signature: #60A5FA (blue-400)
    active: #22C55E (green-500)
    expired: #64748B (slate-500)
    terminated: #EF4444 (red-500)

# Chart Colors (for dashboards)
Chart Palette:
  - #3B82F6 (blue-500)
  - #22C55E (green-500)
  - #F59E0B (amber-500)
  - #EF4444 (red-500)
  - #8B5CF6 (violet-500)
  - #EC4899 (pink-500)
  - #14B8A6 (teal-500)
  - #F97316 (orange-500)
```

### 2.4 Dark Mode

```typescript
// Dark mode color mapping
const darkMode = {
  // Backgrounds
  'bg-page': '#0F172A',        // slate-900
  'bg-card': '#1E293B',        // slate-800
  'bg-elevated': '#334155',    // slate-700
  
  // Text
  'text-primary': '#F8FAFC',   // slate-50
  'text-secondary': '#CBD5E1', // slate-300
  'text-muted': '#94A3B8',     // slate-400
  
  // Borders
  'border-default': '#334155', // slate-700
  'border-strong': '#475569',  // slate-600
  
  // Interactive
  'interactive-primary': '#3B82F6',
  'interactive-hover': '#60A5FA',
  
  // Status (slightly brighter for dark mode)
  'success': '#4ADE80',        // green-400
  'warning': '#FBBF24',        // amber-400
  'error': '#F87171',          // red-400
  'info': '#60A5FA',           // blue-400
};
```

### 2.5 Color Usage Rules

```yaml
Rules:
  1. NEVER use color as the only way to convey information
     - Always pair with icon, text, or pattern
     - Example: Error state = red border + error icon + error text
  
  2. Contrast ratios (WCAG 2.1 AA):
     - Normal text: 4.5:1 minimum
     - Large text (18px+): 3:1 minimum
     - UI components: 3:1 minimum
  
  3. Color blindness considerations:
     - Never rely on red/green alone
     - Use patterns/icons for status
     - Provide colorblind-friendly theme option
  
  4. Brand color usage:
     - Primary blue: CTAs, links, active states
     - Max 10% of screen should be primary color
     - Use sparingly to maintain impact
  
  5. Semantic color usage:
     - Success: Only for positive outcomes
     - Error: Only for actual errors (not warnings)
     - Warning: For caution, not errors
```

---

## 3. Typography

### 3.1 Font Stack

```css
/* Primary Font: Inter (UI text) */
--font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 
             Roboto, 'Helvetica Neue', Arial, sans-serif;

/* Monospace Font: JetBrains Mono (numbers, code) */
--font-mono: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;

/* Display Font: Plus Jakarta Sans (headings, marketing) */
--font-display: 'Plus Jakarta Sans', 'Inter', sans-serif;
```

### 3.2 Type Scale

```
┌─────────────────────────────────────────────────────────────────┐
│                  TYPE SCALE (Major Third - 1.25)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Token       │ Size   │ Line Height │ Weight  │ Usage            │
│  ─────────────┼────────┼─────────────┼─────────┼────────────────  │
│  text-xs     │ 12px   │ 16px        │ 400     │ Captions, labels │
│  text-sm     │ 14px   │ 20px        │ 400     │ Body small       │
│  text-base   │ 16px   │ 24px        │ 400     │ Body default ★   │
│  text-lg     │ 18px   │ 28px        │ 500     │ Lead text        │
│  text-xl     │ 20px   │ 28px        │ 600     │ Section heading  │
│  text-2xl    │ 24px   │ 32px        │ 600     │ Page heading     │
│  text-3xl    │ 30px   │ 36px        │ 700     │ Hero heading     │
│  text-4xl    │ 36px   │ 40px        │ 700     │ Display          │
│  text-5xl    │ 48px   │ 48px        │ 700     │ Marketing        │
│                                                                  │
│  Numeric Scale (for financial data)                              │
│  ──────────────────────────────────────────────────────────────  │
│  num-sm      │ 14px   │ 20px        │ 500     │ Table cells      │
│  num-base    │ 16px   │ 24px        │ 500     │ Inline amounts   │
│  num-lg      │ 20px   │ 28px        │ 600     │ Card values      │
│  num-xl      │ 28px   │ 32px        │ 700     │ Dashboard KPIs   │
│  num-2xl     │ 36px   │ 40px        │ 700     │ Hero metrics     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Typography in Practice

```typescript
// Typography examples
const TypographyExamples = () => (
  <>
    {/* Page Heading */}
    <h1 className="font-display text-3xl font-bold text-slate-900 tracking-tight">
      Revenue Dashboard
    </h1>
    
    {/* Section Heading */}
    <h2 className="font-sans text-xl font-semibold text-slate-800">
      Monthly Recurring Revenue
    </h2>
    
    {/* Body Text */}
    <p className="font-sans text-base text-slate-600 leading-relaxed">
      Track your revenue performance across all products and customer segments.
    </p>
    
    {/* Financial Number (Dashboard KPI) */}
    <div className="font-mono text-4xl font-bold text-slate-900 tabular-nums">
      $1,234,567.89
    </div>
    
    {/* Caption */}
    <p className="font-sans text-xs text-slate-500 uppercase tracking-wide">
      Last updated 5 minutes ago
    </p>
    
    {/* Label */}
    <label className="font-sans text-sm font-medium text-slate-700">
      Customer Email
    </label>
  </>
);
```

### 3.4 Number Formatting

```typescript
// Number formatting rules for financial data
const formatCurrency = (
  amount: number,
  currency: string = 'USD',
  locale: string = 'en-US'
): string => {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount);
};

// Examples:
formatCurrency(1234567.89, 'USD')  // "$1,234,567.89"
formatCurrency(1234567.89, 'THB')  // "THB 1,234,567.89"
formatCurrency(0, 'USD')           // "$0.00"
formatCurrency(-500, 'USD')        // "-$500.00"

// CSS for tabular numbers (aligned columns)
.tabular-nums {
  font-variant-numeric: tabular-nums;
  font-feature-settings: "tnum";
}
```

### 3.5 Typography Rules

```yaml
Rules:
  1. Hierarchy:
     - Max 3 heading levels per page
     - Clear visual distinction between levels
     - Use weight + size + color, not just size
  
  2. Readability:
     - Body text: 16px minimum
     - Line height: 1.5x for body, 1.2x for headings
     - Max line length: 65-75 characters
  
  3. Financial Data:
     - ALWAYS use monospace font for numbers
     - ALWAYS use tabular-nums for alignment
     - Right-align currency columns in tables
     - Use consistent decimal places (2 for currency)
  
  4. Localization:
     - Support RTL languages (Arabic, Hebrew)
     - Use locale-aware number formatting
     - Respect cultural date formats
  
  5. Accessibility:
     - Never use text as decoration only
     - Provide sufficient contrast
     - Allow text resizing up to 200%
```

---

## 4. Components

### 4.1 Component Library Structure

```
components/
├── primitives/           # Basic building blocks
│   ├── Button/
│   ├── Input/
│   ├── Select/
│   ├── Checkbox/
│   ├── Radio/
│   ├── Switch/
│   ├── Badge/
│   ├── Avatar/
│   ├── Icon/
│   └── Spinner/
│
├── composite/            # Composed components
│   ├── Card/
│   ├── Modal/
│   ├── Drawer/
│   ├── Dropdown/
│   ├── Tabs/
│   ├── Accordion/
│   ├── Toast/
│   ├── Tooltip/
│   └── Popover/
│
├── patterns/             # Reusable patterns
│   ├── DataTable/
│   ├── FormBuilder/
│   ├── SearchBar/
│   ├── FilterPanel/
│   ├── Pagination/
│   ├── Breadcrumb/
│   ├── StatusBadge/
│   ├── MoneyDisplay/
│   └── Timeline/
│
├── layouts/              # Layout components
│   ├── AppShell/
│   ├── Sidebar/
│   ├── Header/
│   ├── PageContainer/
│   ├── TwoColumn/
│   └── DashboardGrid/
│
└── business/             # Business-specific
    ├── OpportunityCard/
    ├── QuoteBuilder/
    ├── OrderTimeline/
    ├── InvoicePreview/
    ├── ContractEditor/
    ├── AssetList/
    ├── BillingSchedule/
    └── RevenueChart/
```

### 4.2 Core Components

#### Button

```typescript
// Button component with variants
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
  size: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  icon?: React.ReactNode;
  iconPosition?: 'left' | 'right';
  loading?: boolean;
  disabled?: boolean;
  fullWidth?: boolean;
  onClick?: () => void;
}

// Usage examples
<Button variant="primary" size="md">
  Create Quote
</Button>

<Button variant="outline" size="sm" icon={<PlusIcon />}>
  Add Item
</Button>

<Button variant="danger" size="md" loading>
  Deleting...
</Button>

<Button variant="ghost" size="sm" icon={<MoreVerticalIcon />}>
  {/* Icon only */}
</Button>
```

#### DataTable

```typescript
// DataTable with sorting, filtering, pagination
interface DataTableProps<T> {
  columns: ColumnDef<T>[];
  data: T[];
  pagination?: {
    page: number;
    pageSize: number;
    total: number;
    onPageChange: (page: number) => void;
  };
  sorting?: {
    field: string;
    direction: 'asc' | 'desc';
    onSort: (field: string) => void;
  };
  filters?: FilterConfig[];
  selectable?: boolean;
  onSelectionChange?: (ids: string[]) => void;
  rowActions?: RowAction<T>[];
  emptyState?: React.ReactNode;
}

// Column definition
const columns: ColumnDef<Invoice>[] = [
  {
    id: 'select',
    header: ({ table }) => <Checkbox checked={...} />,
    cell: ({ row }) => <Checkbox checked={...} />,
    size: 40,
  },
  {
    accessorKey: 'invoiceNumber',
    header: 'Invoice #',
    cell: ({ row }) => (
      <Link to={`/invoices/${row.id}`}>
        {row.original.invoiceNumber}
      </Link>
    ),
  },
  {
    accessorKey: 'customer',
    header: 'Customer',
    cell: ({ row }) => (
      <div className="flex items-center gap-2">
        <Avatar src={row.original.customer.avatar} />
        <span>{row.original.customer.name}</span>
      </div>
    ),
  },
  {
    accessorKey: 'amount',
    header: ({ column }) => (
      <SortableHeader column={column} align="right">
        Amount
      </SortableHeader>
    ),
    cell: ({ row }) => (
      <MoneyDisplay 
        amount={row.original.amount} 
        currency={row.original.currency}
        align="right"
      />
    ),
  },
  {
    accessorKey: 'status',
    header: 'Status',
    cell: ({ row }) => (
      <StatusBadge status={row.original.status} type="invoice" />
    ),
  },
  {
    accessorKey: 'dueDate',
    header: 'Due Date',
    cell: ({ row }) => (
      <DateDisplay 
        date={row.original.dueDate}
        highlightOverdue
      />
    ),
  },
  {
    id: 'actions',
    cell: ({ row }) => <RowActions actions={rowActions} data={row.original} />,
  },
];
```

#### StatusBadge

```typescript
// StatusBadge for all entity statuses
interface StatusBadgeProps {
  status: string;
  type: 'opportunity' | 'quote' | 'order' | 'invoice' | 'contract' | 'asset';
  size?: 'sm' | 'md';
  showIcon?: boolean;
}

// Color mapping per type
const statusColors = {
  opportunity: {
    qualification: 'bg-slate-100 text-slate-700',
    discovery: 'bg-blue-100 text-blue-700',
    proposal: 'bg-blue-100 text-blue-700',
    negotiation: 'bg-amber-100 text-amber-700',
    closed_won: 'bg-green-100 text-green-700',
    closed_lost: 'bg-red-100 text-red-700',
  },
  invoice: {
    draft: 'bg-slate-100 text-slate-700',
    issued: 'bg-blue-100 text-blue-700',
    sent: 'bg-blue-50 text-blue-600',
    paid: 'bg-green-100 text-green-700',
    overdue: 'bg-red-100 text-red-700',
    cancelled: 'bg-slate-100 text-slate-500',
  },
  // ... other types
};

// Usage
<StatusBadge status="paid" type="invoice" />
<StatusBadge status="closed_won" type="opportunity" showIcon />
```

#### MoneyDisplay

```typescript
// MoneyDisplay for consistent financial data presentation
interface MoneyDisplayProps {
  amount: number;
  currency: string;
  size?: 'sm' | 'md' | 'lg';
  align?: 'left' | 'right';
  showSign?: boolean;
  highlightNegative?: boolean;
  compact?: boolean; // $1.2M instead of $1,200,000
}

const MoneyDisplay: React.FC<MoneyDisplayProps> = ({
  amount,
  currency,
  size = 'md',
  align = 'left',
  showSign = false,
  highlightNegative = false,
  compact = false,
}) => {
  const formatted = compact 
    ? formatCompactCurrency(amount, currency)
    : formatCurrency(amount, currency);
  
  const isNegative = amount < 0;
  const colorClass = highlightNegative && isNegative 
    ? 'text-red-600' 
    : 'text-slate-900';
  
  return (
    <span 
      className={cn(
        'font-mono tabular-nums',
        size === 'sm' && 'text-sm',
        size === 'md' && 'text-base',
        size === 'lg' && 'text-xl font-semibold',
        align === 'right' && 'text-right',
        colorClass
      )}
    >
      {showSign && isNegative ? '-' : ''}
      {formatted}
    </span>
  );
};
```

### 4.3 Business Components

#### QuoteBuilder

```typescript
// QuoteBuilder - Multi-step quote creation
const QuoteBuilder: React.FC = () => {
  const [step, setStep] = useState<'details' | 'items' | 'review'>('details');
  
  return (
    <div className="max-w-5xl mx-auto">
      {/* Stepper */}
      <Stepper 
        steps={['Quote Details', 'Line Items', 'Review & Submit']}
        currentStep={stepIndex}
      />
      
      {/* Step Content */}
      <Card className="mt-6">
        {step === 'details' && <QuoteDetailsForm />}
        {step === 'items' && <QuoteLineItemsEditor />}
        {step === 'review' && <QuoteReviewPanel />}
      </Card>
      
      {/* Actions */}
      <div className="flex justify-between mt-6">
        <Button variant="outline" onClick={handleBack}>
          Back
        </Button>
        <div className="flex gap-3">
          <Button variant="outline" onClick={handleSaveDraft}>
            Save as Draft
          </Button>
          <Button 
            variant="primary" 
            onClick={handleNext}
            disabled={!isValid}
          >
            {step === 'review' ? 'Submit for Approval' : 'Next'}
          </Button>
        </div>
      </div>
    </div>
  );
};
```

#### RevenueChart

```typescript
// RevenueChart - Dashboard chart component
interface RevenueChartProps {
  data: RevenueDataPoint[];
  type: 'line' | 'bar' | 'area';
  timeRange: '7d' | '30d' | '90d' | '1y';
  metrics: ('mrr' | 'arr' | 'revenue')[];
  comparison?: 'previous_period' | 'yoy';
}

const RevenueChart: React.FC<RevenueChartProps> = ({
  data,
  type = 'area',
  timeRange,
  metrics,
  comparison,
}) => {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <div>
            <CardTitle>Revenue Overview</CardTitle>
            <CardDescription>
              {formatDateRange(timeRange)}
            </CardDescription>
          </div>
          <TimeRangeSelector value={timeRange} onChange={...} />
        </div>
      </CardHeader>
      
      <CardContent>
        {/* KPI Summary */}
        <div className="grid grid-cols-3 gap-4 mb-6">
          <KpiCard 
            label="MRR" 
            value={formatCurrency(data.currentMrr)} 
            change={data.mrrChange}
          />
          <KpiCard 
            label="ARR" 
            value={formatCurrency(data.currentArr)} 
            change={data.arrChange}
          />
          <KpiCard 
            label="Growth" 
            value={`${data.growthRate}%`} 
            change={data.growthChange}
          />
        </div>
        
        {/* Chart */}
        <ResponsiveContainer width="100%" height={300}>
          <RechartsComponent type={type}>
            {/* Chart configuration */}
          </RechartsComponent>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
};
```

---

## 5. Pages

### 5.1 Page Inventory

```
┌─────────────────────────────────────────────────────────────────┐
│                  PAGE INVENTORY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DASHBOARD                                                       │
│  ├─ /                           → Executive Dashboard           │
│  ├─ /dashboard/sales            → Sales Dashboard               │
│  ├─ /dashboard/finance          → Finance Dashboard             │
│  └─ /dashboard/operations       → Operations Dashboard          │
│                                                                  │
│  CRM / SALES                                                     │
│  ├─ /opportunities              → Opportunities list             │
│  ├─ /opportunities/:id          → Opportunity detail             │
│  ├─ /quotes                     → Quotes list                   │
│  ├─ /quotes/new                 → Create quote                  │
│  ├─ /quotes/:id                 → Quote detail                  │
│  ├─ /quotes/:id/edit            → Edit quote                    │
│  └─ /customers                  → Customers list                │
│                                                                  │
│  ORDERS & BILLING                                                │
│  ├─ /orders                     → Orders list                   │
│  ├─ /orders/:id                 → Order detail                  │
│  ├─ /invoices                   → Invoices list                 │
│  ├─ /invoices/:id               → Invoice detail                │
│  ├─ /invoices/:id/preview       → Invoice preview (print)       │
│  ├─ /payments                   → Payments list                 │
│  ├─ /payments/:id               → Payment detail                │
│  └─ /billing-schedules          → Billing schedules             │
│                                                                  │
│  CONTRACTS & ASSETS                                              │
│  ├─ /contracts                  → Contracts list                │
│  ├─ /contracts/:id              → Contract detail               │
│  ├─ /contracts/:id/edit         → Edit contract                 │
│  ├─ /assets                     → Assets list                   │
│  ├─ /assets/:id                 → Asset detail                  │
│  └─ /subscriptions              → Subscriptions list            │
│                                                                  │
│  PRODUCTS & CATALOG                                              │
│  ├─ /products                   → Products list                 │
│  ├─ /products/:id               → Product detail                │
│  ├─ /price-books                → Price books list              │
│  └─ /price-books/:id            → Price book detail             │
│                                                                  │
│  REPORTS & ANALYTICS                                             │
│  ├─ /reports                    → Reports hub                   │
│  ├─ /reports/revenue            → Revenue reports               │
│  ├─ /reports/sales              → Sales reports                 │
│  ├─ /reports/forecasting        → Forecasting reports           │
│  └─ /reports/custom             → Custom report builder         │
│                                                                  │
│  ADMINISTRATION                                                  │
│  ├─ /settings                   → Settings hub                  │
│  ├─ /settings/users             → User management               │
│  ├─ /settings/roles             → Role management               │
│  ├─ /settings/integrations      → Integrations                  │
│  ├─ /settings/billing           → Billing settings              │
│  ├─ /settings/notifications     → Notification preferences      │
│  └─ /settings/audit-log         → Audit log                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Page Layouts

```typescript
// Layout types
type PageLayout = 
  | 'dashboard'      // Full-width with widgets
  | 'list'           // Table-based list view
  | 'detail'         // Two-column detail view
  | 'form'           // Centered form
  | 'wizard'         // Multi-step wizard
  | 'split'          // Split view (list + detail)
  | 'empty';         // Empty state

// AppShell structure
const AppShell: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <div className="min-h-screen bg-slate-50">
    {/* Top Header */}
    <Header />
    
    <div className="flex">
      {/* Sidebar Navigation */}
      <Sidebar />
      
      {/* Main Content */}
      <main className="flex-1 overflow-auto">
        {/* Breadcrumb */}
        <Breadcrumb />
        
        {/* Page Content */}
        <div className="p-6">
          {children}
        </div>
      </main>
      
      {/* Right Panel (Contextual) */}
      <RightPanel />
    </div>
    
    {/* Global Overlays */}
    <CommandPalette />
    <ToastContainer />
    <ModalContainer />
  </div>
);
```

### 5.3 Key Pages Detail

#### Dashboard Page

```
┌─────────────────────────────────────────────────────────────────┐
│  [Logo]  [Search...]              [🔔] [👤 User] [⚙️]           │
├────────┬────────────────────────────────────────────────────────┤
│        │  Dashboard                          [Today ▼] [Export] │
│  📊    │                                                         │
│ Dash   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  🎯    │  │ MRR      │ │ ARR      │ │ New      │ │ Churn    │ │
│ Opps   │  │ $1.2M    │ │ $14.4M   │ │ $245K    │ │ 2.3%     │ │
│  📝    │  │ ▲ 12%    │ │ ▲ 15%    │ │ ▲ 8%     │ │ ▼ 0.5%   │ │
│ Quotes │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│  🛒    │                                                         │
│ Orders │  ┌────────────────────────┐ ┌──────────────────────┐  │
│  📄    │  │ Revenue Trend          │ │ Revenue by Product   │  │
│ Invoices│ │ [Line Chart]           │ │ [Pie Chart]          │  │
│  📑    │  │                        │ │                      │  │
│ Contracts│ │                        │ │                      │  │
│  📦    │  └────────────────────────┘ └──────────────────────┘  │
│ Assets │                                                         │
│  📈    │  ┌────────────────────────┐ ┌──────────────────────┐  │
│ Reports│  │ Top Deals              │ │ Recent Activity      │  │
│  ⚙️    │  │ [Table]                │ │ [Timeline]           │  │
│ Settings│ │                        │ │                      │  │
│        │  └────────────────────────┘ └──────────────────────┘  │
└────────┴────────────────────────────────────────────────────────┘
```

#### Opportunities List Page

```
┌─────────────────────────────────────────────────────────────────┐
│  Opportunities                              [+ New Opportunity] │
├─────────────────────────────────────────────────────────────────┤
│  [Search...]  [Stage ▼] [Owner ▼] [Date ▼]  [⚙️ Columns] [⇩]  │
├─────────────────────────────────────────────────────────────────┤
│  ☐ │ Name              │ Account      │ Stage      │ Amount    │
│  ──┼───────────────────┼──────────────┼────────────┼───────────│
│  ☐ │ Enterprise CRM    │ ACME Corp    │ [Proposal] │ $125,000  │
│  ☐ │ Cloud Migration   │ TechStart    │ [Discovery]│ $85,000   │
│  ☐ │ Analytics Suite   │ DataFlow Inc │ [Negotiat] │ $240,000  │
│  ☐ │ Security Platform │ SecureBank   │ [Qualific] │ $500,000  │
│  ☐ │ Mobile App        │ RetailCo     │ [Won ✓]    │ $75,000   │
│  ──┼───────────────────┼──────────────┼────────────┼───────────│
│                                                                  │
│  Showing 1-20 of 156        [< 1 2 3 ... 8 >]                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Quote Detail Page

```
┌─────────────────────────────────────────────────────────────────┐
│  ← Back to Quotes                                               │
│  QUO-2026-00456                    [Edit] [Clone] [⋮ More]     │
│  Status: [Approved ✓]    Expires: Jun 30, 2026                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────┐ ┌──────────────────────────┐ │
│  │ Quote Details                │ │ Customer                 │ │
│  │                              │ │                          │ │
│  │ Opportunity: Enterprise CRM  │ │ [Avatar] ACME Corp       │ │
│  │ Created by: John Doe         │ │ Industry: Technology     │ │
│  │ Created: Jun 15, 2026        │ │ Tier: Enterprise         │ │
│  │ Last modified: Jun 20, 2026  │ │ [View Customer →]        │ │
│  │                              │ │                          │ │
│  │ Line Items (4)         [$]   │ │ Contact                  │ │
│  │ ┌──────────────────────────┐ │ │ Jane Smith               │ │
│  │ │ Enterprise License × 100 │ │ │ jane@acme.com            │ │
│  │ │ $150,000                 │ │ │ +1 555-0123              │ │
│  │ ├──────────────────────────┤ │ │                          │ │
│  │ │ Premium Support × 1      │ │ │ Approval History         │ │
│  │ │ $25,000                  │ │ │ ✓ Approved by Manager    │ │
│  │ ├──────────────────────────┤ │ │   Jun 18, 2026           │ │
│  │ │ Implementation × 1       │ │ │ ✓ Approved by VP Sales   │ │
│  │ │ $50,000                  │ │ │   Jun 19, 2026           │ │
│  │ ├──────────────────────────┤ │ │                          │ │
│  │ │ Training Package × 1     │ │ │ Documents                │ │
│  │ │ $15,000                  │ │ │ 📄 Quote_Proposal.pdf    │ │
│  │ └──────────────────────────┘ │ │ 📄 Terms.pdf             │ │
│  │                              │ │                          │ │
│  │ Subtotal: $240,000           │ │                          │ │
│  │ Discount (5%): -$12,000      │ │                          │ │
│  │ Tax (7%): $15,960            │ │                          │ │
│  │ ─────────────────            │ │                          │ │
│  │ Total: $243,960              │ │                          │ │
│  └──────────────────────────────┘ └──────────────────────────┘ │
│                                                                  │
│  [Create Order →]  [Send to Customer]  [Download PDF]           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Responsive Design

### 6.1 Breakpoints

```css
/* Breakpoint system (mobile-first) */
:root {
  --breakpoint-sm: 640px;   /* Mobile landscape */
  --breakpoint-md: 768px;   /* Tablet */
  --breakpoint-lg: 1024px;  /* Small laptop */
  --breakpoint-xl: 1280px;  /* Desktop */
  --breakpoint-2xl: 1536px; /* Large desktop */
}

/* Tailwind configuration */
module.exports = {
  theme: {
    screens: {
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',
    },
  },
};
```

### 6.2 Device Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                  RESPONSIVE STRATEGY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MOBILE (< 768px)                                                │
│  ├─ Simplified navigation (hamburger menu)                      │
│  ├─ Single column layouts                                       │
│  ├─ Stacked cards instead of tables                             │
│  ├─ Bottom navigation for primary actions                       │
│  ├─ Touch-friendly targets (min 44px)                           │
│  └─ Swipe gestures for lists                                    │
│                                                                  │
│  TABLET (768px - 1024px)                                         │
│  ├─ Collapsible sidebar                                         │
│  ├─ Two-column layouts where appropriate                        │
│  ├─ Tables with horizontal scroll                               │
│  ├─ Floating action buttons                                     │
│  └─ Split views for detail pages                                │
│                                                                  │
│  DESKTOP (1024px+)                                               │
│  ├─ Full sidebar navigation                                     │
│  ├─ Multi-column layouts                                        │
│  ├─ Full data tables                                            │
│  ├─ Keyboard shortcuts                                          │
│  ├─ Hover states                                                │
│  └─ Context menus                                               │
│                                                                  │
│  LARGE DESKTOP (1536px+)                                         │
│  ├─ Wider content area                                          │
│  ├─ More data density option                                    │
│  ├─ Multi-panel layouts                                         │
│  └─ Advanced visualizations                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Responsive Patterns

```typescript
// Responsive DataTable
const ResponsiveTable = ({ columns, data }) => {
  const isMobile = useMediaQuery('(max-width: 768px)');
  
  if (isMobile) {
    return (
      <div className="space-y-3">
        {data.map(item => (
          <Card key={item.id}>
            <CardContent className="p-4">
              <div className="flex justify-between items-start mb-2">
                <div className="font-semibold">{item.name}</div>
                <StatusBadge status={item.status} type="opportunity" />
              </div>
              <div className="text-sm text-slate-600">{item.account}</div>
              <div className="flex justify-between items-center mt-3">
                <MoneyDisplay amount={item.amount} currency="USD" />
                <Button size="sm" variant="outline">View</Button>
              </div>
            </CardContent>
          </Card>
        ))}
      </div>
    );
  }
  
  return <DataTable columns={columns} data={data} />;
};

// Responsive Dashboard Grid
const DashboardGrid = ({ children }) => (
  <div className="grid gap-4 grid-cols-1 md:grid-cols-2 xl:grid-cols-3 2xl:grid-cols-4">
    {children}
  </div>
);

// Responsive Sidebar
const Sidebar = () => {
  const [collapsed, setCollapsed] = useState(false);
  const isMobile = useMediaQuery('(max-width: 768px)');
  
  if (isMobile) {
    return (
      <Sheet>
        <SheetTrigger asChild>
          <Button variant="ghost" size="icon">
            <MenuIcon />
          </Button>
        </SheetTrigger>
        <SheetContent side="left">
          <SidebarContent />
        </SheetContent>
      </Sheet>
    );
  }
  
  return (
    <aside className={cn(
      'transition-all duration-200',
      collapsed ? 'w-16' : 'w-64'
    )}>
      <SidebarContent collapsed={collapsed} />
    </aside>
  );
};
```

### 6.4 Touch & Interaction Guidelines

```yaml
Touch Targets:
  Minimum: 44x44px (iOS HIG)
  Recommended: 48x48px (Material Design)
  Spacing between targets: 8px minimum

Gestures:
  Swipe Right: Back / Dismiss
  Swipe Left: Actions menu
  Long Press: Context menu
  Pull Down: Refresh
  Pinch: Zoom (charts)

Desktop Enhancements:
  Keyboard Shortcuts:
    - Cmd/Ctrl + K: Command palette
    - Cmd/Ctrl + N: New record
    - Cmd/Ctrl + S: Save
    - Cmd/Ctrl + /: Show shortcuts
    - Esc: Close modal/drawer
    - ?: Show help
  
  Hover States:
    - Cards: Subtle elevation
    - Buttons: Color shift
    - Links: Underline
    - Table rows: Background tint
  
  Context Menus:
    - Right-click on records
    - Three-dot menu on mobile
```

---

## 7. Accessibility

### 7.1 WCAG 2.1 AA Compliance

```
┌─────────────────────────────────────────────────────────────────┐
│                  ACCESSIBILITY REQUIREMENTS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PERCEIVABLE                                                     │
│  ├─ 1.1 Text Alternatives: All images have alt text             │
│  ├─ 1.3 Adaptable: Content works with screen readers            │
│  ├─ 1.4 Distinguishable: Colors, contrast, text sizing          │
│  └─ All charts have data tables as alternatives                 │
│                                                                  │
│  OPERABLE                                                        │
│  ├─ 2.1 Keyboard Accessible: All features work via keyboard     │
│  ├─ 2.4 Navigable: Clear focus indicators, skip links           │
│  ├─ 2.5 Input Modalities: Voice, switch, touch support          │
│  └─ No keyboard traps, logical tab order                        │
│                                                                  │
│  UNDERSTANDABLE                                                  │
│  ├─ 3.1 Readable: Clear language, proper headings               │
│  ├─ 3.2 Predictable: Consistent navigation                      │
│  ├─ 3.3 Input Assistance: Error messages, labels                │
│  └─ Forms have clear instructions                               │
│                                                                  │
│  ROBUST                                                          │
│  ├─ 4.1 Compatible: Works with assistive technologies           │
│  ├─ Valid HTML, ARIA attributes where needed                    │
│  └─ Tested with NVDA, VoiceOver, JAWS                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Implementation Guidelines

```typescript
// Accessibility patterns

// 1. Focus Management
const Modal = ({ isOpen, onClose, children }) => {
  const previousFocus = useRef(document.activeElement);
  
  useEffect(() => {
    if (isOpen) {
      previousFocus.current = document.activeElement;
      // Focus first focusable element in modal
      const firstFocusable = modalRef.current.querySelector(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      firstFocusable?.focus();
    } else {
      previousFocus.current?.focus();
    }
  }, [isOpen]);
  
  return (
    <div 
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      onKeyDown={(e) => {
        if (e.key === 'Escape') onClose();
      }}
    >
      <h2 id="modal-title">Dialog Title</h2>
      {children}
    </div>
  );
};

// 2. Form Accessibility
const FormField = ({ label, error, required, ...props }) => (
  <div>
    <label htmlFor={props.id} className="...">
      {label}
      {required && <span aria-hidden="true" className="text-red-500">*</span>}
      {required && <span className="sr-only">(required)</span>}
    </label>
    <input 
      {...props}
      aria-required={required}
      aria-invalid={!!error}
      aria-describedby={error ? `${props.id}-error` : undefined}
    />
    {error && (
      <p id={`${props.id}-error`} role="alert" className="text-red-600 text-sm">
        {error}
      </p>
    )}
  </div>
);

// 3. Data Table Accessibility
const AccessibleTable = ({ columns, data }) => (
  <table role="table" aria-label="Invoices">
    <caption className="sr-only">List of customer invoices</caption>
    <thead>
      <tr>
        {columns.map(col => (
          <th 
            key={col.key}
            scope="col"
            aria-sort={col.sortDirection}
          >
            <button onClick={() => handleSort(col.key)}>
              {col.label}
              {col.sortDirection === 'ascending' && <span aria-hidden>↑</span>}
              {col.sortDirection === 'descending' && <span aria-hidden>↓</span>}
              <span className="sr-only">
                , sorted {col.sortDirection}
              </span>
            </button>
          </th>
        ))}
      </tr>
    </thead>
    <tbody>
      {data.map(row => (
        <tr key={row.id}>
          {columns.map(col => (
            <td key={col.key}>{row[col.key]}</td>
          ))}
        </tr>
      ))}
    </tbody>
  </table>
);

// 4. Chart Accessibility
const AccessibleChart = ({ data, title, description }) => (
  <figure>
    <figcaption>{title}</figcaption>
    <p className="sr-only">{description}</p>
    <div aria-hidden="true">
      {/* Visual chart */}
      <RechartsComponent data={data} />
    </div>
    <details>
      <summary>View data table</summary>
      <table>
        {/* Data table alternative */}
      </table>
    </details>
  </figure>
);

// 5. Live Regions for Dynamic Content
const Toast = ({ message, type }) => (
  <div 
    role="status" 
    aria-live="polite"
    aria-atomic="true"
    className={cn(
      'toast',
      type === 'error' && 'role="alert" aria-live="assertive"'
    )}
  >
    {message}
  </div>
);
```

### 7.3 Testing Checklist

```yaml
Accessibility Testing:
  Automated:
    - [ ] axe-core scans (CI/CD)
    - [ ] Lighthouse accessibility score > 90
    - [ ] HTML validation (no errors)
    - [ ] Color contrast checks
  
  Manual:
    - [ ] Keyboard navigation (all features)
    - [ ] Screen reader testing (VoiceOver, NVDA)
    - [ ] Zoom to 200% (layout still works)
    - [ ] High contrast mode
    - [ ] Reduced motion preference
    - [ ] Focus indicators visible
    - [ ] Form labels and errors clear
    - [ ] Images have alt text
    - [ ] Videos have captions
  
  User Testing:
    - [ ] Test with users with disabilities
    - [ ] Feedback incorporation
    - [ ] Regular accessibility audits
```

---

## 8. User Flow

### 8.1 Primary User Flows

```
┌─────────────────────────────────────────────────────────────────┐
│                  USER FLOW: CREATE QUOTE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Sales Rep Dashboard]                                           │
│         │                                                        │
│         ▼                                                        │
│  [Click "+ New Quote"]                                           │
│         │                                                        │
│         ▼                                                        │
│  [Select Customer] ──(search)──> [Customer List]                 │
│         │                              │                         │
│         │<─────────────────────────────┘                         │
│         ▼                                                        │
│  [Quote Details Form]                                            │
│   ├─ Quote Name                                                  │
│   ├─ Opportunity (link)                                          │
│   ├─ Expiration Date                                             │
│   └─ Notes                                                       │
│         │                                                        │
│         ▼                                                        │
│  [Add Line Items]                                                │
│   ├─ Search product                                              │
│   ├─ Set quantity                                                │
│   ├─ Apply discount                                              │
│   └─ Review pricing                                              │
│         │                                                        │
│         ▼                                                        │
│  [Review Quote]                                                  │
│   ├─ See total                                                   │
│   ├─ Preview PDF                                                 │
│   └─ Check terms                                                 │
│         │                                                        │
│         ▼                                                        │
│  [Submit for Approval]                                           │
│         │                                                        │
│         ▼                                                        │
│  [Manager Approval]                                              │
│   ├─ Review details                                              │
│   ├─ Check discount                                              │
│   └─ Approve / Reject                                            │
│         │                                                        │
│         ▼                                                        │
│  [Quote Approved]                                                │
│   ├─ Notify sales rep                                            │
│   ├─ Send to customer                                            │
│   └─ Create order (next step)                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Order-to-Cash Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  USER FLOW: ORDER-TO-CASH                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │ Quote    │───▶│ Order    │───▶│ Invoice  │                  │
│  │ Approved │    │ Created  │    │ Generated│                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                       │                          │
│                                       ▼                          │
│                                  ┌──────────┐                   │
│                                  │ Send to  │                   │
│                                  │ Customer │                   │
│                                  └──────────┘                   │
│                                       │                          │
│                                       ▼                          │
│                                  ┌──────────┐                   │
│                                  │ Payment  │                   │
│                                  │ Received │                   │
│                                  └──────────┘                   │
│                                       │                          │
│                                       ▼                          │
│                                  ┌──────────┐                   │
│                                  │ Revenue  │                   │
│                                  │ Recognized│                  │
│                                  └──────────┘                   │
│                                                                  │
│  Parallel Activities:                                            │
│  ├─ Assets generated when order activated                        │
│  ├─ Contract created/linked                                      │
│  ├─ Billing schedule set up                                      │
│  └─ Notifications sent at each step                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 State Machines

```typescript
// Opportunity State Machine
const opportunityStates = {
  initial: 'qualification',
  states: {
    qualification: {
      on: {
        ADVANCE: 'discovery',
        CLOSE_LOST: 'closed_lost',
      },
    },
    discovery: {
      on: {
        ADVANCE: 'proposal',
        CLOSE_LOST: 'closed_lost',
      },
    },
    proposal: {
      on: {
        ADVANCE: 'negotiation',
        CLOSE_LOST: 'closed_lost',
      },
    },
    negotiation: {
      on: {
        CLOSE_WON: 'closed_won',
        CLOSE_LOST: 'closed_lost',
      },
    },
    closed_won: { type: 'final' },
    closed_lost: { type: 'final' },
  },
};

// Invoice State Machine
const invoiceStates = {
  initial: 'draft',
  states: {
    draft: {
      on: {
        ISSUE: 'issued',
        CANCEL: 'cancelled',
      },
    },
    issued: {
      on: {
        SEND: 'sent',
        CANCEL: 'cancelled',
      },
    },
    sent: {
      on: {
        PAY: 'paid',
        OVERDUE: 'overdue',
        VOID: 'cancelled',
      },
    },
    overdue: {
      on: {
        PAY: 'paid',
        VOID: 'cancelled',
      },
    },
    paid: { type: 'final' },
    cancelled: { type: 'final' },
  },
};
```

---

## 9. Wireframes

### 9.1 Dashboard Wireframe

```
┌─────────────────────────────────────────────────────────────────┐
│  ☰  Revenue Management       [🔍 Search...]  [🔔] [👤] [⚙️]    │
├────┬────────────────────────────────────────────────────────────┤
│    │                                                            │
│ 📊 │  Dashboard                              [This Month ▼]    │
│ 🎯 │                                                            │
│ 📝 │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │
│ 🛒 │  │ MRR    │ │ ARR    │ │ New    │ │ Churn  │            │
│ 📄 │  │ $1.2M  │ │ $14.4M │ │ $245K  │ │ 2.3%   │            │
│ 📑 │  │ ▲12%   │ │ ▲15%   │ │ ▲8%    │ │ ▼0.5%  │            │
│ 📦 │  └────────┘ └────────┘ └────────┘ └────────┘            │
│ 📈 │                                                            │
│ ⚙️ │  ┌────────────────────────┐ ┌──────────────────────┐     │
│    │  │ Revenue Trend          │ │ Revenue by Product   │     │
│    │  │                        │ │                      │     │
│    │  │    ╱╲                  │ │    ╭──╮              │     │
│    │  │   ╱  ╲    ╱╲          │ │   ╱    ╲             │     │
│    │  │  ╱    ╲  ╱  ╲         │ │  │      │            │     │
│    │  │ ╱      ╲╱    ╲        │ │  ╰──────╯            │     │
│    │  │╱                    │ │                      │     │
│    │  └────────────────────────┘ └──────────────────────┘     │
│    │                                                            │
│    │  ┌────────────────────────┐ ┌──────────────────────┐     │
│    │  │ Top Opportunities      │ │ Recent Activities    │     │
│    │  │                        │ │                      │     │
│    │  │ 1. Enterprise CRM     │ │ • John closed deal   │     │
│    │  │    $125K  [Proposal]   │ │ • Invoice #123 paid  │     │
│    │  │ 2. Cloud Migration    │ │ • New quote created  │     │
│    │  │    $85K   [Discovery]  │ │ • Contract renewed   │     │
│    │  │ 3. Analytics Suite    │ │                      │     │
│    │  │    $240K  [Negotiat]   │ │ [View all →]         │     │
│    │  └────────────────────────┘ └──────────────────────┘     │
│    │                                                            │
└────┴────────────────────────────────────────────────────────────┘
```

### 9.2 Quote Builder Wireframe

```
┌─────────────────────────────────────────────────────────────────┐
│  ← Back to Quotes                                               │
│  New Quote                                   Step 2 of 3        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① Details    ② Line Items    ③ Review                          │
│  ─────────    ─────────────    ────────                          │
│   ✓ Done       ● Active         Pending                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  Customer: ACME Corp                          [Change]   │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │ [🔍 Search products...]                            │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │ ☐ │ Product              │ Qty │ Price  │ Total   │ │  │
│  │  │ ──┼──────────────────────┼─────┼────────┼──────── │ │  │
│  │  │ ☐ │ Enterprise License   │ 100 │ $1,500 │ $150K  │ │  │
│  │  │ ☐ │ Premium Support      │  1  │ $25K   │ $25K   │ │  │
│  │  │ ☐ │ Implementation       │  1  │ $50K   │ $50K   │ │  │
│  │  │ ☐ │ Training Package     │  1  │ $15K   │ $15K   │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  │  [+ Add Line Item]  [📦 Bundle]  [💰 Apply Discount]    │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │ Subtotal:                          $240,000.00     │ │  │
│  │  │ Discount (5%):                    -$12,000.00      │ │  │
│  │  │ Tax (7%):                          $15,960.00      │ │  │
│  │  │ ──────────────────────────────────────────────     │ │  │
│  │  │ Total:                             $243,960.00     │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  [← Back]                              [Save Draft] [Next →]   │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 Invoice Detail Wireframe

```
┌─────────────────────────────────────────────────────────────────┐
│  ← Back to Invoices                                             │
│  INV-2026-00123                  [Edit] [Send] [⋮ More]         │
│  Status: [Paid ✓]    Due: Jun 15, 2026    Paid: Jun 10, 2026   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────┐ ┌──────────────────────────┐ │
│  │ Invoice Details              │ │ Customer                 │ │
│  │                              │ │                          │ │
│  │ Order: ORD-2026-00456        │ │ [Avatar] ACME Corp       │ │
│  │ Billing Period: Jun 2026     │ │ Enterprise Tier          │ │
│  │ Currency: USD                │ │                          │ │
│  │                              │ │ Billing Address:         │ │
│  │ Line Items (4)               │ │ 123 Business Ave         │ │
│  │ ┌──────────────────────────┐ │ │ Suite 100                │ │
│  │ │ Enterprise License × 100 │ │ │ New York, NY 10001       │ │
│  │ │ $150,000                 │ │ │                          │ │
│  │ ├──────────────────────────┤ │ │ Payment History          │ │
│  │ │ Premium Support × 1      │ │ │ ✓ Jun 10: $243,960      │ │
│  │ │ $25,000                  │ │ │   Credit Card ****1234   │ │
│  │ ├──────────────────────────┤ │ │                          │ │
│  │ │ Implementation × 1       │ │ │ Related Documents        │ │
│  │ │ $50,000                  │ │ │ 📄 Invoice.pdf           │ │
│  │ ├──────────────────────────┤ │ │ 📄 Receipt.pdf           │ │
│  │ │ Training × 1             │ │ │ 📄 Contract.pdf          │ │
│  │ │ $15,000                  │ │ │                          │ │
│  │ └──────────────────────────┘ │ │                          │ │
│  │                              │ │                          │ │
│  │ Subtotal: $240,000           │ │                          │ │
│  │ Discount: -$12,000           │ │                          │ │
│  │ Tax: $15,960                 │ │                          │ │
│  │ ─────────────────            │ │                          │ │
│  │ Total: $243,960              │ │                          │ │
│  │ Paid: -$243,960              │ │                          │ │
│  │ ─────────────────            │ │                          │ │
│  │ Balance Due: $0              │ │                          │ │
│  └──────────────────────────────┘ └──────────────────────────┘ │
│                                                                  │
│  Timeline:                                                       │
│  ─────────                                                       │
│  ● Jun 1  │ Invoice created by John Doe                          │
│  ● Jun 2  │ Invoice sent to customer                             │
│  ● Jun 5  │ Reminder sent (3 days before due)                    │
│  ● Jun 10 │ Payment received ($243,960)                          │
│  ● Jun 10 │ Invoice marked as paid                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Mockups

### 10.1 Visual Design Direction

```
┌─────────────────────────────────────────────────────────────────┐
│                  VISUAL DESIGN DIRECTION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STYLE: Modern Professional                                      │
│  ├─ Clean, minimal interface                                     │
│  ├─ Generous whitespace                                          │
│  ├─ Subtle shadows for depth                                     │
│  ├─ Rounded corners (8-12px)                                     │
│  └─ Smooth animations (200-300ms)                                │
│                                                                  │
│  INSPIRATION:                                                    │
│  ├─ Linear.app (clean SaaS)                                      │
│  ├─ Stripe Dashboard (financial clarity)                         │
│  ├─ Notion (flexible layouts)                                    │
│  ├─ Vercel (modern developer tools)                              │
│  └─ Salesforce Lightning (enterprise features)                   │
│                                                                  │
│  MOOD:                                                           │
│  ├─ Professional but approachable                                │
│  ├─ Trustworthy and reliable                                     │
│  ├─ Efficient and productive                                     │
│  └─ Modern and forward-thinking                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Component Mockups (ASCII)

#### Login Screen

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                                                                  │
│                     ┌─────────────────────┐                     │
│                     │                     │                     │
│                     │   [Logo: RM]        │                     │
│                     │                     │                     │
│                     │  Welcome Back       │                     │
│                     │  Sign in to your    │                     │
│                     │  account            │                     │
│                     │                     │                     │
│                     │  Email              │                     │
│                     │  ┌───────────────┐ │                     │
│                     │  │               │ │                     │
│                     │  └───────────────┘ │                     │
│                     │                     │                     │
│                     │  Password           │                     │
│                     │  ┌───────────────┐ │                     │
│                     │  │               │ │                     │
│                     │  └───────────────┘ │                     │
│                     │                     │                     │
│                     │  ☐ Remember me      │                     │
│                     │                     │                     │
│                     │  ┌───────────────┐ │                     │
│                     │  │   Sign In     │ │                     │
│                     │  └───────────────┘ │                     │
│                     │                     │                     │
│                     │  Forgot password?   │                     │
│                     │                     │                     │
│                     └─────────────────────┘                     │
│                                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Dashboard Mockup

```
┌─────────────────────────────────────────────────────────────────┐
│  ═══  Revenue Management   🔍 Search...   🔔 3  👤 John  ⚙️    │
├────┬────────────────────────────────────────────────────────────┤
│    │                                                            │
│ 📊 │  Dashboard                              [June 2026 ▼]     │
│    │                                                            │
│ 🎯 │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│ 📝 │  │ ╔═══════╗   │ │ ╔═══════╗   │ │ ╔═══════╗   │         │
│ 🛒 │  │ ║ $1.2M ║   │ │ ║ $14.4M║   │ │ ║ $245K ║   │         │
│ 📄 │  │ ╚═══════╝   │ │ ╚═══════╝   │ │ ╚═══════╝   │         │
│ 📑 │  │ MRR         │ │ ARR         │ │ New Revenue │         │
│ 📦 │  │ ▲ 12% vs lm │ │ ▲ 15% vs lm │ │ ▲ 8% vs lm  │         │
│ 📈 │  └─────────────┘ └─────────────┘ └─────────────┘         │
│    │                                                            │
│ ⚙️ │  ┌────────────────────────────────────────────────────┐  │
│    │  │ Revenue Trend                                      │  │
│    │  │                                                    │  │
│    │  │  $1.5M ┤                              ╭──╮         │  │
│    │  │        │                         ╭───╯  ╰───╮      │  │
│    │  │  $1.0M ┤                    ╭───╯            ╰──   │  │
│    │  │        │              ╭─────╯                       │  │
│    │  │  $0.5M ┤         ╭───╯                              │  │
│    │  │        │    ╭────╯                                  │  │
│    │  │  $0    ┼────╯                                       │  │
│    │  │        Jan  Feb  Mar  Apr  May  Jun  Jul  Aug       │  │
│    │  └────────────────────────────────────────────────────┘  │
│    │                                                            │
│    │  ┌──────────────────────────┐ ┌──────────────────────┐   │
│    │  │ Top Opportunities        │ │ Recent Activity      │   │
│    │  │                          │ │                      │   │
│    │  │ 🎯 Enterprise CRM        │ │ ● 2m ago           │   │
│    │  │    ACME Corp • $125K     │ │   John closed deal  │   │
│    │  │    [Proposal]            │ │   with TechStart    │   │
│    │  │                          │ │                      │   │
│    │  │ 🎯 Cloud Migration       │ │ ● 1h ago           │   │
│    │  │    TechStart • $85K      │ │   Invoice #123      │   │
│    │  │    [Discovery]           │ │   paid by ACME      │   │
│    │  │                          │ │                      │   │
│    │  │ 🎯 Analytics Suite       │ │ ● 3h ago           │   │
│    │  │    DataFlow • $240K      │ │   New quote created │   │
│    │  │    [Negotiation]         │ │   for SecureBank    │   │
│    │  │                          │ │                      │   │
│    │  │ [View all →]             │ │ [View all →]        │   │
│    │  └──────────────────────────┘ └──────────────────────┘   │
│    │                                                            │
└────┴────────────────────────────────────────────────────────────┘
```

### 10.3 Figma Structure

```yaml
Figma File Organization:
  
  📁 Revenue Management
  │
  ├── 📄 Cover
  │
  ├── 📁 Design System
  │   ├── 📄 Colors
  │   ├── 📄 Typography
  │   ├── 📄 Icons
  │   ├── 📄 Spacing
  │   ├── 📄 Components - Buttons
  │   ├── 📄 Components - Inputs
  │   ├── 📄 Components - Cards
  │   ├── 📄 Components - Tables
  │   ├── 📄 Components - Modals
  │   └── 📄 Components - Charts
  │
  ├── 📁 Flows
  │   ├── 📄 Authentication Flow
  │   ├── 📄 Opportunity Flow
  │   ├── 📄 Quote Flow
  │   ├── 📄 Order Flow
  │   ├── 📄 Invoice Flow
  │   └── 📄 Contract Flow
  │
  ├── 📁 Pages - Desktop
  │   ├── 📄 Dashboard
  │   ├── 📄 Opportunities - List
  │   ├── 📄 Opportunities - Detail
  │   ├── 📄 Quotes - List
  │   ├── 📄 Quotes - Builder
  │   ├── 📄 Quotes - Detail
  │   ├── 📄 Orders - List
  │   ├── 📄 Orders - Detail
  │   ├── 📄 Invoices - List
  │   ├── 📄 Invoices - Detail
  │   ├── 📄 Invoices - Preview
  │   ├── 📄 Contracts - List
  │   ├── 📄 Contracts - Detail
  │   ├── 📄 Assets - List
  │   ├── 📄 Products - List
  │   ├── 📄 Reports - Revenue
  │   └── 📄 Settings - Users
  │
  ├── 📁 Pages - Mobile
  │   ├── 📄 Dashboard
  │   ├── 📄 List Views
  │   └── 📄 Detail Views
  │
  ├── 📁 Prototypes
  │   ├── 📄 Quote Creation Flow
  │   ├── 📄 Order Activation Flow
  │   └── 📄 Invoice Payment Flow
  │
  └── 📁 Assets
      ├── 📄 Icons
      ├── 📄 Illustrations
      └── 📄 Empty States

Figma Variables:
  - Colors (Light + Dark mode)
  - Spacing (4px grid)
  - Radius (sm, md, lg, xl)
  - Shadows (sm, md, lg)
  - Typography (all scales)
  
Figma Components:
  - All UI components as variants
  - Auto-layout enabled
  - Responsive constraints
  - Interactive components (hover, focus, active)
```

### 10.4 AI Design-to-Code Prompts

```typescript
// Prompts for AI design tools (v0, Galileo AI, etc.)

// Example prompt for dashboard page
const dashboardPrompt = `
Create a modern SaaS dashboard for a Revenue Management system.

Layout:
- Fixed left sidebar (240px) with navigation icons + labels
- Top header with search, notifications, user menu
- Main content area with responsive grid

Content:
- 4 KPI cards at top (MRR, ARR, New Revenue, Churn)
  - Each shows value, trend indicator, percentage change
  - Use green for positive, red for negative
- Large area chart showing revenue trend (6 months)
- Pie chart showing revenue by product
- Table of top opportunities (5 rows)
- Timeline of recent activities (5 items)

Style:
- Clean, minimal, professional
- Primary color: Blue (#3B82F6)
- Background: Light gray (#F8FAFC)
- Cards: White with subtle shadow
- Typography: Inter font family
- Financial numbers: Monospace, tabular alignment
- Rounded corners: 8-12px
- Generous whitespace

Interactions:
- Hover states on cards (subtle elevation)
- Clickable chart points (tooltip)
- Sortable table columns
- Time range selector
`;

// Example prompt for quote builder
const quoteBuilderPrompt = `
Create a multi-step quote builder wizard.

Steps:
1. Quote Details (customer, expiration, notes)
2. Line Items (product selection, quantities, pricing)
3. Review & Submit (summary, approval)

Step 2 (Line Items) layout:
- Top: Customer info card
- Middle: Searchable product table
  - Columns: Checkbox, Product, Qty, Price, Discount, Total
  - Inline editing for quantity and discount
  - Running total at bottom
- Bottom: Action buttons (Add item, Bundle, Discount)
- Footer: Back, Save Draft, Next buttons

Style:
- Stepper at top showing progress
- Clean form inputs with labels
- Table with zebra striping
- Money values right-aligned, monospace
- Status badges for product categories
- Clear visual hierarchy

Accessibility:
- Keyboard navigation for table
- ARIA labels for all inputs
- Focus indicators visible
- Error messages inline
`;
```

### 10.5 Design Handoff Checklist

```yaml
Design Handoff to Development:
  
  Documentation:
    - [ ] All components documented in Storybook
    - [ ] Design tokens exported (JSON/CSS variables)
    - [ ] Figma variables synced with code
    - [ ] Component API documented (props, states)
    - [ ] Interaction specifications (hover, focus, active)
    - [ ] Animation specifications (duration, easing)
  
  Assets:
    - [ ] Icons exported as SVG (optimized)
    - [ ] Illustrations exported (SVG + PNG fallback)
    - [ ] Images optimized (WebP + fallback)
    - [ ] Fonts licensed and available
    - [ ] Empty state illustrations
  
  Specifications:
    - [ ] Responsive breakpoints defined
    - [ ] Spacing system documented
    - [ ] Color usage guidelines
    - [ ] Typography scale
    - [ ] Accessibility requirements
    - [ ] Error states defined
    - [ ] Loading states defined
    - [ ] Empty states defined
  
  Prototypes:
    - [ ] Key user flows prototyped
    - [ ] Interactive components tested
    - [ ] Animation timing validated
    - [ ] Responsive behavior tested
  
  Review:
    - [ ] Design review with stakeholders
    - [ ] Accessibility audit
    - [ ] Technical feasibility review
    - [ ] Performance considerations
    - [ ] Browser compatibility check
```

---

## ภาคผนวก

### ภาคผนวก A: Design System Tools

```yaml
Tools & Technologies:
  
  Design:
    - Figma (primary design tool)
    - FigJam (brainstorming, flows)
    - Figma Variables (design tokens)
    - Figma Dev Mode (handoff)
  
  Documentation:
    - Storybook (component documentation)
    - Zeroheight (design system docs)
    - Notion (design decisions)
  
  Code:
    - Tailwind CSS (utility-first styling)
    - Radix UI (accessible primitives)
    - shadcn/ui (component library)
    - Lucide React (icons)
    - Recharts (data visualization)
  
  Testing:
    - Chromatic (visual regression)
    - Percy (visual testing)
    - axe-core (accessibility)
    - Lighthouse (performance)
  
  Collaboration:
    - Figma comments
    - Slack integration
    - Jira integration
    - GitHub integration
```

### ภาคผนวก B: Design Review Process

```yaml
Design Review Checklist:
  
  Pre-Review:
    - [ ] Requirements understood
    - [ ] User research conducted
    - [ ] Competitive analysis done
    - [ ] Initial sketches created
  
  During Review:
    - [ ] Design aligns with brand
    - [ ] Accessibility standards met
    - [ ] Responsive design considered
    - [ ] Edge cases handled
    - [ ] Error states designed
    - [ ] Loading states designed
    - [ ] Empty states designed
    - [ ] Interactions defined
    - [ ] Animations specified
  
  Post-Review:
    - [ ] Feedback incorporated
    - [ ] Final designs approved
    - [ ] Assets exported
    - [ ] Documentation updated
    - [ ] Handoff to development
    - [ ] QA criteria defined
```

### ภาคผนวก C: Component API Examples

```typescript
// Example component APIs for developers

// Button
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
  size: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  icon?: React.ReactNode;
  iconPosition?: 'left' | 'right';
  loading?: boolean;
  disabled?: boolean;
  fullWidth?: boolean;
  onClick?: () => void;
  type?: 'button' | 'submit' | 'reset';
}

// Input
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
  hint?: string;
  icon?: React.ReactNode;
  iconPosition?: 'left' | 'right';
  loading?: boolean;
}

// DataTable
interface DataTableProps<T> {
  columns: ColumnDef<T>[];
  data: T[];
  pagination?: PaginationConfig;
  sorting?: SortingConfig;
  filters?: FilterConfig[];
  selectable?: boolean;
  onSelectionChange?: (ids: string[]) => void;
  rowActions?: RowAction<T>[];
  emptyState?: React.ReactNode;
  loading?: boolean;
}

// Modal
interface ModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title?: string;
  description?: string;
  children: React.ReactNode;
  size?: 'sm' | 'md' | 'lg' | 'xl' | 'full';
  closeOnOverlayClick?: boolean;
  closeOnEscape?: boolean;
}

// Toast
interface ToastProps {
  title: string;
  description?: string;
  variant?: 'default' | 'success' | 'warning' | 'error';
  duration?: number;
  action?: {
    label: string;
    onClick: () => void;
  };
}
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการออกแบบ UX/UI สำหรับระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** UX/UI Design Team
