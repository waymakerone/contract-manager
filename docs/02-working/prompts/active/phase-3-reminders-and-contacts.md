---
sync:
  type: doc
  layer: Contract Manager
build:
  status: todo
  phase: 3
  priority: P0
  depends_on: ["phase-2-calendar-and-timeline"]
  started_at: null
  completed_at: null
---

# Phase 3: Reminders & Contacts

**Goal:** Automated email reminders for contract renewals, notice periods, and milestones via Commander Journeys. Counterparty contact linking via Commander Contacts. Automation rules via Commander Automations for auto-status and auto-milestone creation.

**PRD Reference:** `docs/01-planning/product-requirements/contract-manager-prd.md` — Phase 3

---

## What to Build

### 1. Reminder Configuration

Build a reminder settings system that lets users configure when and how they receive contract alerts.

**Reminder settings page** (`/reminders`):

**Global defaults section:**
- Renewal reminder intervals: checkboxes for 90, 60, 30, 14, 7 days before end_date/renewal_date
- Notice period reminder: checkbox "Remind at notice deadline" (if notice_period_days set)
- Milestone reminders: checkboxes for 7 days before, on due date
- Default enabled intervals: 90, 30, 7 days + notice deadline + 7 days before milestone

**Per-contract override:**
- On contract detail page, "Reminders" tab/section
- Toggle: "Use custom reminder schedule" (overrides global defaults)
- Same interval checkboxes as global
- Additional: "Add custom reminder" → date picker for one-off reminder

**Data model for reminder config:**
```typescript
interface ReminderConfig {
  id: string;
  organization_id: string;
  contract_id: string | null;      // null = global default
  reminder_type: 'renewal' | 'notice' | 'milestone';
  days_before: number;             // days before the event
  enabled: boolean;
  created_at: string;
}

// Store as org-level settings or per-contract overrides
// Could also be a JSON column on cm_contracts for simplicity
```

**Alternative simpler approach (recommended for V1):**
- Store reminder intervals as a JSON array on the contract: `reminder_days: [90, 30, 7]`
- Global defaults in a single org settings record
- No separate table needed

### 2. Automated Email Reminders

Send reminder emails using Commander Journeys (`journeys-send-email`).

**Reminder check logic:**
```typescript
// Run on app load, dashboard visit, or scheduled automation
async function checkAndSendReminders(organizationId: string) {
  const today = new Date();
  const contracts = await getActiveContracts(organizationId);

  for (const contract of contracts) {
    const reminderDays = contract.reminder_days || [90, 60, 30, 7];

    for (const daysBefore of reminderDays) {
      const reminderDate = subtractDays(contract.end_date || contract.renewal_date, daysBefore);

      if (isToday(reminderDate) && !alreadySent(contract.id, daysBefore)) {
        await sendRenewalReminder(contract, daysBefore);
        await markReminderSent(contract.id, daysBefore);
      }
    }

    // Notice period reminder
    if (contract.notice_period_days && contract.end_date) {
      const noticeDeadline = subtractDays(contract.end_date, contract.notice_period_days);
      if (isToday(noticeDeadline) && !alreadySent(contract.id, 'notice')) {
        await sendNoticeReminder(contract);
        await markReminderSent(contract.id, 'notice');
      }
    }
  }

  // Milestone reminders
  const pendingMilestones = await getPendingMilestones(organizationId);
  for (const milestone of pendingMilestones) {
    // 7 days before
    const sevenDaysBefore = subtractDays(milestone.due_date, 7);
    if (isToday(sevenDaysBefore) && !milestone.reminder_sent) {
      await sendMilestoneReminder(milestone);
    }
    // On due date
    if (isToday(milestone.due_date) && !milestone.reminder_sent) {
      await sendMilestoneReminder(milestone);
      await updateMilestone(milestone.id, { reminder_sent: true });
    }
  }
}
```

**Sending emails via Journeys:**
```typescript
async function sendRenewalReminder(contract: Contract, daysBefore: number) {
  const token = await getToken();

  await fetch(`${SUPABASE_URL}/functions/v1/journeys-send-email`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'send',
      data: {
        to: contract.owner_id, // Resolve to email via Clerk
        subject: `Contract Renewal: ${contract.title} — ${daysBefore} days remaining`,
        template: 'contract-renewal-reminder',
        variables: {
          contract_title: contract.title,
          contract_number: contract.contract_number,
          counterparty: contract.counterparty,
          end_date: formatDate(contract.end_date),
          days_remaining: daysBefore,
          value: formatCurrency(contract.value, contract.currency),
          auto_renew: contract.auto_renew ? 'Yes' : 'No',
          action_url: `${APP_URL}/contracts/${contract.id}`,
        },
      },
    }),
  });
}
```

### 3. Email Templates

Design email templates for each reminder type.

**Renewal Reminder Email:**
```
Subject: Contract Renewal: [Title] — [X] days remaining

Hi [Owner Name],

Your contract "[Title]" with [Counterparty] is expiring in [X] days.

Contract Details:
- Contract #: [CON-0001]
- Value: [$X,XXX]
- End Date: [March 30, 2026]
- Auto-Renew: [Yes/No]

[If notice_period_days set and within notice period:]
⚠️ Notice Period: You have [Y] days to give notice before auto-renewal.

[View Contract] [Renew] [Terminate]

— Waymaker Contract Manager
```

**Notice Period Reminder Email:**
```
Subject: ⚠️ Notice Deadline Today: [Title] — [Counterparty]

Hi [Owner Name],

Today is the notice deadline for your contract "[Title]" with [Counterparty].

If you do not give notice by today, the contract will [auto-renew / expire] on [End Date].

Contract Details:
- Contract #: [CON-0001]
- Value: [$X,XXX]
- End Date: [March 30, 2026]
- Notice Period: [30] days

[View Contract] [Terminate] [Renew]

— Waymaker Contract Manager
```

**Milestone Reminder Email:**
```
Subject: Contract Milestone Due: [Milestone Title] — [Contract Title]

Hi [Owner Name],

A milestone for contract "[Contract Title]" is [due in 7 days / due today / overdue].

Milestone: [Milestone Title]
Due Date: [March 15, 2026]
Contract: [Contract Title] with [Counterparty]
Notes: [Milestone notes if any]

[View Contract] [Mark Complete]

— Waymaker Contract Manager
```

### 4. Reminder Log

Track all sent reminders for audit and troubleshooting.

**Reminder log section** (on `/reminders` page):
- Table: date sent, contract title, reminder type (renewal/notice/milestone), recipient, days before event
- Filter by: contract, type, date range
- Sorted by most recent first

**Data tracking:**
- Store sent reminders in a simple log (could be a new table `cm_reminder_log` or JSON on the contract)
- Fields: contract_id, reminder_type, days_before, sent_at, recipient_id

### 5. Counterparty Contact Linking

Connect contracts to Commander Contact records for counterparties.

**Contact linking on contract form:**
- "Counterparty Contact" field below the counterparty text field
- Search/select from Commander Contacts
- Shows: contact name, company, email
- "Create New Contact" link → opens Commander Contacts to create

**Implementation:**
```typescript
// Search contacts
const contacts = await fetch(`${SUPABASE_URL}/functions/v1/commander-contacts`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    action: 'search',
    data: { query: searchTerm },
  }),
});

// Link contact to contract
await updateContract(contractId, {
  counterparty_contact_id: selectedContact.id,
});
```

**Contact display on contract detail:**
- If linked: show contact card with name, company, email, phone
- Click contact card → link to Commander Contacts detail
- "Unlink" button to remove the association
- If not linked: "Link to Contact" search button

**Contracts list on contact (future enhancement):**
- When viewing a contact in Commander, show associated contracts
- Out of scope for this blueprint — would require Commander UI changes

### 6. Automation Rules

Set up automated actions using Commander Automations.

**Auto-status automation:**
```typescript
// Register automation rule
await fetch(`${SUPABASE_URL}/functions/v1/commander-automation-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    action: 'create',
    data: {
      name: 'Contract Auto-Expire',
      trigger: 'schedule',
      schedule: 'daily',
      conditions: [
        { field: 'status', operator: 'equals', value: 'active' },
        { field: 'end_date', operator: 'before', value: 'today' },
        { field: 'auto_renew', operator: 'equals', value: false },
      ],
      actions: [
        { type: 'update', field: 'status', value: 'expired' },
      ],
      table: 'cm_contracts',
    },
  }),
});
```

**Auto-create milestones for new contracts:**
- When a new contract is created with an end_date:
  - Auto-create "Renewal Decision" milestone at `end_date - 60 days`
  - Auto-create "Annual Review" milestone at `start_date + 12 months` (if contract is longer than 12 months)
  - Auto-create "Notice Deadline" milestone at `end_date - notice_period_days` (if notice_period_days set)

**Implementation:**
```typescript
async function autoCreateMilestones(contract: Contract) {
  const milestones: Partial<ContractMilestone>[] = [];

  if (contract.end_date) {
    // Renewal decision 60 days before end
    milestones.push({
      contract_id: contract.id,
      organization_id: contract.organization_id,
      title: 'Renewal Decision',
      due_date: subtractDays(contract.end_date, 60),
      status: 'pending',
      notes: `Decide whether to renew, terminate, or renegotiate the contract with ${contract.counterparty}.`,
    });

    // Notice deadline if notice period set
    if (contract.notice_period_days) {
      milestones.push({
        contract_id: contract.id,
        organization_id: contract.organization_id,
        title: 'Notice Deadline',
        due_date: subtractDays(contract.end_date, contract.notice_period_days),
        status: 'pending',
        notes: `Last day to give notice to ${contract.counterparty}. After this date, the contract will ${contract.auto_renew ? 'auto-renew' : 'expire'}.`,
      });
    }
  }

  // Annual review if contract is longer than 12 months
  if (contract.start_date && contract.end_date) {
    const durationMonths = monthsBetween(contract.start_date, contract.end_date);
    if (durationMonths > 12) {
      milestones.push({
        contract_id: contract.id,
        organization_id: contract.organization_id,
        title: 'Annual Review',
        due_date: addMonths(contract.start_date, 12),
        status: 'pending',
        notes: `Annual review of contract terms and performance with ${contract.counterparty}.`,
      });
    }
  }

  for (const milestone of milestones) {
    await createMilestone(milestone);
  }
}
```

### 7. Reminder Preferences UI

Settings page for managing reminder preferences.

**Global settings card:**
- "Default Reminder Schedule" heading
- Checkbox list: 90 days, 60 days, 30 days, 14 days, 7 days, 1 day
- Toggle: "Send notice period reminders"
- Toggle: "Send milestone reminders (7 days before + on due date)"
- "Save" button

**Per-contract reminder override (on contract detail):**
- Collapsible "Reminders" section
- Toggle: "Custom schedule" (off = use global defaults)
- If on: same checkbox list as global, pre-filled with global defaults
- Additional: "Add one-off reminder" → date picker
- Preview: "Next reminders" list showing upcoming reminder dates for this contract

---

## Acceptance Criteria

- [ ] Reminder settings page loads with global defaults
- [ ] Default intervals (90, 30, 7 days) are pre-selected
- [ ] Global reminder settings save and persist
- [ ] Per-contract custom reminder schedule works
- [ ] Renewal reminder emails sent at configured intervals
- [ ] Renewal email contains correct contract details and action links
- [ ] Notice period reminder email sent on the notice deadline date
- [ ] Notice email contains urgency messaging and correct deadline
- [ ] Milestone reminder email sent 7 days before and on due date
- [ ] Milestone email contains milestone and contract details
- [ ] Reminder log shows all sent reminders with date and recipient
- [ ] Duplicate reminders are not sent (already-sent tracking works)
- [ ] Contact search finds Commander Contacts by name or company
- [ ] Contact can be linked to a contract from the contract form
- [ ] Linked contact displays on contract detail with name, email, phone
- [ ] Contact can be unlinked from a contract
- [ ] "Create New Contact" link opens Commander Contacts
- [ ] Auto-create milestones triggers when new contract is created
- [ ] "Renewal Decision" milestone auto-created 60 days before end
- [ ] "Notice Deadline" milestone auto-created at notice period
- [ ] "Annual Review" milestone auto-created for contracts > 12 months
- [ ] Automation rules register with Commander Automations
- [ ] "Next reminders" preview shows correct upcoming dates
