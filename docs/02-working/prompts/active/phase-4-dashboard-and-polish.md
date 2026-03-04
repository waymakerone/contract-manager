---
sync:
  type: doc
  layer: Contract Manager
build:
  status: todo
  phase: 4
  priority: P0
  depends_on: ["phase-3-reminders-and-contacts"]
  started_at: null
  completed_at: null
---

# Phase 4: Dashboard & Polish

**Goal:** Executive dashboard with contract metrics and charts, value tracking, CSV export, document bulk download, mobile-responsive layouts, and final polish including waymaker.config.ts manifest.

**PRD Reference:** `docs/01-planning/product-requirements/contract-manager-prd.md` — Phase 4

---

## What to Build

### 1. Executive Dashboard (/dashboard)

A summary view pulling together all contract data for leadership visibility.

**Layout — grid of cards:**

**Row 1: Key Metrics (4 cards)**
- Total Active Contracts (count of status = active)
- Total Active Value (sum of value for active contracts, formatted currency)
- Expiring in 30 Days (count, amber if > 0, link to filtered register)
- Overdue Milestones (count, red if > 0, link to milestone view)

**Row 2: Expiring Soon (3 cards — 30/60/90 days)**
- Expiring in 30 Days: list of contracts with counterparty, value, end date, action buttons
- Expiring in 60 Days: same format
- Expiring in 90 Days: same format
- Each card shows urgency color: red (30d), amber (60d), default (90d)

**Row 3: Charts (2 cards, equal width)**
- Contracts by Type (donut chart, coloured by type colour)
  - Each slice: type name + count
  - Center text: total count
  - Click slice: filter register by type
  - Use `recharts` PieChart
- Contracts by Department (horizontal bar chart)
  - Each bar: department name + count + total value
  - Sorted by count descending
  - Use `recharts` BarChart

**Row 4: Renewal Timeline (full width)**
- Timeline chart showing renewals over the next 12 months
- X-axis: months (next 12)
- Y-axis: count of renewals per month
- Bar chart with contract count per month
- Hover: show contract names expiring that month
- Use `recharts` BarChart

**Row 5: Recent Activity (full width)**
- Last 10 contracts added or modified
- Columns: contract title, counterparty, status change, date, by whom
- Click row → navigate to contract detail

**Row 6: Quick Actions**
- "Add Contract" button
- "View Register" button
- "View Timeline" button
- "Export All" button (CSV)

### 2. Value Tracking

Aggregate financial metrics across all contracts.

**Value metrics (on dashboard and register summary bar):**
- Total Committed Value: sum of `value` for all active contracts
- Annual Value: estimated annual value (total value / contract duration in years, or value for 12-month contracts)
- Value by Type: breakdown of total value per contract type
- Value by Department: breakdown of total value per department

**Value calculations:**
```typescript
interface ValueSummary {
  totalActiveValue: number;
  totalCommittedValue: number;  // includes draft
  annualEstimate: number;
  byType: { typeId: string; typeName: string; colour: string; value: number; count: number }[];
  byDepartment: { department: string; value: number; count: number }[];
  currency: string;  // primary currency (most common)
}

function calculateValueSummary(contracts: Contract[], types: ContractType[]): ValueSummary {
  // Sum values for active contracts
  // Calculate annual estimate: for each contract, value / (duration in years)
  // Group by type and department
  // Handle null values (exclude from sums)
}
```

**Value display:**
- Format with currency symbol and thousands separators
- Show change vs previous period where applicable
- Handle multi-currency: display in primary currency with note "Values in multiple currencies"

### 3. Contract Comparison

A view for comparing contracts side by side, particularly useful during renewal decisions.

**Comparison view (accessible from "Expiring Soon" cards):**
- Side-by-side display of current contract terms
- Fields: title, counterparty, value, start date, end date, type, department
- Placeholder for "New Terms" column (manual entry — actual negotiation happens outside the app)
- "Renew with New Terms" action → creates new contract

**Implementation (keep simple for V1):**
- Modal or slide-out panel
- Current contract details on the left
- "Notes for Renewal" textarea on the right
- Action buttons: Renew, Terminate, Extend

### 4. CSV Export

Export contracts as CSV files for finance, legal, or compliance reporting.

**Export options:**
- "Export All Contracts" — all contracts matching current filters
- "Export Active Contracts" — only active status
- "Export Expiring Contracts" — contracts expiring within a configurable window
- Export from register page via "Export" button
- Export from dashboard via "Export All" button

**CSV format for contracts:**
```
Contract #,Title,Type,Counterparty,Status,Value,Currency,Start Date,End Date,Renewal Date,Auto Renew,Notice Period (Days),Owner,Department,Tags,Created
CON-0001,Cloud Services Agreement,SaaS Agreement,Acme Corp,Active,24000,USD,2025-03-01,2026-03-01,2026-03-01,No,30,Jane Smith,Engineering,"cloud,infrastructure",2025-02-15
```

**Implementation:**
```typescript
function exportContractsToCSV(contracts: Contract[], types: ContractType[]): void {
  const headers = [
    'Contract #', 'Title', 'Type', 'Counterparty', 'Status',
    'Value', 'Currency', 'Start Date', 'End Date', 'Renewal Date',
    'Auto Renew', 'Notice Period (Days)', 'Owner', 'Department', 'Tags', 'Created'
  ];

  const rows = contracts.map(c => {
    const type = types.find(t => t.id === c.type_id);
    return [
      c.contract_number, c.title, type?.name || '', c.counterparty, c.status,
      c.value?.toString() || '', c.currency, c.start_date, c.end_date || 'Evergreen',
      c.renewal_date || '', c.auto_renew ? 'Yes' : 'No',
      c.notice_period_days?.toString() || '', c.owner_id, c.department || '',
      (c.tags || []).join(';'), c.created_at
    ];
  });

  const csv = [headers, ...rows].map(row =>
    row.map(cell => `"${String(cell).replace(/"/g, '""')}"`).join(',')
  ).join('\n');

  const blob = new Blob([csv], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `contracts-export-${new Date().toISOString().split('T')[0]}.csv`;
  a.click();
  URL.revokeObjectURL(url);
}
```

**Toast notification:** "Exported X contracts to CSV"

### 5. Document Bulk Download

Download multiple contract documents at once.

**Bulk download flow:**
1. User selects multiple contracts in the register (checkbox column)
2. "Download Documents" button appears in bulk action bar
3. Downloads each document from Supabase Storage
4. For single document: direct download
5. For multiple: download as individual files (or zip if a zip library is available)

**Implementation:**
- Use Supabase Storage download URLs
- For each selected contract with a document_url: trigger download
- Show progress: "Downloading 3 of 5 documents..."
- Skip contracts with no document attached

### 6. Mobile-Responsive Layouts

Ensure all views work well on mobile devices.

**Contract register:**
- Card view on mobile (instead of table)
- Each card: title + counterparty (prominent), status badge + type badge + value (secondary), end date with "X days" countdown
- Floating action button: "+" to add contract
- Pull to refresh

**Contract detail:**
- Single column layout on mobile
- Document preview: tap to open full-screen
- Milestones: stacked list view
- Action buttons: fixed bottom bar

**Dashboard:**
- Single column layout
- Charts full-width
- Key metrics: 2x2 grid
- Expiring soon: stacked cards

**Calendar/timeline:**
- Calendar: simplified month view with day dots
- Timeline: horizontal scroll with touch support
- Expiring panel: bottom sheet on mobile

**Add/edit contract modal:**
- Full-screen on mobile
- Document upload zone: full-width, tap to browse files
- Form fields: single column
- Sticky save/cancel buttons at bottom

### 7. Final Polish

- [ ] Loading skeletons for all data-fetching views
- [ ] Error boundaries with friendly error messages
- [ ] Toast notifications for all actions (saved, deleted, renewed, terminated, etc.)
- [ ] Keyboard shortcuts: Cmd+N (new contract), Cmd+E (export), Cmd+F (search/filter)
- [ ] URL-based filters (register filters in query params for shareable links)
- [ ] Confirm dialogs for destructive actions (delete, terminate)
- [ ] Optimistic UI updates (immediate feedback, rollback on error)
- [ ] Empty state illustrations for all views
- [ ] "No results" state for filters that match nothing
- [ ] Date formatting consistent across the app (locale-aware)
- [ ] Currency formatting consistent (locale-aware, 2 decimal places)
- [ ] Tab title updates per page: "Contract Register — Contract Manager"

### 8. waymaker.config.ts Manifest

Create the Host Schema manifest for the Contract Manager app.

**File:** `waymaker.config.ts`

```typescript
import { defineConfig } from '@waymakerone/sdk';

export default defineConfig({
  name: 'Contract Manager',
  slug: 'contract-manager',
  type: 'ex',
  tables: [
    {
      name: 'cm_contracts',
      purpose: 'Core contract records with counterparty, value, dates, and document references',
      operations: ['select', 'insert', 'update', 'delete'],
    },
    {
      name: 'cm_contract_types',
      purpose: 'Contract type definitions with colours, icons, and default durations',
      operations: ['select', 'insert', 'update'],
    },
    {
      name: 'cm_contract_milestones',
      purpose: 'Contract milestones for tracking reviews, renewals, and deadlines',
      operations: ['select', 'insert', 'update', 'delete'],
    },
  ],
  storage: [
    {
      bucket: 'contracts',
      purpose: 'Contract document storage (PDF, DOCX)',
      operations: ['upload', 'download', 'delete'],
    },
  ],
  integrations: [
    {
      service: 'commander-calendar',
      purpose: 'Sync renewal dates and milestones to team calendar',
      operations: ['create', 'update', 'delete'],
    },
    {
      service: 'journeys-send-email',
      purpose: 'Automated renewal, notice period, and milestone reminder emails',
      operations: ['send'],
    },
    {
      service: 'commander-contacts',
      purpose: 'Link counterparties to Commander Contact records',
      operations: ['search', 'get'],
    },
    {
      service: 'commander-automation-operations',
      purpose: 'Auto-expire contracts and auto-create milestones',
      operations: ['create', 'list'],
    },
  ],
});
```

### 9. Deploy-Ready Configuration

- [ ] Verify all environment variables documented
- [ ] Build succeeds with no TypeScript errors: `npm run build`
- [ ] All API calls use authenticated fetch with Bearer token
- [ ] Storage bucket `contracts` created in Supabase
- [ ] RLS policies on all 3 tables (organization_id scoping)
- [ ] Default contract types seed runs on first load
- [ ] Test end-to-end: create contract → upload document → add milestones → view in calendar → receive reminder → export CSV

---

## Acceptance Criteria

- [ ] Dashboard loads with 4 key metric cards showing correct counts and values
- [ ] Expiring in 30/60/90 days sections show correct contracts
- [ ] Contracts by Type donut chart renders with correct type colours
- [ ] Click chart slice filters the register
- [ ] Contracts by Department bar chart renders with correct data
- [ ] Renewal Timeline bar chart shows next 12 months of renewals
- [ ] Recent Activity shows last 10 contract changes
- [ ] Quick Action buttons navigate to correct pages
- [ ] Total Committed Value and Annual Value calculations are correct
- [ ] Value by Type breakdown matches active contract data
- [ ] CSV export downloads with correct format and all columns
- [ ] Export respects current filters (exports only filtered results)
- [ ] CSV opens correctly in Excel and Google Sheets
- [ ] Document bulk download works for multiple selected contracts
- [ ] Mobile: card view for contract register
- [ ] Mobile: full-screen add contract with file upload
- [ ] Mobile: single column contract detail with tap-to-preview document
- [ ] Mobile: dashboard single column with full-width charts
- [ ] Loading skeletons show on all data-fetching views
- [ ] Toast notifications fire on all actions
- [ ] Error boundaries catch and display friendly errors
- [ ] waymaker.config.ts manifest is complete and valid
- [ ] Build succeeds with `npm run build`
- [ ] Deploy succeeds to Waymaker Host
