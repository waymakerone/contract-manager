---
sync:
  type: doc
  layer: Contract Manager
build:
  status: todo
  phase: 2
  priority: P0
  depends_on: ["phase-1-register-and-documents"]
  started_at: null
  completed_at: null
---

# Phase 2: Calendar & Timeline

**Goal:** Calendar view of renewal dates and milestones, Commander Calendar sync, Gantt-style renewal timeline, "Expiring This Month" panel, and automatic status updates for contracts and milestones.

**PRD Reference:** `docs/01-planning/product-requirements/contract-manager-prd.md` — Phase 2

---

## What to Build

### 1. Calendar View (/timeline — Calendar Tab)

A month/quarter calendar view showing renewal dates and milestones across all contracts.

**Layout:**
- Tab bar: "Calendar" | "Timeline" (Gantt)
- Calendar grid: month view by default, quarter toggle

**Calendar entries:**
- Renewal dates: shown as teal markers with contract title
- End dates: shown as red markers for contracts without auto-renew
- Milestones: shown as amber markers with milestone title
- Notice period deadlines: shown as orange markers (calculated: end_date - notice_period_days)

**Each calendar entry shows on hover/click:**
- Contract title + contract number
- Counterparty
- Event type (renewal, end date, milestone, notice deadline)
- Action button (view contract)

**Month navigation:**
- Previous/next month arrows
- "Today" button to jump to current month
- Month/year header

**Quarter view:**
- 3-month grid side by side
- Condensed markers (dots instead of full labels)
- Quarter navigation: Q1, Q2, Q3, Q4

**Colour coding:**
- Teal (`#0F766E`): renewal dates
- Red (`#DC2626`): end dates (non-auto-renew)
- Amber (`#D97706`): milestones
- Orange (`#EA580C`): notice period deadlines
- Grey (`#9CA3AF`): completed milestones

### 2. Commander Calendar Sync

Sync contract events to Commander Calendar so they appear alongside tasks and meetings.

**Events to sync:**
- Contract renewal dates → calendar event: "Contract Renewal: [Title] — [Counterparty]"
- Contract end dates (non-auto-renew) → calendar event: "Contract Expires: [Title] — [Counterparty]"
- Milestones → calendar event: "Contract Milestone: [Milestone Title] — [Contract Title]"
- Notice period deadlines → calendar event: "Notice Deadline: [Title] — [Counterparty]"

**Sync behaviour:**
```typescript
// Create calendar event for a contract renewal
const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-calendar`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    action: 'create',
    data: {
      title: `Contract Renewal: ${contract.title} — ${contract.counterparty}`,
      date: contract.renewal_date,
      type: 'contract-renewal',
      metadata: {
        contract_id: contract.id,
        contract_number: contract.contract_number,
        counterparty: contract.counterparty,
        value: contract.value,
      },
    },
  }),
})
```

**Sync triggers:**
- On contract create: sync renewal date and end date events
- On contract update: update or delete existing calendar events
- On contract delete: remove associated calendar events
- On milestone create/update/delete: sync milestone events
- On milestone complete: remove calendar event

**Calendar event metadata:**
- Store `contract_id` in event metadata for back-linking
- Store event type: `contract-renewal`, `contract-end`, `contract-milestone`, `contract-notice`
- Store `calendar_event_id` on the contract/milestone record for update/delete operations

### 3. Renewal Timeline (Gantt View)

A horizontal timeline showing contract lifespans with key date markers.

**Layout:**
- Horizontal scrollable timeline
- Y-axis: one row per contract (sorted by end_date, soonest first)
- X-axis: months (scrollable, showing 12 months by default)

**Each contract bar:**
- Horizontal bar from start_date to end_date
- Colour: matches contract type colour
- Label: contract title + counterparty (truncated)
- Markers on the bar:
  - Diamond: renewal date
  - Circle: milestones
  - Triangle: notice period deadline
- Hover tooltip: full details (title, counterparty, value, dates, status)
- Click: navigate to contract detail

**Timeline controls:**
- Zoom: 6 months, 12 months, 24 months
- Scroll: horizontal drag or scroll
- Filter: by status, type, department
- "Today" line: vertical red dashed line marking current date

**Evergreen contracts:**
- Bar extends to the right edge with a fade/arrow indicating no end date
- No renewal markers

**Grouping (optional):**
- Toggle grouping by: type, department, status
- Collapsed groups show count and total value

### 4. "Expiring This Month" Panel

A dedicated panel showing contracts that need attention this month.

**Location:** Sidebar panel on the calendar/timeline view, or standalone card on the register page.

**Content:**
- List of contracts where `end_date` is within the current month
- Each entry shows:
  - Contract title + number
  - Counterparty
  - End date with "X days remaining" countdown
  - Auto-renew indicator (yes/no)
  - Value

**Action buttons per contract:**
- **Renew** → Create a new contract pre-filled with the original's details, link via notes/tags, set original's status to "renewed"
- **Terminate** → Set status to "terminated", remove calendar events
- **Extend** → Open edit modal with end_date field focused, update calendar events

**Urgency indicators:**
- Red background: end date within 7 days
- Amber background: end date within 30 days
- Default: end date within the month

### 5. Contract Lifecycle Auto-Update

Automatic status transitions based on dates.

**Status auto-update logic:**
```typescript
// Run on app load and periodically (or via Commander Automations)
async function updateContractStatuses(organizationId: string) {
  const today = new Date().toISOString().split('T')[0];

  // Active contracts that have expired
  const expiredContracts = await queryContracts(organizationId, {
    status: 'active',
    end_date_before: today,
    auto_renew: false,
  });

  for (const contract of expiredContracts) {
    await updateContract(contract.id, { status: 'expired' });
    // Update calendar events
    // Send notification to owner
  }

  // Active contracts with auto_renew that passed end_date
  const autoRenewContracts = await queryContracts(organizationId, {
    status: 'active',
    end_date_before: today,
    auto_renew: true,
  });

  for (const contract of autoRenewContracts) {
    // Auto-renew: extend end_date by type's default_duration_months
    const type = await getContractType(contract.type_id);
    if (type?.default_duration_months) {
      const newEndDate = addMonths(contract.end_date, type.default_duration_months);
      await updateContract(contract.id, {
        end_date: newEndDate,
        renewal_date: newEndDate,
        status: 'active',
      });
      // Update calendar events
    }
  }
}
```

**When to run:**
- On app load (check all active contracts)
- On dashboard visit
- Optionally: via Commander Automations on a schedule

### 6. Milestone Status Auto-Update

Automatic milestone status transitions based on dates.

**Milestone auto-update logic:**
```typescript
async function updateMilestoneStatuses(organizationId: string) {
  const today = new Date().toISOString().split('T')[0];

  // Pending milestones that are overdue
  const overdueMilestones = await queryMilestones(organizationId, {
    status: 'pending',
    due_date_before: today,
  });

  for (const milestone of overdueMilestones) {
    await updateMilestone(milestone.id, { status: 'overdue' });
  }
}
```

**Visual indicators:**
- Pending milestones: amber dot + "Due in X days"
- Overdue milestones: red dot + "Overdue by X days"
- Completed milestones: green dot + "Completed on [date] by [user]"

### 7. Calendar Calculations Utility

Build a utility for calendar-related calculations.

**File:** `src/lib/calendar.ts`

```typescript
interface CalendarEvent {
  id: string;
  date: string;
  title: string;
  type: 'renewal' | 'end-date' | 'milestone' | 'notice-deadline';
  contractId: string;
  contractTitle: string;
  counterparty: string;
  colour: string;
  status: string;
}

function getCalendarEvents(
  contracts: Contract[],
  milestones: ContractMilestone[],
  month: number,
  year: number
): CalendarEvent[];

function getExpiringContracts(
  contracts: Contract[],
  withinDays: number
): Contract[];

function calculateNoticePeriodDeadline(
  endDate: string,
  noticePeriodDays: number
): string;

function isNoticeDeadlinePassed(
  endDate: string,
  noticePeriodDays: number
): boolean;

function getTimelineData(
  contracts: Contract[],
  milestones: ContractMilestone[],
  startMonth: Date,
  monthsToShow: number
): TimelineRow[];
```

---

## Acceptance Criteria

- [ ] Calendar view renders with month grid showing days
- [ ] Renewal dates appear on correct calendar days with teal markers
- [ ] End dates appear on correct calendar days with red markers
- [ ] Milestones appear on correct calendar days with amber markers
- [ ] Notice period deadlines appear on correct calendar days with orange markers
- [ ] Month navigation (previous/next/today) works
- [ ] Quarter view shows 3 months side by side
- [ ] Calendar entry hover shows contract details
- [ ] Calendar entry click navigates to contract detail
- [ ] Commander Calendar sync creates events for renewals and end dates
- [ ] Commander Calendar sync creates events for milestones
- [ ] Calendar events update when contract dates change
- [ ] Calendar events deleted when contract or milestone deleted
- [ ] Gantt timeline shows horizontal bars for each contract
- [ ] Timeline bars span from start_date to end_date with correct width
- [ ] Timeline markers show renewal dates, milestones, and notice deadlines
- [ ] Timeline zoom (6/12/24 months) works
- [ ] "Today" line shows current date on timeline
- [ ] Evergreen contracts show open-ended bars
- [ ] "Expiring This Month" panel lists contracts ending this month
- [ ] Renew action creates new contract pre-filled from original
- [ ] Terminate action sets status to terminated
- [ ] Extend action opens edit with end_date focused
- [ ] Contract status auto-updates from active to expired when end_date passes
- [ ] Auto-renew contracts extend end_date automatically
- [ ] Milestone status auto-updates from pending to overdue when due_date passes
- [ ] Urgency indicators (red/amber) show on expiring contracts
