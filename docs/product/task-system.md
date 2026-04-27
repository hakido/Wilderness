# Task system

The operator workflow spine. Every confirmed booking generates a structured list of work to be done; the system tracks who's doing it, when it's due, and what's at risk of slipping.

The principle: **operators don't fail at any single task — they fail at handoffs and dropped balls.** The task system's job is to make the implicit explicit. What lives today in someone's head, in a WhatsApp thread, or on a Post-it gets a row in `tasks` with an assignee, a due date, and a reminder schedule.

## Core model

**Tasks are generated upfront when a booking is confirmed.** The full list is visible from day one. Tasks become "active" as their due dates approach.

**Reminders are dynamic.** They're computed against the current state of each task — fired, escalated, cancelled, or rescheduled based on what's actually happening, not on a static schedule set at task creation.

This split is deliberate: the structure (what work exists) is stable and predictable; the behavior (when to nag, who to escalate to) responds to reality.

## Task generation

When a booking is confirmed (payment webhook received, booking record created), the system generates tasks based on:

- The trip's components (which parks, which lodges, which vendors, how many travelers)
- The trip's dates (due dates are computed relative to trip start, trip end, or booking date)
- The operator's task templates (operator-configurable; ships with sensible defaults per trip type)
- The traveler details (international travelers have visa-related tasks; domestic don't)

The generation runs once at booking confirmation and is **transactional** with the booking creation — either the booking is confirmed and tasks exist, or neither happens.

### Example task list

For a 6-day Bandhavgarh + Kanha safari for 4 travelers (2 international + 2 domestic), the system might generate:

**Pre-trip — operator-facing**
- Apply for Bandhavgarh permits (Tala zone, 4 morning + 4 afternoon safaris) — due 120 days before trip start
- Apply for Kanha permits — due 120 days before trip start
- Confirm Lodge Tigergarh booking (3 nights) — due 14 days before trip
- Confirm Lodge Kanha Jungle Camp (3 nights) — due 14 days before trip
- Confirm transfer driver — due 7 days before trip
- Confirm naturalist for Bandhavgarh — due 7 days before trip
- Confirm naturalist for Kanha — due 7 days before trip
- Send packing list and briefing to traveler — due 14 days before trip
- Send final vouchers and itinerary — due 3 days before trip
- Send day-1 welcome message — due 1 day before trip
- Confirm GST invoice details — due 7 days before trip

**Pre-trip — customer-facing**
- Collect passport copies for permits — due 30 days before trip
- Collect emergency contacts and medical disclosures — due 30 days before trip
- Collect arrival flight details — due 14 days before trip
- Collect insurance proof — due 21 days before trip

**During trip — operator-facing**
- Confirm arrival at airport — due day 1
- Daily check-in with lodge — recurring across trip days
- Day 1 morning safari verification — due day 2 morning
- Day 2 morning safari verification — due day 3 morning
- (similar daily verifications)
- Confirm departure transfer — due last day

**Post-trip — operator-facing**
- Send review request — due 1 day after trip end
- Send GST invoice — due 3 days after trip end
- Reconcile vendor invoices — due 7 days after trip end
- Pay vendor balances — due 14 days after trip end

That's roughly 25 tasks for one booking. The operator's actual day-to-day work is managing this list across all active bookings.

### Manual task creation

The auto-generated list is the baseline. Operators can manually add ad-hoc tasks at any time — "call Sharma to discuss accessibility needs," "follow up with Lodge X about the previous complaint."

Manual tasks have the same structure as auto-generated tasks (assignee, due date, type, status), with `source = manual` to distinguish.

## Task structure

Every task has:

- **id** — uuid
- **operator_id** — tenant scope (RLS)
- **booking_id** — the booking this task belongs to (or null for non-booking tasks)
- **type** — categorical, from a per-operator vocabulary (e.g., `permit_application`, `vendor_confirmation`, `customer_form_collection`, `manifest_generation`, `payment_collection`)
- **title** — human-readable summary
- **description** — optional longer description, populated from template
- **assignee_user_id** — operator team member assigned, or null if unassigned
- **due_at** — timestamp with timezone
- **status** — `pending`, `in_progress`, `blocked`, `completed`, `cancelled`
- **priority** — `low`, `normal`, `high`, `critical` (from template + booking value modifiers)
- **source** — `auto` (template-generated) or `manual`
- **template_id** — references the template that generated this task, if auto-sourced
- **dependencies** — array of task IDs that must complete before this one becomes active
- **completed_at**, **completed_by_user_id** — set on completion
- **completion_notes** — free text the user can add
- **created_at**, **updated_at** — timestamps

## Task templates

Operator-configurable. Each template defines:

- **type** — the task type
- **applies_to** — which trip configurations trigger this template (e.g., "all trips that include Bandhavgarh," "all international travelers," "any trip over $5,000")
- **title_template** — string template for the task title (e.g., "Apply for {park_name} permits — {dates}")
- **description_template** — longer version
- **due_relative_to** — anchor: `booking_confirmed`, `trip_start`, `trip_end`, `payment_due_date`
- **due_offset** — duration before/after the anchor (e.g., -120 days for permit applications)
- **default_priority**
- **default_assignment_rule** — see assignment section below
- **dependencies** — references to other templates whose generated tasks block this one
- **reminder_schedule** — see reminders section

Templates ship with **sensible defaults per trip type** (safari, heritage, trek, etc.) so a new operator doesn't start from scratch. They can edit, add, remove, and create custom templates.

Hakido maintains a library of starter templates per vertical that operators import and customize.

## Assignment

Tasks are assigned to operator team members per the operator's configured rules.

### Assignment rules

The operator configures rules at the template level or globally:

- **Bulk-assign all tasks for a booking to one person** — small operator default ("everything → Priya")
- **Assign by task type** — "all permit tasks → Ramesh," "all customer comms → Sneha"
- **Assign by booking attribute** — "all tasks for international travelers → Sales+CS team," "all tasks for trips over $10K → Owner"
- **Pool / unassigned** — task goes into a queue anyone with permission can claim
- **Round-robin** within a role — distribute new tasks evenly among the sales team

In v1, assignment rules are **simple** — a small set of conditions per template. Sophisticated rule engines come later if needed.

### Reassignment

Any task can be manually reassigned at any time. The system tracks reassignment history (audit log).

When a user is removed from the operator's team or their role changes, their open tasks are flagged for reassignment. The operator owner sees a list of orphaned tasks requiring action.

### Out-of-office

Operator team members can set themselves as out-of-office with a date range and a delegate. New auto-assignments during that window go to the delegate. Manual assignments still go to the original person but trigger a notification.

## Reminders

Reminders are computed dynamically against current task state. They are **not** stored as scheduled jobs at task creation; they are recomputed by a periodic job that examines task state.

This means: completing a task cancels its reminders. Snoozing a task pushes them out. Trip-date changes recompute due dates and reset the reminder schedule for affected tasks.

### Reminder schedules

Per-task-type, configurable, with sensible defaults.

**Operator reminders** (for tasks needing operator action):
- 2 days before due — notification to assignee
- Day of due — notification to assignee
- Day after due — notification to assignee, surfaced in "overdue" view
- Daily until resolved — notifications to assignee + escalation to owner after 3 days overdue

**Customer reminders** (for tasks needing customer action — passport, flight details, payment):
- 7 days before due — WhatsApp + email to customer
- 3 days before due — WhatsApp + email to customer
- 1 day before due — WhatsApp + email to customer
- Day of due — operator follow-up call task generated automatically
- Day after due — escalation to operator team

**Owner escalations**:
- Tasks overdue 3+ days appear on the owner's "needs attention" view automatically
- Tasks marked `critical` priority appear immediately when overdue
- Optional notification per owner preference

### Channels

- **In-dashboard** — notification badge, surfaced in the assignee's home view
- **Email** — daily digest of reminders and overdue tasks (operator-configurable frequency)
- **WhatsApp** — for customer-facing reminders, optionally for operator (operator preference)
- **Push notification** — when operator mobile app exists (v2+)

### What changes a reminder

- Task completed → all future reminders cancelled
- Task cancelled → all future reminders cancelled
- Task due_at changed → reminders recomputed against new due date, notification to assignee about the change
- Task reassigned → reminders re-aimed at new assignee, notification to both old and new
- Task priority elevated to critical → immediate notification, more aggressive reminder schedule
- Operator on out-of-office → reminders go to delegate

### Suppression

Operators can snooze a task ("not urgent, check back next week") which pushes due_at and recomputes reminders. They can also suppress reminders for a task without changing due_at if they need to stop notifications without stretching the deadline (the task stays overdue but stops nagging).

## Customer-side reminders

When a task needs customer input — pre-trip forms, document uploads, balance payments — the system sends reminders to the **customer**, not just the operator.

These are operator-branded WhatsApp + email messages. The system schedules them based on the task's due_at; if the customer takes action, the task moves toward completion and reminders cancel; if not, the next reminder fires per schedule.

After the customer-side reminders exhaust without action, an operator follow-up task is auto-generated ("Call customer to collect passport details — due tomorrow"). This is the T-2-days reminder call we promised travelers.

## Priorities and SLAs

Tasks have a priority level (`low`, `normal`, `high`, `critical`). Defaults come from templates; manual elevation is allowed.

**SLA-driven priority elevation:**
- A task within 24 hours of due_at: priority elevates from `normal` to `high`
- A task overdue: elevates to `high` (or stays at `critical` if already there)
- A task overdue 3+ days: elevates to `critical`

The "today" view sorts by priority then due date, so the most urgent work surfaces first.

**Operator-defined SLA rules:**
- High-value bookings (configurable threshold) get all tasks elevated by one level
- VIP customers (operator-flagged) get the same treatment
- Specific task types ("permit application") can have custom SLAs

## Views

The dashboard exposes tasks through several views.

### "My today" — the operator's home page

When an operator team member logs in, the home page is **a worklist of their tasks for today** — sorted by priority, due time, and any dependencies. Not a chart dashboard. The screen they live in.

Above the worklist: a small status row showing pipeline summary, today's arrivals/departures, anything urgent across the team. But the worklist is the focus.

### Per-booking task view

Inside a booking detail page, the full task list for that booking — completed, pending, overdue — with assignees and due dates. This is the "what's left to do for this customer" view.

### Team view

For users with permission, a view of all tasks across the team — filterable by assignee, status, type, due date. Useful for owners who want global visibility, and for ops managers covering during someone's absence.

### Overdue and at-risk views

A dedicated view of tasks that are overdue or at risk (due within 24 hours and not started). The operator's "fire drill" surface — what needs attention right now.

### Permit calendar

A specialized view for permit-related tasks, organized by park and date. Shows permit-release dates, applications in progress, allocations confirmed. (Permit work has unique cadence — see operations.)

## Dependencies

Some tasks depend on others. "Send final vouchers" can't be completed until "Confirm all vendor reservations" is done. "Send GST invoice" can't fire until "Reconcile vendor invoices" completes.

Dependencies are declared at the template level and instantiated when tasks generate.

A task with unmet dependencies is `pending` but not `active`. It doesn't fire reminders, doesn't appear in worklists, and shows on the booking detail page as "waiting for: [dependency]." When the dependency completes, the dependent task becomes active and its due date is computed (if it was anchored to dependency-completion rather than trip dates).

Dependencies are advisory in v1 — the system won't refuse to let a user complete a downstream task if upstream is incomplete; it just warns. Hard enforcement is v2 if needed.

## Tasks and the agent

The agent and the task system intersect at booking confirmation.

**Agent confirms a booking** (via the payment webhook flow, not directly):
- Booking record created
- Task generation runs, creating the full upfront task list
- Operator's task list updates in real time via Supabase Realtime — the team sees new work appear

**Agent does not create or modify tasks directly.** Tasks are an operator-side concept. The agent's role is conversation; once it produces a confirmed booking, task system takes over.

**Agent does read task status** for inquiries: "Did you confirm my lodge?" — the agent calls `getBookingTasks(bookingId)` to see the status of the relevant task and respond truthfully.

**Agent can create handoff tasks**: when it escalates a conversation to a human, the system creates a handoff task assigned per operator rules.

## Tasks and customer communication

Many tasks generate customer-facing messages — booking confirmation, payment reminder, briefing, daily welcome. These messages are tied to tasks, sent through templates, with the template's content compiled with booking-specific variables.

When a templated message sends successfully, the sending task is marked complete (or marked "sent, awaiting customer action" for tasks that depend on customer response).

When a customer-facing template fails to send (WhatsApp delivery failure, email bounce), the task is flagged `blocked` and an operator-facing task is generated ("Customer didn't receive briefing — investigate and follow up").

## Audit and history

Every state change on every task is logged: created, assigned, reassigned, status changed, due_at changed, reminder fired, completed, cancelled. The audit log is queryable per task and per user.

The audit log is the operator's "what happened" trail. When something goes wrong on a booking, the audit log answers: who was assigned what, when, what reminders fired, what the assignee did.

## Reporting

V1 reports the task system supports:

- **Completion rate** — tasks completed on time, late, or not at all (per assignee, per type, per period)
- **Escalation rate** — tasks that hit owner-escalation thresholds
- **Volume** — tasks generated, completed, in flight (per period)
- **Bottlenecks** — task types or assignees with disproportionate overdue rates
- **Customer-side completion** — % of pre-trip forms completed on time

These feed the owner's understanding of how their operation is running.

## What v1 doesn't include

- **Sophisticated workflow orchestration** — tasks are atomic, with simple dependencies. No complex branching, parallel paths, or BPMN-style workflows.
- **Task subtasks / nested hierarchies** — flat list per booking.
- **Cross-booking task batching** — "send all confirmation emails for tomorrow's bookings at once" is a v2 nicety.
- **Time tracking** — operators can't log hours per task.
- **Public APIs for task integration** — third-party tools can't read/write tasks via API in v1.
- **Workflow automation rules** — "if task X completes, automatically create task Y in another system." Out of v1.

These are all reasonable v2 features once the basic system is proven and operators have specific asks.
