# Contract Manager

A complete contract management app built on WaymakerOS. Track contracts, store documents, monitor renewal dates, manage milestones, and automate reminders — all integrated with Commander Tables, Documents, Calendar, Contacts, and Automations.

## What You Get

- **Contract Register** — All contracts in a searchable, filterable table with status badges, counterparty, value, and key dates
- **Document Storage** — Upload contract PDFs and DOCX files to Supabase Storage with inline PDF preview
- **Contract Types** — Manage contract categories (NDA, SaaS Agreement, Employment, Vendor) with colours and default durations
- **Milestones** — Track key events within each contract: annual reviews, price adjustments, renewal decisions
- **Renewal Timeline** — Calendar and Gantt-style timeline view of upcoming renewals synced to Commander Calendar
- **Automated Reminders** — Email notifications at configurable intervals before renewal dates and milestone deadlines
- **Dashboard** — Total contracts, active value, expiring soon, breakdown by type and department, renewal timeline

## Commander Tools Used

| Tool | How the Contract Manager Uses It |
|------|------|
| **Tables** | All contract data (contracts, types, milestones) |
| **Storage** | Contract document storage (PDFs and DOCX) |
| **Calendar** | Renewal dates and milestones on team calendar |
| **Email/Journeys** | Automated renewal and milestone reminder emails |
| **Contacts** | Counterparty contact linking |
| **Automations** | Auto-expire contracts, auto-create milestones for new contracts |

## How to Build

1. Clone this blueprint into your project
2. Open `CLAUDE.md` — it's the router file for your AI coding tool
3. Read the PRD in `docs/01-planning/product-requirements/`
4. Work through the 4 phase prompts in `docs/02-working/prompts/active/`
5. Point Claude Code, Cursor, or Codex at each phase and build

**Estimated build time:** 3-4 hours across all 4 phases.

## Build Phases

| Phase | What You Get |
|-------|-------------|
| 1 — Register & Documents | App scaffold, schema (3 tables), contract CRUD, document upload, contract types, milestones, auto-numbering |
| 2 — Calendar & Timeline | Calendar view of renewals and milestones, Commander Calendar sync, Gantt-style timeline, auto-status updates |
| 3 — Reminders & Contacts | Automated email reminders (90/60/30/7 day), counterparty contact linking, automation rules |
| 4 — Dashboard & Polish | Executive dashboard, value charts, CSV export, mobile responsive, waymaker.config.ts manifest |

## Prerequisites

- WaymakerOS organization with Commander access
- Commander Tables enabled
- Supabase Storage bucket for contract documents
- (Optional) Commander Calendar for renewal sync
- (Optional) Commander Journeys for automated reminders
- (Optional) Commander Contacts for counterparty linking
