# Contract Manager

Track contracts, monitor renewal dates, store documents, and automate reminders — powered by Commander Tables, Documents, Calendar, and Automations.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Auth | Clerk (via WaymakerOS) |
| Data | Commander Tables (Supabase PostgreSQL) |
| Storage | Supabase Storage (contracts bucket) |
| Calendar | Commander Calendar (renewal dates, milestones) |
| Email | Commander Journeys (automated reminders) |
| Contacts | Commander Contacts (counterparty linking) |
| Automations | Commander Automations (status updates, milestone creation) |
| Charts | recharts |
| Hosting | Waymaker Host (EX app) |

## Documentation

| Folder | Contents |
|--------|----------|
| `docs/01-planning/product-requirements/` | PRD — data model, views, integration map |
| `docs/02-working/prompts/active/` | Build prompts — 4 phases with YAML front matter |
| `docs/02-working/sessions/` | Session briefs as you build |
| `docs/03-knowledge/` | Patterns discovered during the build |

## Build Phases

| Phase | Prompt | Status |
|-------|--------|--------|
| 1 — Register & Documents | `docs/02-working/prompts/active/phase-1-register-and-documents.md` | todo |
| 2 — Calendar & Timeline | `docs/02-working/prompts/active/phase-2-calendar-and-timeline.md` | todo |
| 3 — Reminders & Contacts | `docs/02-working/prompts/active/phase-3-reminders-and-contacts.md` | todo |
| 4 — Dashboard & Polish | `docs/02-working/prompts/active/phase-4-dashboard-and-polish.md` | todo |

## Data Model (Quick Reference)

```
cm_contract_types ──< cm_contracts ──< cm_contract_milestones
                         │
                    document files
                    (Supabase Storage)
```

- **Contract Type** defines categories of contracts (NDA, SaaS Agreement, Employment, etc.) with colour and default duration
- **Contract** is the core record — one row per contract with counterparty, value, dates, status, and document attachment
- **Contract Milestone** tracks key events within a contract (reviews, price adjustments, renewal decisions) with due dates and completion status
- Contract documents stored in Supabase Storage `contracts` bucket, linked via `document_url`

## Commander Integration

| Action | Commander Tool | API |
|--------|---------------|-----|
| All contract data | Tables | `commander-table-operations` → CRUD on cm_contracts, cm_contract_types, cm_contract_milestones |
| Document storage | Storage | Supabase Storage `contracts` bucket, URL saved on contract record |
| Renewal calendar | Calendar | `commander-calendar` → create/update/delete events for renewals and milestones |
| Reminder emails | Email/Journeys | `journeys-send-email` → automated renewal and milestone reminders |
| Counterparty contacts | Contacts | `commander-contacts` → link contracts to contact records |
| Status automations | Automations | `commander-automation-operations` → auto-expire, auto-create milestones |

## Critical Rules

- All data lives in Commander Tables — the app is the view + logic layer
- Auth via Clerk: `useAuth().getToken()` → Bearer token on all API calls
- API calls POST to `${SUPABASE_URL}/functions/v1/{function-name}` with `{ action, data }` body
- Document uploads go to Supabase Storage, URL saved on the contract record
- Contract numbers auto-generated: CON-0001, CON-0002, etc.
- Renewal dates and milestones sync to Commander Calendar
- Automated reminders via Journeys at configurable intervals (90, 60, 30, 7 days)
- Design tokens: Teal `#0F766E` primary, White `#FFFFFF` background, Green `#059669` active, Amber `#D97706` expiring soon, Red `#DC2626` expired
- Use Geist font family
- All tables scoped by `organization_id` with RLS policies

## How to Build

1. Read the PRD: `docs/01-planning/product-requirements/contract-manager-prd.md`
2. Work through each phase prompt in order
3. Update the YAML `status` field as you go: `todo` → `in-progress` → `review` → `done`
4. Write session briefs in `docs/02-working/sessions/completed/` between sessions

## Quick Reference

| What | How |
|------|-----|
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Tables API | `commander-table-operations` → `query`, `insert`, `update`, `delete` |
| Storage API | Supabase Storage → `contracts` bucket |
| Calendar API | `commander-calendar` → create events for renewals and milestones |
| Email API | `journeys-send-email` → send reminder notifications |
| Contacts API | `commander-contacts` → link counterparties |
| Automations API | `commander-automation-operations` → auto-status, auto-milestones |
