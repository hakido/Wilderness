# Operators

The operator-side of TravelHub — the dashboard, workflows, and configuration surfaces that tour management companies use to run their business. This is the bulk of the software.

The system is replacing **WhatsApp + spreadsheets + memory**, not Salesforce. Most operators today run their business on WhatsApp threads scattered across team members' phones, with critical knowledge living in someone's head. The product's primary job is to make that implicit knowledge **legible and shared**.

Optimize for state visibility and dropped-ball prevention over sophisticated workflow automation. A 3-person operator drowning in WhatsApp wins big from "here are all your active customers in one place"; they win nothing from "here is your sophisticated lead-routing engine."

## Three operational pillars

The operator's work divides into three pillars. The dashboard is organized around them.

### Sales (quote-to-closure)

The pipeline from a customer first reaching out, through itinerary planning and quote negotiation, to a confirmed booking. Most operator pain today: leads disappear, quotes go unanswered, follow-ups fall through.

**Capabilities:**
- Pipeline visibility: Kanban view of every active lead/conversation/quote with stage, value, owner, last-touched
- Conversation inbox: every customer interaction (web chat, WhatsApp inbound, email) in one place, threaded by customer
- Quote management: view, edit, version, send, track acceptance
- Follow-up reminders: automatic nudges on stale quotes
- Lost-deal capture: simple reason capture when a quote dies (price, dates, lost to competitor, postponed, etc.)
- Pipeline metrics: pipeline value by stage, conversion rates, by source, by sales rep

**Out of v1 scope:**
- Sophisticated lead routing rules (manual claim or simple round-robin only)
- Discount approval workflows beyond per-user authority
- Multi-stage quote approvals
- Forecasting and territory management
- Sales coaching tooling

### Operations (the supply side)

Vendor management, permits, manifests, day-of-trip coordination. The operator's actual delivery work.

**Capabilities:**
- Vendor directory: by type (lodge, transport, naturalist, driver, guide), with linked canonical records (see `vendors.md`)
- Vendor reservation tracking per booking: status (requested → confirmed → completed)
- Vendor confirmation worklist: "trips departing in next 14 days with unconfirmed vendors"
- Permit calendar per park: release dates, allocated, available, applied, confirmed
- Permit assignment to bookings
- Manifest generation: daily run-sheets for the ground team
- Daily ops view: today's arrivals, departures, safaris, transfers across all active trips
- Per-trip live status during the trip (booking confirmed → traveler arrived → trip in progress → completed)
- Issue capture during trip: what happened, who's affected, status, resolution
- Conflict detection: room double-bookings, vehicle conflicts, naturalist overlaps (basic v1; sophistication grows in v2)

**Out of v1 scope:**
- Sophisticated inventory ownership models (we support hard/soft/request allocation in v1; complex contractual variations come later)
- Automated re-issue and replan workflows for vendor cancellations (v1 is human-driven with system support)
- Vendor portal or vendor-side calendar
- Vendor payment processing (track payables, generate settlement reports; payment happens outside the system)

### Customer operations (relationship and trip success)

Pre-trip and during-trip customer relationships, issue handling, post-trip continuity. The relationship layer that distinguishes premium operators.

**Capabilities:**
- Per-customer unified timeline: every web chat, WhatsApp message, email, quote, booking, payment, issue, internal note in one chronological view
- Pre-trip task tracking per booking: passport collected, insurance verified, briefing sent, payment received, vendor confirmations done
- Customer profile: preferences, past trips, saved travelers, dietary, accessibility, internal notes
- VIP and repeat-customer flagging: surface to the team so a 5-trip repeat customer isn't treated like a stranger
- Active-trip monitoring: green/yellow/red status across all in-flight trips
- Issue management: capture mid-trip complaints, assign, resolve, follow up
- Post-trip review collection: ratings on operator, lodges, naturalists, drivers, vehicles
- Day-by-day customer touchpoint scheduling: templated daily messages during trips, personalized

**Out of v1 scope:**
- Loyalty programs and points (operator can configure rebook discounts but not full loyalty)
- Sophisticated NPS and survey tooling beyond basic post-trip review
- AI-driven sentiment analysis on customer communications
- Cross-operator customer recognition

## v1 priority ranking by pillar

Given the design partner's reality (3-person team, WhatsApp-driven, no current system), the v1 build effort prioritizes:

1. **Customer operations** — highest priority. The unified per-customer timeline is the dashboard's hero feature. Pre-trip task tracking and reminders are the most visible operator wins.
2. **Sales** — high priority. Pipeline visibility and follow-up nudges are direct dropped-ball prevention.
3. **Operations** — medium priority. Entities and basic workflows in v1; sophistication grows in v2.

This ordering is reflected in build sequencing and informs scope decisions when something has to slip.

## The task model

The operator's workflow spine. Detailed in `task-system.md`.

When a booking is confirmed, the system generates an **upfront list of tasks** based on the trip's shape (parks, vendors, dates, travelers). Tasks have type, due date, assignee, status, dependencies. Reminders against those tasks are **dynamic** — fired based on current task state as time passes.

The operator's home page is a **task list**, not a chart dashboard: "what do I need to do today, this week, that's overdue."

This is the architecture that lets the system adapt from 3-person operators (everything assigned to one person) to 20-person operators (specialists) without changing the underlying model.

## RBAC

Permissions → roles → users. Flexible enough to scale from "one person does everything" to "specialists with narrow responsibilities."

### Permissions

Atomic things a user can do. The system has roughly 30-40 of these. Examples:

- `view_pipeline`, `edit_pipeline_deals`, `assign_pipeline_deals`
- `view_bookings`, `edit_bookings`, `cancel_bookings`, `modify_confirmed_bookings`
- `approve_discount_5pct`, `approve_discount_10pct`, `approve_refund`
- `view_vendors`, `edit_vendors`, `edit_vendor_contracts`, `edit_vendor_rates`
- `view_permits`, `apply_permits`, `assign_permits_to_bookings`
- `view_financials`, `view_pipeline_value`, `view_revenue_reports`
- `send_communications`, `edit_communication_templates`
- `view_team`, `manage_team`, `manage_roles`
- `configure_agent`, `configure_catalog`, `configure_policies`, `configure_branding`
- `platform_admin` (Hakido-internal only, never granted to operators)

### Roles

Bundles of permissions. The operator owner can use system-shipped templates or create custom roles.

**System-shipped role templates** (operators can use as-is, customize, or ignore):

- **Owner** — all permissions
- **Sales + Customer Success** — pipeline, bookings, communications, view financials, no configuration
- **Operations** — vendors, permits, manifests, bookings, no financial visibility, no configuration
- **Finance** — view financials, revenue reports, refunds, no operational editing
- **Read-only** — view everything, edit nothing

Operators can also create custom roles ("Senior Sales," "Trainee," "Permits Manager"). Permission editing is owner-only.

### Users

Each user has one or more roles. Effective permissions are the union.

### v1 simplifications

- **No per-record ownership rules** in v1 ("rep can only see their own deals"). This is where RBAC gets genuinely complex. Start with role-based access; add ownership scoping in v2 if operators ask.
- **No time-based or conditional permissions** ("can approve discounts only on weekdays"). Out of v1.
- **No delegation chains** ("if owner is on vacation, deputy gets owner's permissions"). Out of v1.
- **Owner cannot lose all permissions**. The owner role on the operator account always retains the ability to manage team and roles, even if a misconfigured custom role is assigned.

### Audit log

All permission-relevant actions are audit-logged: role creation, role edits, user role assignments, permission grants, sensitive actions (refund approval, vendor contract edits, configuration changes). Audit log is owner-visible only.

## The unified inbox / timeline

The hero feature. If your design partner's biggest pain is "I don't know what's happening with each customer," the answer is one chronological view of everything.

### Per-customer timeline

For every customer, a single chronological view containing:

- Web chat messages (with the agent or with humans)
- WhatsApp messages (inbound and outbound, both directions ingested via webhook)
- Emails (sent from the system; future: ingested replies)
- Quotes (created, sent, viewed, modified, accepted, rejected)
- Bookings (created, modified, cancelled, completed)
- Payments (initiated, completed, refunded, failed)
- Tasks related to this customer's bookings (created, due, completed, overdue)
- Issues (raised, assigned, resolved)
- Internal notes from the team
- Reviews (post-trip, with ratings)

Filterable by type if the timeline gets noisy. Searchable by keyword. Anyone with permission to view this customer can see it.

### Inbox view

Across all customers, the operator sees an inbox of active conversations — anyone who needs a response, sorted by last activity, filterable by stage, owner, source.

The inbox supports:
- Reply directly (web chat goes to web; WhatsApp reply queued through the operator's WhatsApp Business API number)
- Assign or claim ownership of the conversation
- Mark as resolved / awaiting customer / handed back to agent
- Flag for follow-up at a specific time
- Add internal notes

### Mobile responsiveness

The inbox is the most-mobile-used surface. Operators check it constantly, often on phones. Critical mobile workflows: read messages, send templated replies, assign tasks, mark things done. Heavy authoring (building quotes, configuring trip catalog) is desktop-acceptable.

## Operator behaviors

Ten categories of how operators actually work, with architectural implications. These behaviors shape the dashboard's design.

### 1. One-person-owns-it vs. baton-pass

**Behavior:** Small operators (3 people) tend to have one person own a booking from quote to completion. The relationship is personal. Larger operators split ownership by stage — sales handles quote-to-confirmation, ops takes over for permit/vendor work, customer success handles pre-trip and during-trip.

**Implications:**
- The system supports both: a booking has a primary owner, and tasks have individual assignees. At small operators these collapse to one person; at larger operators they spread across specialists.
- Handoffs between team members are explicit events: notification, inbox item, timestamp. Without this, the larger operator's pain (things falling between teams) doesn't get solved.
- The customer never sees the handoff — they see one consistent operator persona.

### 2. WhatsApp is the work

**Behavior:** Operators do their work *inside* WhatsApp threads. They quote there, confirm there, send updates there. A system that lives outside WhatsApp loses unless it integrates.

**Implications:**
- The dashboard cannot be the only interface. Operators need to act from inside their phone.
- v1 includes both **outbound WhatsApp** (system to customer/vendor via operator's WhatsApp Business number) and **inbound webhook ingestion** (replies flow into the dashboard's timeline so any team member can see them).
- When operators reply via WhatsApp on their phone, the same webhook ingests their reply into the system. End state: full conversation captured regardless of where the operator typed.
- Operator mobile app for staff is v2+. v1 relies on responsive web dashboard + WhatsApp-on-phone.

### 3. Concurrent edits and shared visibility

**Behavior:** Multiple team members will be looking at the same booking, customer, or quote at the same time. Two people might edit the same itinerary. One on a call while another sees an inbound WhatsApp from the same customer.

**Implications:**
- Realtime presence: show who else is viewing/editing a booking. (Supabase Realtime earns its keep here.)
- Optimistic concurrency control on writes: last-write-wins is fine for most things, but quotes and itineraries (where two people might edit simultaneously) use version numbers and surface conflicts.
- Activity feed per booking: "Priya updated the lodge to Tigergarh." "Ramesh marked permit confirmed." So the team can catch up without asking each other.

### 4. Mobile-first operators

**Behavior:** The team is rarely all at desks. Owner is at trade shows, sales is at meetings, ops is at the bank. They check the system from phones constantly.

**Implications:**
- Dashboard is responsive and phone-usable. Critical workflows must work well on mobile: inbox, today's tasks, send a message, view a booking, mark something done.
- Quick actions: confirm a vendor, send a templated message, mark a booking paid, see today's arrivals — all one-tap on mobile.
- Heavy authoring (quote-building, trip catalog config, rate cards) can be desktop-optimized.

### 5. Constant interruption and partial work

**Behavior:** Operator workflows are deeply interrupt-driven. A salesperson starts a quote, customer calls about another booking, vendor WhatsApps about a permit, owner asks a question, phone rings, 47 minutes later they return to the original quote.

**Implications:**
- Drafts everywhere. Quotes, messages, itineraries auto-save aggressively. Losing 20 minutes of work to a tab close is unacceptable.
- "Resume where I left off" affordances. Last-viewed booking, drafts in progress, my open tasks. The dashboard home answers "what was I doing?"
- Don't build long forms. Big multi-step wizards die in interrupt-heavy workflows. Prefer many small actions over a few big ones.

### 6. Trust through transparency, not gatekeeping

**Behavior:** Small operators are deeply uncomfortable when they can't see what's happening. Owners want to see every quote that's gone out, who's unhappy, what the team is doing — without micromanaging.

**Implications:**
- Dashboards default to "everything I have permission to see," not "just my stuff." Owners want the global view.
- Notifications and activity feeds for the owner: "Priya sent a quote for ₹2.4L," "Ramesh marked Sharma booking complete," "Ops flagged Bandhavgarh permit issue." Configurable so it doesn't drown them.
- No surprises in reporting. The owner reading the monthly summary should not learn anything for the first time.

### 7. Memory and context loss

**Behavior:** The 3-person operator's biggest existential risk: someone leaves, gets sick, or goes on vacation, and their knowledge goes with them.

**Implications:**
- Internal notes on every booking and customer. Free-text notes the team writes for each other. "Customer prefers ground-floor rooms." "Husband is the decision maker." "Don't book Lodge X, bad experience last year."
- Required fields on certain actions. Marking a deal lost requires a reason. Cancelling a booking requires a reason. Forces context capture humans naturally skip.
- Customer profile is the source of truth — preferences, dietary, past trips, special considerations. Built up over time, available to anyone helping that customer.

### 8. Numbers anxiety

**Behavior:** Operators worry about money constantly. Cash flow, payments due, pending receivables, vendor payables, refunds, GST.

**Implications:**
- Financial visibility in v1, not v2. Even without full accounting, the dashboard needs: pipeline value, confirmed bookings revenue (this month, this quarter), outstanding payments due from customers, outstanding vendor payables.
- Currency conversion in reports. Multi-currency makes "how are we doing" harder. Reports consolidate to base currency (typically INR) with option to see by transaction currency.
- Export to spreadsheet. Owners and accountants want data out for their own analysis or for Tally/Zoho Books. Make export easy.

### 9. Seasonality and burst loads

**Behavior:** Indian wilderness tourism is sharply seasonal:
- Oct-March: peak season, parks open, bookings flood in
- April-June: shoulder, fewer bookings, planning time
- July-September: monsoon, parks closed, near-zero bookings, time for permits and contracts

Permit-release dates create burst loads — when Bandhavgarh opens 120-day-out bookings, half the country's safari operators are racing for the same dates.

**Implications:**
- System has to be fast under peak-season load. Efficient queries, good caching.
- Onboarding new operators is best in off-season (when they have time to use new features). Build the onboarding flow knowing operators come on in batches in summer monsoon.
- Burst-load workflows for permit releases — a "permit application day" mode where the system is optimized for entering 50 permit applications in 2 hours.

### 10. Reluctance to abandon their tools

**Behavior:** Small operators have built their workflow on Tally for accounts, Google Sheets for trip catalogs, WhatsApp for everything else. They are not going to abandon Tally for your finance module.

**Implications:**
- Import flows that respect existing data: upload your vendor list as CSV, import your trip catalog from a spreadsheet.
- Export flows that integrate with their existing tools: Tally export, GST report export, vendor payable export.
- Don't fight what the system isn't. v1 is a CRM + ops tool that exports to their accounting system, not a replacement for it. Be clear about scope.

## Configuration surfaces

Everything the operator configures, and where it lives in the dashboard.

### Trip catalog

**Fixed departures** — packaged trips with dates, prices, inclusions, photos, marketing copy. Each fixed departure has many specific dated instances. Editable in the trip catalog area.

**Components catalog** — building blocks for custom itineraries: parks, lodges, transfers, naturalists, activities. Each has structured data the agent reasons over: rates, capacity rules, child policies, compatibility with other components.

**Pricing rules** — per-component rates, seasonal adjustments, party-size adjustments, operator margin. The pricing engine the agent's `priceItinerary` tool uses.

**Constraints** — operator-configured rules: minimum/maximum trip duration, acceptable party sizes, pacing rules ("no more than 2 transfer days in a row"), component compatibility ("Lodge X requires minimum 2 nights").

### Park and vendor information

**Park content** — operator-authored info about parks: zones, closures (weekly and seasonal), child-age policies, fitness requirements, photos, sample sighting data, what to expect.

**Vendor profiles** — for lodges, naturalists, drivers, etc. The operator creates `operator_vendor` rows linking to canonical vendors and adds their contracts, rates, internal notes. Customer-facing data (photos, room categories, naturalist bios) is shown on the customer site; commercial data stays internal.

### Policies and statements

**Cancellation policy** — operator-authored, surfaced to customers before payment, quoted by the agent in refund explanations.

**Expectation statements** — operator-authored content the agent quotes verbatim on relevant topics: tiger sightings, weather, child suitability, fitness requirements, etc. Categorized by trigger (e.g., "show this when customer asks about sightings").

**Force majeure language** — what the operator says when they have to cancel a trip for reasons outside the customer's control.

**Terms and waivers** — required disclosures and acknowledgments at booking time.

### Agent configuration

Per `agent.md`. Operator configures:
- Agent name and persona
- Voice samples and tone
- Required discovery questions
- Custom escalation triggers
- Forbidden topics
- Out-of-scope handling

The operator does **not** write raw prompts. They fill in structured fields; the system compiles the prompt.

### Communications

**WhatsApp templates** — outbound messages approved by Meta: confirmations, payment reminders, pre-trip nudges, daily briefings, post-trip thank-yous. Each template is a structured message the operator (or the system on their behalf) sends on triggers.

**Email templates** — same structure, different channel.

**Trigger rules** — when each template fires (booking confirmed → confirmation sent, payment due tomorrow → reminder sent, etc.).

### Integration settings

- **WhatsApp Business API** — operator's verified business number, Meta credentials, approved template list
- **Stripe** — operator's Stripe Connect account, supported currencies
- **Email sender** — operator's verified domain for outbound email via SES
- **Currencies** — base currency, supported display currencies, FX rate source
- **Tax** — GST registration details for domestic transactions

### Branding and white-label

- **Domain** — `{operator}.travelhub.hakido.co` is the default; custom domains via CNAME are v2
- **Logo, colors, fonts** — applied to the customer-facing site
- **Hero copy and marketing content** — operator's homepage and trip-catalog descriptions
- **Contact info, social links** — surfaced on the customer site
- **Legal pages** — operator's terms, privacy policy

### Team and access

- Add/remove team members via email invite
- Assign role(s) from preset templates or operator-created custom roles
- Audit log of significant changes

## Reporting and analytics

V1 reports — operators can view in dashboard, export as CSV/Excel, schedule for email delivery.

**Pipeline reports**
- Pipeline value by stage
- Conversion rate from lead to booking
- Quote-to-booking time
- Lost reasons distribution
- Source attribution (custom-trip path vs fixed-departure path)

**Bookings reports**
- Bookings by month, by trip type, by source
- Revenue by month, by currency (in base + display)
- Average booking value
- Repeat customer rate

**Operations reports**
- Per-vendor performance (from feedback ratings)
- Permit utilization (allocated vs sold)
- Vendor cost share by booking
- Outstanding vendor payables

**Financial reports**
- Customer payments received (this month/quarter/year)
- Outstanding customer receivables
- Refund volume and reasons
- GST report (domestic transactions, formatted for Tally import)

**Operator can build:**
- Custom views (saved filters)
- Scheduled email digests (daily/weekly/monthly)

**Out of v1:**
- Sophisticated BI / dashboard builder
- Predictive analytics (forecast, churn, conversion likelihood)
- Cohort analysis
- A/B testing infrastructure

## What v1 is not

To be explicit, the operator dashboard in v1 does **not** include:

- Full accounting (use Tally / Zoho Books; export bridges)
- Marketing automation (email campaigns, drip sequences, lead nurturing)
- Detailed employee performance management
- Complex commission structures
- A/B testing of agent prompts or marketing content
- A dedicated mobile app for operator staff
- Calendar integration with Google Calendar / Outlook (manual export only)
- Offline mode
- White-glove onboarding tools (each new operator is hand-onboarded by Hakido in v1)

These are reasonable v2 features as the product matures and operator needs become clearer.
