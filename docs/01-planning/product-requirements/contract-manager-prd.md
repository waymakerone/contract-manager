# Contract Manager — Product Requirements

**Status:** Approved
**Author:** Waymaker
**Date:** 2026-03-04
**Last Updated:** 2026-03-04

## Problem Statement

Every organisation manages contracts — vendor agreements, NDAs, employment contracts, SaaS subscriptions, leases. The workflow is universal: negotiate terms, sign the document, track the dates, and remember to renew or terminate before the deadline. Yet most SMBs manage this in a shared folder of PDFs and a spreadsheet of dates, or pay $20-50/user/month for Ironclad, Juro, or ContractSafe and fight with rigid workflows that don't match how they actually work.

The pain is rarely in the signing — it's in what happens after. Contracts expire without anyone noticing. Renewal deadlines pass, locking the organisation into unfavourable terms. Notice periods are missed, triggering automatic renewals on contracts they wanted to renegotiate. Milestones like annual reviews or price adjustments go untracked. The document itself is buried in someone's email or a shared drive folder no one can find.

This blueprint builds a contract manager as a WaymakerOS app. Contracts live in Commander Tables (the same database that holds tasks, contacts, and goals). Documents are stored in Supabase Storage. Renewal dates and milestones sync to Commander Calendar. Automated reminders go out via Commander Journeys. Counterparties link to Commander Contacts. The app is the view and logic layer — Commander is the data layer. The result is a contract manager that's part of the operating system, not a standalone tool with its own silo.

## Goals

1. **One place for all contracts** — Every contract recorded in a searchable, filterable register with type, counterparty, value, status, and key dates
2. **Document storage with preview** — Upload contract PDFs and DOCX files, view them inline without leaving the app
3. **Renewal and milestone tracking** — Never miss a renewal date, notice period, or milestone deadline
4. **Automated reminders** — Email notifications at configurable intervals before key dates
5. **Contract lifecycle management** — Track contracts through draft, active, expired, terminated, and renewed statuses

## Non-Goals

- This is NOT a contract authoring tool — no document editor, clause library, or template merging
- This is NOT an e-signature platform — no DocuSign/HelloSign-style signing workflow
- This is NOT a legal review tool — no clause comparison, risk scoring, or AI contract analysis
- This is NOT a procurement system — no purchase orders, vendor scoring, or sourcing workflows

## Data Model

### Core Objects

**Contract** — A single contract between the organisation and a counterparty.
- Core fields: title, contract_number (auto-generated CON-0001), type, counterparty, status
- Value: value (total contract value), currency
- Dates: start_date, end_date (null for evergreen), renewal_date, auto_renew, notice_period_days
- Document: document_url (Supabase Storage), document_type (pdf, docx, none)
- Ownership: owner_id (person responsible), department
- Metadata: tags, notes
- Status: draft, active, expired, terminated, renewed

**Contract Type** — A way to classify contracts. Seeded with defaults, user-editable.
- Fields: name, icon, colour, default_duration_months, is_active
- Seeds: NDA, SaaS Agreement, Employment, Vendor Agreement, Client Agreement, Lease, Service Agreement, Other

**Contract Milestone** — A key event within a contract's lifecycle.
- Fields: title, due_date, status, notes, completed_at, completed_by, reminder_sent
- Status: pending, completed, overdue
- Examples: Annual Review, Price Adjustment, Renewal Decision, Compliance Check

### Tables Schema (Commander Tables)

```
cm_contracts
├── id (uuid, PK)
├── organization_id (text, FK → Clerk org)
├── title (text, required)
├── contract_number (text, nullable — auto-generated: CON-0001)
├── type_id (uuid, FK → cm_contract_types)
├── counterparty (text, required — the other party)
├── counterparty_contact_id (text, nullable — Clerk contact ID or Commander contact)
├── status (text: draft, active, expired, terminated, renewed)
├── value (numeric, nullable — total contract value)
├── currency (text, default 'USD')
├── start_date (date, required)
├── end_date (date, nullable — null for evergreen)
├── renewal_date (date, nullable)
├── auto_renew (boolean, default false)
├── notice_period_days (integer, nullable — days before end to give notice)
├── document_url (text, nullable — Supabase Storage URL)
├── document_type (text: pdf, docx, none)
├── owner_id (text, Clerk user ID — person responsible)
├── department (text, nullable)
├── tags (text[], nullable)
├── notes (text, nullable)
├── created_at (timestamptz)
└── updated_at (timestamptz)

cm_contract_types
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── icon (text, nullable)
├── colour (text, nullable — hex code)
├── default_duration_months (integer, nullable)
├── is_active (boolean, default true)
├── created_at (timestamptz)
└── updated_at (timestamptz)

cm_contract_milestones
├── id (uuid, PK)
├── organization_id (text)
├── contract_id (uuid, FK → cm_contracts)
├── title (text, required)
├── due_date (date, required)
├── status (text: pending, completed, overdue)
├── notes (text, nullable)
├── completed_at (timestamptz, nullable)
├── completed_by (text, nullable)
├── reminder_sent (boolean, default false)
└── created_at (timestamptz)
```

### Default Contract Type Seeds

| Name | Icon | Colour | Default Duration |
|------|------|--------|-----------------|
| NDA | shield | #6B7280 | 24 months |
| SaaS Agreement | monitor | #8B5CF6 | 12 months |
| Employment | user | #3B82F6 | null |
| Vendor Agreement | truck | #F59E0B | 12 months |
| Client Agreement | handshake | #059669 | 12 months |
| Lease | building | #EC4899 | 36 months |
| Service Agreement | wrench | #06B6D4 | 12 months |
| Other | circle | #9CA3AF | null |

### Commander Integration Map

| Action | Commander Tool | How |
|--------|---------------|-----|
| All contract data | Tables | `commander-table-operations` → CRUD on cm_contracts, cm_contract_types, cm_contract_milestones |
| Document storage | Storage | Upload to Supabase Storage `contracts` bucket, store URL on contract record |
| Renewal calendar | Calendar | `commander-calendar` → create/update/delete events for renewal dates and milestones |
| Reminder emails | Email/Journeys | `journeys-send-email` → automated reminders at 90, 60, 30, 7 days before key dates |
| Counterparty contacts | Contacts | `commander-contacts` → link contracts to contact records for counterparties |
| Status automations | Automations | `commander-automation-operations` → auto-expire contracts, auto-create default milestones |

## Proposed Solution

### Overview

An internal (EX) app deployed to Waymaker Host that provides complete contract lifecycle management: contract register, document storage, type management, milestone tracking, renewal timeline, automated reminders, and an executive dashboard. All data lives in Commander Tables. Documents in Supabase Storage. Calendar events via Commander Calendar. Reminders via Commander Journeys.

The app is the view layer and the contract logic. Commander is the data layer and the storage layer.

### Key Views

1. **Contract Register** (`/`) — Home page. Table of all contracts, filterable by status, type, counterparty, date range, value range, and department. Summary stats at top (total active, total value, expiring soon). Click row to open contract detail.

2. **Contract Detail** (`/contracts/:id`) — Full contract view. Left panel: all fields (editable if draft). Right panel: document preview (PDF viewer or download link). Below: milestones timeline with add/edit/complete actions. Status bar showing lifecycle position. Action buttons: renew, terminate, extend.

3. **Add/Edit Contract** (modal) — Form with all contract fields: title, type, counterparty, value, currency, dates, auto-renew toggle, notice period, document upload, owner, department, tags, notes. Type selection auto-fills default duration.

4. **Contract Types** (`/types`) — Manage contract categories: add, edit, deactivate. Each type shows contract count, default duration, and colour.

5. **Renewal Timeline** (`/timeline`) — Two views: (a) Calendar view showing renewal dates and milestones by month/quarter. (b) Gantt-style timeline showing contract bars with start/end dates and renewal markers. "Expiring This Month" panel with action buttons.

6. **Reminders** (`/reminders`) — Configure automated email reminders: intervals (90, 60, 30, 7 days), enable/disable per contract, preview email templates. Reminder log showing sent notifications.

7. **Dashboard** (`/dashboard`) — Executive overview: total active contracts, total active value, expiring in 30/60/90 days, contracts by type (donut chart), contracts by department (bar chart), renewal timeline (next 12 months), recently added/modified.

### User Flow

1. Legal/ops team member opens Contract Manager from Host
2. Clicks "Add Contract" → enters title, selects type "Vendor Agreement"
3. Type auto-fills 12-month duration; user sets start date, end date calculated
4. Uploads signed contract PDF → stored in Supabase Storage
5. Sets counterparty "Acme Corp", links to Commander Contact
6. Adds milestones: "Annual Review" at 6 months, "Renewal Decision" at 11 months
7. Sets auto-renew off, notice period 30 days
8. Contract saved as active, renewal date and milestones synced to Commander Calendar
9. At 90 days before end: automated email reminder to contract owner
10. At 30 days (notice period): urgent reminder with action buttons
11. Owner reviews, decides to renew — creates new contract linked to the original
12. Dashboard shows updated value and renewal timeline

## Scope

### Phase 1 (MVP) — Register & Documents

- [ ] App scaffold: React + Vite + Tailwind + Clerk auth
- [ ] Tables schema: cm_contracts, cm_contract_types, cm_contract_milestones
- [ ] API layer: CRUD operations for all tables via authenticated edge function
- [ ] Seed default contract types
- [ ] Contract register with search, filter by status/type/counterparty/date range/value range/department
- [ ] Add/edit contract form with all fields
- [ ] Auto-numbering: CON-0001, CON-0002, etc.
- [ ] Document upload to Supabase Storage (PDF, DOCX)
- [ ] Contract detail page with document preview (PDF viewer)
- [ ] Contract type management (list, add, edit, deactivate)
- [ ] Milestone CRUD on contract detail (add, edit, complete)

### Phase 2 — Calendar & Timeline

- [ ] Calendar view (month/quarter) showing renewal dates and milestones
- [ ] Sync events to Commander Calendar (create/update/delete)
- [ ] Renewal timeline view (Gantt-style: contract bars with start/end, renewals marked)
- [ ] "Expiring This Month" panel with action buttons (renew, terminate, extend)
- [ ] Contract lifecycle status auto-update (active → expired when end_date passes)
- [ ] Milestone status auto-update (pending → overdue when due_date passes)

### Phase 3 — Reminders & Contacts

- [ ] Automated email reminders via journeys-send-email
- [ ] Configurable intervals: 90, 60, 30, 7 days before renewal/end date
- [ ] Notice period alerts (remind at notice deadline if notice_period_days set)
- [ ] Milestone reminder emails (7 days before, on due date)
- [ ] Counterparty contact linking via commander-contacts
- [ ] Automation rules via commander-automation-operations
- [ ] Email templates for renewal reminders with contract summary

### Phase 4 — Dashboard & Polish

- [ ] Executive dashboard with metric cards, charts, and timeline
- [ ] Value tracking: total committed value, annual value
- [ ] Contracts by type (donut chart) and by department (bar chart)
- [ ] Expiring in 30/60/90 days counters
- [ ] Renewal timeline (next 12 months)
- [ ] CSV export (all contracts, filtered contracts)
- [ ] Mobile-responsive layouts (card view)
- [ ] Document bulk download
- [ ] waymaker.config.ts manifest for Host Schema
- [ ] Deploy-ready configuration

### Out of Scope

- Contract authoring or document editing (use Word/Google Docs)
- E-signature workflows (use DocuSign, HelloSign)
- AI contract analysis or clause extraction
- Procurement, purchase orders, or vendor scoring
- Multi-party contracts with complex approval chains
- Version control for contract amendments (store new version as separate document)
- Integration with external CLM platforms

## Success Criteria

| Metric | Target |
|--------|--------|
| Contract entry | < 60 seconds to add a contract with document upload |
| Renewal visibility | All contracts with end dates visible in calendar and timeline views |
| Reminder delivery | Automated emails sent at configured intervals before key dates |
| Search and filter | Find any contract by counterparty, type, or status in < 2 seconds |
| Build time (with AI) | < 4 hours for all 4 phases |

## Dependencies

- WaymakerOS organization with Commander access
- Commander Tables for data storage
- Supabase Storage for contract documents
- (Optional) Commander Calendar for renewal date sync
- (Optional) Commander Journeys for automated email reminders
- (Optional) Commander Contacts for counterparty linking
- (Optional) Commander Automations for status auto-update rules

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Large document uploads | Low | Low | Supabase Storage handles large files natively; set 50MB limit |
| Missed reminders | Medium | High | Multiple reminder intervals; fallback: dashboard "expiring soon" panel |
| Calendar sync failures | Low | Medium | Calendar sync is enhancement; core tracking works without it |
| Complex renewal logic | Medium | Medium | Keep renewal simple: create new contract, link to original |
| Date timezone issues | Medium | Low | Store all dates as UTC; display in user's timezone |

## Open Questions

- [x] Internal (EX) or external (CX)? **EX — team members only**
- [x] E-signature integration? **Out of scope — upload signed documents**
- [x] Contract versioning? **Out of scope — upload new document for amendments**
- [ ] Multi-currency value tracking? **Display only — no conversion**
- [ ] Approval workflow for contract creation? **Out of scope for V1 — add later if needed**
