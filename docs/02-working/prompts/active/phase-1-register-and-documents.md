---
sync:
  type: doc
  layer: Contract Manager
build:
  status: todo
  phase: 1
  priority: P0
  depends_on: []
  started_at: null
  completed_at: null
---

# Phase 1: Contract Register & Documents

**Goal:** App scaffold with auth, data schema (3 tables), contract CRUD, document upload to Supabase Storage, contract type management, milestone CRUD, auto-numbering, and a searchable contract register.

**PRD Reference:** `docs/01-planning/product-requirements/contract-manager-prd.md` — Phase 1

---

## What to Build

### 1. Project Setup

Create a React + Vite + TypeScript + Tailwind app.

**Files:**
- `src/main.tsx` — Clerk provider wrapper
- `src/App.tsx` — Router with routes: `/`, `/contracts/:id`, `/types`, `/timeline`, `/reminders`, `/dashboard`
- `src/lib/api.ts` — Authenticated fetch helper for Commander Tables
- `src/lib/storage.ts` — Supabase Storage helper for document uploads
- `src/lib/types.ts` — TypeScript interfaces for Contract, ContractType, ContractMilestone
- `src/lib/numbering.ts` — Auto-numbering utility for contract numbers
- `src/index.css` — Tailwind base + design tokens

**Auth pattern:**
```typescript
import { useAuth } from '@clerk/clerk-react'

const { getToken } = useAuth()
const token = await getToken()

const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ action: 'query', data: { table: 'cm_contracts', filters: {} } }),
})
```

**TypeScript interfaces:**
```typescript
interface Contract {
  id: string;
  organization_id: string;
  title: string;
  contract_number: string | null;
  type_id: string;
  counterparty: string;
  counterparty_contact_id: string | null;
  status: 'draft' | 'active' | 'expired' | 'terminated' | 'renewed';
  value: number | null;
  currency: string;
  start_date: string;
  end_date: string | null;
  renewal_date: string | null;
  auto_renew: boolean;
  notice_period_days: number | null;
  document_url: string | null;
  document_type: 'pdf' | 'docx' | 'none';
  owner_id: string;
  department: string | null;
  tags: string[] | null;
  notes: string | null;
  created_at: string;
  updated_at: string;
}

interface ContractType {
  id: string;
  organization_id: string;
  name: string;
  icon: string | null;
  colour: string | null;
  default_duration_months: number | null;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}

interface ContractMilestone {
  id: string;
  organization_id: string;
  contract_id: string;
  title: string;
  due_date: string;
  status: 'pending' | 'completed' | 'overdue';
  notes: string | null;
  completed_at: string | null;
  completed_by: string | null;
  reminder_sent: boolean;
  created_at: string;
}
```

### 2. Database Schema

Create these tables in Commander Tables. Use the `commander-table-ddl` edge function or run migrations directly.

**cm_contract_types** — Seed with defaults:

| name | icon | colour | default_duration_months |
|------|------|--------|------------------------|
| NDA | shield | #6B7280 | 24 |
| SaaS Agreement | monitor | #8B5CF6 | 12 |
| Employment | user | #3B82F6 | null |
| Vendor Agreement | truck | #F59E0B | 12 |
| Client Agreement | handshake | #059669 | 12 |
| Lease | building | #EC4899 | 36 |
| Service Agreement | wrench | #06B6D4 | 12 |
| Other | circle | #9CA3AF | null |

**cm_contracts** — See PRD for full schema. Key fields: title, contract_number, type_id, counterparty, status, value, currency, start_date, end_date, renewal_date, auto_renew, notice_period_days, document_url, document_type, owner_id, department.

**cm_contract_milestones** — See PRD. Key fields: contract_id, title, due_date, status, completed_at, completed_by, reminder_sent.

### 3. API Layer

Create a service module for each entity:

```
src/services/
├── contracts.ts      — list, get, create, update, delete
├── types.ts          — list, get, create, update, toggleActive
├── milestones.ts     — list, get, create, update, delete, complete
└── storage.ts        — uploadDocument, getDocumentUrl, deleteDocument
```

All table operations go through `commander-table-operations` edge function with the appropriate table name and action.

Document uploads use Supabase Storage directly:
```typescript
// Upload contract document to storage
const filePath = `${organizationId}/${contractId}/${file.name}`
const { data, error } = await supabase.storage
  .from('contracts')
  .upload(filePath, file, { upsert: true })

// Get public URL
const { data: { publicUrl } } = supabase.storage
  .from('contracts')
  .getPublicUrl(filePath)
```

**Auto-numbering utility:**
```typescript
// src/lib/numbering.ts
async function getNextContractNumber(organizationId: string): Promise<string> {
  // Query max contract_number for the org
  // Parse the numeric part, increment, pad to 4 digits
  // Return "CON-0001", "CON-0002", etc.
  const contracts = await queryContracts(organizationId, { orderBy: 'contract_number', order: 'desc', limit: 1 });
  if (contracts.length === 0) return 'CON-0001';
  const lastNum = parseInt(contracts[0].contract_number?.replace('CON-', '') || '0');
  return `CON-${String(lastNum + 1).padStart(4, '0')}`;
}
```

### 4. Contract Register (/)

The home page. A table of all contracts with summary stats and quick actions.

**Summary bar at top:**
- Total active contracts (count)
- Total active value (sum of value for active contracts)
- Expiring this month (count of contracts with end_date in current month)

**Table columns:**
- Contract # (CON-0001, sortable)
- Title (searchable)
- Type (colored badge matching type colour)
- Counterparty (searchable)
- Status (badge: draft=grey, active=green, expired=red, terminated=grey-strikethrough, renewed=blue)
- Value (formatted currency, right-aligned)
- Start Date (sortable)
- End Date (sortable, "Evergreen" if null)
- Renewal Date (sortable)
- Owner (user name)
- Department

**Filters:**
- Status dropdown (multi-select: draft, active, expired, terminated, renewed)
- Type dropdown (multi-select from cm_contract_types)
- Counterparty search (text)
- Date range picker (start date range)
- Value range (min/max)
- Department dropdown
- Search by title or counterparty

**Actions:**
- "Add Contract" button → opens add contract modal
- Click row → navigate to contract detail
- Bulk select → bulk export (CSV)

**Empty state:** "No contracts yet. Add your first contract to get started." with Add Contract button.

### 5. Add/Edit Contract (Modal)

A modal form for creating or editing a contract.

**Fields:**
- Title (text input, required)
- Type (dropdown from cm_contract_types where is_active = true) — on select, auto-fill duration
- Counterparty (text input, required)
- Status (dropdown: draft, active — default draft for new)
- Value (number input, optional, 2 decimal places)
- Currency (dropdown, defaults to USD)
- Start Date (date picker, required)
- End Date (date picker, optional — auto-calculated from type's default_duration_months if set)
- Renewal Date (date picker, optional — defaults to end_date)
- Auto Renew (toggle, default off)
- Notice Period (number input, optional — days before end date)
- Owner (user selector from org members)
- Department (text input, optional)
- Tags (tag input, optional)
- Notes (textarea, optional)
- Document upload (drag-and-drop zone: accepts application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document)

**On type select:**
- If type has `default_duration_months` and start_date is set: auto-calculate end_date
- Show: "Default duration: 12 months" hint text

**On submit:**
1. Generate contract_number (CON-XXXX) if new contract
2. If document provided: upload to Supabase Storage first, get URL
3. Create/update contract record in Commander Tables with document_url
4. Status defaults to `draft` for new contracts
5. Close modal, refresh contract register

**Document upload zone:**
- Drag-and-drop area with "Drop contract document here or click to upload" text
- Accept: application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document
- Max file size: 50MB
- Show file name and type icon after upload
- "Remove" button to clear the upload

### 6. Contract Detail (/contracts/:id)

Full view of a single contract.

**Layout:**
- Left column (60%): contract fields (editable if status is draft)
- Right column (40%): document preview

**Header:**
- Contract number (CON-0001) + Title
- Status badge (colored)
- Type badge (colored)
- Action buttons: Edit, Renew, Terminate, Extend, Delete (draft only)

**Contract info section:**
- All fields displayed in a structured card layout
- Counterparty name (linked to contact if counterparty_contact_id set)
- Value + currency formatted
- Key dates: start, end, renewal, notice deadline (calculated: end_date - notice_period_days)
- Auto-renew indicator
- Owner name
- Department
- Tags displayed as pills

**Document preview (right column):**
- If PDF: embedded PDF viewer (iframe or react-pdf)
- If DOCX: download button with file icon and name
- If none: "No document attached" with upload button
- "Download" button
- "Replace" button (upload new document)

**Milestones section (below):**
- Timeline/list of milestones for this contract
- Each milestone shows: title, due date, status (pending=yellow, completed=green, overdue=red)
- "Add Milestone" button → inline form or modal
- "Complete" button on pending milestones → sets completed_at and completed_by
- "Edit" and "Delete" buttons on each milestone

**Status bar:**
- Visual lifecycle indicator: draft → active → expired/terminated/renewed
- Current status highlighted
- Key dates marked on the timeline

### 7. Contract Type Management (/types)

List of all contract types with management actions.

**Table/grid view:**
- Type name with icon and colour dot
- Default duration (if set, otherwise "No default")
- Contract count (number of contracts using this type)
- Active/inactive toggle

**Actions:**
- "Add Type" → modal: name, icon (picker), colour (picker), default duration in months (optional)
- Edit type → same modal in edit mode
- Toggle active/inactive (inactive types hidden from contract form dropdown)

### 8. Design Tokens

Use a professional, trust-oriented design:
- Font: `Geist` (load from Google Fonts)
- Primary: `#0F766E` (teal) — primary buttons, links, active states
- Background: `#FFFFFF` (white) — page background
- Surface: `#F9FAFB` (off-white) — card backgrounds, table rows
- Text: `#111827` (near-black)
- Text Muted: `#6B7280` (grey)
- Success/Active: `#059669` (green) — active contract status
- Warning/Expiring: `#D97706` (amber) — expiring soon indicators
- Danger/Expired: `#DC2626` (red) — expired status, overdue milestones
- Blue: `#3B82F6` — renewed status, informational
- Border: `#E5E7EB`
- Cards: white, `border-radius: 12px`, subtle shadow
- Spacing: 8px base grid

---

## Acceptance Criteria

- [ ] App loads with Clerk auth (sign-in required)
- [ ] Contract register shows all contracts with correct columns
- [ ] Search by title and counterparty works
- [ ] Filters work: status, type, counterparty, date range, value range, department
- [ ] Add contract modal creates a contract with all fields
- [ ] Contract number auto-generated (CON-0001, CON-0002, etc.)
- [ ] Type selection auto-fills default duration and calculates end date
- [ ] Document upload works (PDF and DOCX) with file name display
- [ ] Document stored in Supabase Storage, URL saved on contract
- [ ] Contract detail page shows all fields + document preview
- [ ] PDF documents render inline in the detail view
- [ ] Edit contract works (draft status only)
- [ ] Delete contract works (draft status only) with confirmation
- [ ] Contract type list shows all types with contract count
- [ ] Add/edit contract type works with icon and colour pickers
- [ ] Toggle contract type active/inactive works
- [ ] Default contract types seeded on first load
- [ ] Milestones display on contract detail page
- [ ] Add milestone with title, due date, and notes works
- [ ] Complete milestone sets completed_at and completed_by
- [ ] Edit and delete milestone works
- [ ] Summary stats show at top of contract register
- [ ] Empty states for all views
- [ ] Responsive: table scrolls horizontally on mobile
