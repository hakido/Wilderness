# TravelHub

Multi-tenant SaaS for tour operators, built by Hakido. Each operator (tenant) gets a white-labeled customer-facing site at `{operator}.travelhub.hakido.co` and an operator dashboard. Initial design partner is a wilderness/safari operator in India; the system is designed to serve any tour operator.

This file is the spine. It's loaded on every session. Long-form context lives in `docs/product/`, `docs/architecture/`, and `docs/adr/` — read those when working in the relevant area.

## What this product is

Three customer-facing flows on each operator's site:

1. **Browse fixed departures** — packaged trips with set dates and prices. Pure e-commerce: list, filter, view, book, pay. No agent involved.
2. **Plan a custom trip** — conversational itinerary planning with an AI agent. The agent constructs bespoke itineraries against the operator's components catalog, iterates with the customer, and produces a quote leading to payment.
3. **FAQ + WhatsApp** — static FAQ for common questions on fixed-departure pages; anything else routes to humans via WhatsApp.

All three converge on the same booking record, the same task system, the same pre-trip and post-trip flows. The differentiation is upfront in how the booking is made; everything after is unified.

The operator dashboard is the bulk of the software. It runs the operator's sales pipeline, ops workflow (vendors, permits, manifests), customer relationship management, and configuration of the agent and the catalog. The dashboard's hero feature is a unified per-customer timeline that consolidates web chat, WhatsApp, email, quotes, bookings, payments, and internal notes.

The system is replacing WhatsApp-and-spreadsheets, not Salesforce. Optimize for state visibility and dropped-ball prevention over sophisticated workflow automation.

## Stack (locked)

**Frontend**
- `apps/web-customer` — Next.js 15 (App Router) + TypeScript + Tailwind, white-labeled per tenant via subdomain
- `apps/web-operator` — Next.js 15 + TypeScript, operator dashboard
- `apps/mobile` — Expo + React Native + TypeScript, Expo Router, EAS for build/update/submit (iOS + Android)

**Backend**
- `services/agent` — Python (FastAPI) on AWS Fargate. Runs the conversational tool-calling loop, calls Bedrock, persists conversation state. Python chosen for LLM ecosystem maturity.
- `services/api` — TypeScript (Next.js Route Handlers or standalone Node service). Booking CRUD, vendor management, operator dashboard API, payments orchestration. TypeScript chosen for type-sharing with frontend.
- Workers — AWS Lambda (Python or TypeScript per task) triggered by SQS. Emails, WhatsApp sends, PDF generation, payment reconciliation, scheduled reminders.

**Data plane**
- Supabase: Postgres (with `pgvector` and `PostGIS` extensions), Auth, Realtime (operator dashboard live updates), Storage (tour photos, vendor docs, traveler-uploaded passports)
- Row-level security on every tenant-scoped table, keyed on `operator_id`

**Infrastructure (AWS)**
- Bedrock (Claude), Fargate (agent), Lambda + SQS (workers), S3 (large media + backups), CloudFront (CDN), SES (transactional email), EventBridge (cross-service events), Secrets Manager

**Third-party**
- Stripe — payments, multi-currency
- WhatsApp Business API via Meta Cloud API — outbound notifications + inbound webhook ingestion in v1; each operator brings their own verified business number

**Repo and tooling**
- pnpm + Turborepo monorepo
- `packages/shared` — TypeScript types, Zod schemas, shared utilities
- `packages/agent-client` — TypeScript client generated from agent service's OpenAPI spec
- Python service has its own poetry/uv project; communicates with TS code via OpenAPI-generated clients

**Observability**
- Structured logging to CloudWatch
- Sentry for error tracking (web, mobile, both backend services)
- LangSmith or Langfuse for agent traces (decide during agent build)

## Monorepo layout

```
travelhub/
  apps/
    web-customer/        # Next.js, customer-facing white-labeled site
    web-operator/        # Next.js, operator dashboard
    mobile/              # Expo, customer mobile app
  services/
    agent/               # Python FastAPI, agent runtime
    api/                 # Optional standalone TS API if it grows beyond Next route handlers
  packages/
    shared/              # TS types, Zod schemas, shared utilities
    agent-client/        # Generated TS client for the agent service
    ui/                  # Shared UI components (web only)
  infra/                 # IaC (Terraform or CDK — TBD)
  supabase/
    migrations/          # SQL migrations
    seed/                # Seed data for dev
  docs/
    product/             # Long-form product specs by area
    architecture/        # Topology, data model, agent runtime
    adr/                 # Architecture decision records
  CLAUDE.md              # this file
```

Nested `CLAUDE.md` files exist in `apps/mobile/`, `services/agent/`, and other areas with surface-specific rules.

## Conventions

**TypeScript**
- `strict` mode on. No `any`. Use `unknown` and narrow.
- Zod schemas for all runtime validation (API inputs, environment variables, external API responses)
- Types derived from Zod schemas with `z.infer`, not duplicated
- Naming: `camelCase` for variables/functions, `PascalCase` for types/components, `SCREAMING_SNAKE_CASE` for constants
- Imports use absolute paths via tsconfig `paths`, not deep relative

**Python**
- Type hints required on all function signatures
- Pydantic models for all I/O boundaries (HTTP, tool args, tool results)
- `ruff` for linting and formatting; `pyright` for type checking
- Naming: `snake_case` everywhere except classes (`PascalCase`)

**Database**
- Postgres column and table names are `snake_case`, plural for tables (`bookings`, `traveler_profiles`)
- Every tenant-scoped table has `operator_id uuid not null references operators(id)`
- Every tenant-scoped table has RLS enabled with policy `using (operator_id = current_setting('app.operator_id')::uuid)`
- Foreign keys are `<table_singular>_id` (`booking_id`, not `booking`)
- Timestamps: `created_at`, `updated_at`, both `timestamptz`, default `now()`
- Money is always two columns: `amount_minor` (integer, in smallest currency unit) and `currency_code` (ISO 4217). Never store as a single decimal.

**API**
- REST-ish for CRUD, no GraphQL in v1
- All endpoints validate inputs with Zod (TS) or Pydantic (Python)
- All endpoints filter by `operator_id` from the authenticated session even though RLS will too — defense in depth
- State-changing endpoints accept an idempotency key
- Errors are typed (not raw strings)

**Git**
- Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`)
- One concern per PR
- PR titles match commit titles
- Squash-merge to main

**Testing**
- Integration tests for every API endpoint with at minimum: happy path, auth failure, cross-tenant access denied
- Unit tests for pricing rules, constraint engine, task generation
- Agent has its own evals suite (separate from unit tests) — see `services/agent/evals/`

## Glossary

Domain terms used throughout the codebase. When in doubt, use these names exactly.

- **Operator** — a tour management company; one tenant in the system. Has staff who use the operator dashboard.
- **Tenant** — synonym for operator at the infrastructure layer. Use *operator* in product/business contexts; use *tenant* when discussing isolation, RBAC, or data scoping.
- **Vendor** — a service provider used by operators: lodges, transport companies, drivers, naturalists, guides. Vendors are managed entities, not system users in v1. Modeled in two tiers: canonical (shared) and operator-vendor (tenant-scoped relationship).
- **Canonical vendor** — a platform-level record for a real-world vendor entity. Shared across operators, holds structural data only (location, amenities, room categories). See ADR-0002.
- **Operator-vendor** — the tenant-scoped relationship record linking one operator to one canonical vendor. Holds private commercial data: contracts, rates, internal notes, feedback. Two operators working with the same canonical vendor have separate `operator_vendor` records.
- **Traveler** — the end customer who books and goes on the trip. Distinct from *Customer* (the booking-account-holder), though usually the same person.
- **Customer** — the account holder who places a booking. May book on behalf of multiple travelers (family, group).
- **Fixed departure** — a packaged trip with predetermined dates, itinerary, and price. Sold via the e-commerce flow without agent involvement.
- **Custom trip** — a bespoke itinerary built by the agent in conversation with the customer. Distinct from fixed departure.
- **Component** — a building block of custom itineraries: a park visit, a lodge stay, a transfer, a naturalist booking, an activity. Operators maintain a components catalog used by the agent.
- **Itinerary** — a day-by-day plan, either fixed (from a departure) or dynamic (built from components).
- **Quote** — a priced itinerary in a specific currency with a validity period. Produced by the agent or generated from a fixed departure.
- **Booking** — a confirmed sale: a quote that has been paid (deposit or full). Has a state machine (confirmed → in-progress → completed → cancelled).
- **Departure** — a specific dated instance of a fixed trip (e.g., "March 15-22 Bandhavgarh Tigers"). One fixed-departure trip template can have many departures across the year.
- **Manifest** — the day-of-trip run-sheet used by the operator's ground team. Distinct from the customer-facing itinerary.
- **Permit** — a park entry permission. Named, dated, zoned, non-transferable in Indian tiger reserves. Released on specific dates (often 120 days out).
- **Park** — a wildlife reserve or destination. Has zones, closures, seasonal availability, child-age policies.
- **Zone** — a sub-area of a park (e.g., Tala, Magdhi, Khitauli in Bandhavgarh). Permits are zone-specific.
- **Closure** — a period when a park or zone is unavailable. Includes weekly closures (e.g., Monday afternoons) and seasonal closures (monsoon).
- **Naturalist** — a wildlife guide/expert assigned to a trip; often the customer's most-remembered touchpoint.
- **Gypsy** — a Mahindra Gypsy 4x4, the standard safari vehicle in Indian parks. Permits are issued per gypsy.
- **Expectation statement** — operator-authored content (e.g., on sighting probabilities, child suitability, fitness requirements) that the agent quotes verbatim, never paraphrases.
- **Task** — a unit of operator work generated from a booking. Has type, assignee, due date, status, dependencies.
- **Reminder** — a dynamic nudge fired against a task based on its current state and due date.
- **Lead traveler** — the primary contact on a booking; distinct from companions whose details are also captured.

## Roles

### Traveler

End customer. Books and goes on trips. Detailed capabilities and behaviors in `docs/product/travelers.md`.

Three entry paths to a booking:
- Browse fixed departures and book directly
- Chat with the agent to build a custom trip
- Use FAQ or WhatsApp to talk to humans

Post-booking is unified: pre-trip information forms (with operator follow-up calls if not completed by T-2 days), document delivery, in-trip support via WhatsApp, post-trip review. Customer accounts retain past trips, saved travelers, preferences, and documents.

International + domestic travelers, English language only in v1, multi-currency display and payment.

### Operator

Tour management company. Multi-staff team using the operator dashboard. Detailed capabilities and behaviors in `docs/product/operators.md`.

Three operational pillars:
- **Sales** — quote-to-closure pipeline, follow-up cadence, conversion tracking. Most v1 attention because customer pipeline is the operator's biggest visible pain.
- **Operations** — vendor management, permit lifecycle, manifest generation, day-of-trip coordination. Biggest entity surface area.
- **Customer operations** — pre-trip and during-trip relationship management, issue handling, post-trip continuity. The relationship layer that distinguishes premium operators.

The operator is the **source of truth** for everything the system says or does. Inventory, prices, policies, expectation statements, vendor capacity, closures — all configured by the operator, never invented by the system. The agent is a consumer of this truth.

Operators range from 2-3 person teams (the design partner) to 20+ person teams. RBAC is flexible: permissions → roles → users, with operator-defined custom roles. The system ships with role templates (Owner, Sales+CS, Ops, Finance) but operators can create their own.

### Vendor

Lodges, transport companies, drivers, naturalists, guides. **Not system users in v1.**

Vendors use a **two-tier model**: shared canonical entities at the platform level + operator-scoped relationship records. The same physical vendor (e.g., Sanjay Dugri Lodge) exists once in the canonical catalog and is linked to by every operator who works with it. Each operator has their own private contract, rates, internal notes, and performance data attached to their relationship. See `docs/adr/0002-vendor-data-model.md`.

- `canonical_vendors` — platform-level table. Structural facts only: name, type, location, room categories, amenities, photos. Read-only to operators. Not subject to standard RLS; accessed via a separate code path.
- `operator_vendors` — tenant-scoped. One row per operator-vendor pair. Holds the operator's display name override, status (active/paused/blacklisted), primary contact, internal notes.
- `vendor_contracts`, `vendor_rate_cards`, `vendor_feedback` — all keyed to `operator_vendors`, RLS-scoped via parent.

Operator A and Operator B can both work with the same canonical vendor without ever seeing each other's commercial terms or relationship data. Cross-operator inference from canonical changes is accepted as a low-severity tradeoff.

When a new vendor is added by an operator, the system suggests fuzzy matches from the canonical catalog. If no match, the vendor is auto-added with `is_verified=false`. Hakido staff occasionally curate, dedupe, and verify. A formal review queue is v1.5+.

Each booking generates vendor reservations against the operator's `operator_vendors` row. Operators communicate with vendors via WhatsApp/SMS/email outside the system; the system tracks status, generates reminders, and aggregates post-trip feedback per (operator, vendor) pair.

No vendor portal, no vendor login, no vendor-side calendar in v1. Vendor payouts are tracked but not paid by the system — operator handles payment outside.

Detailed model in `docs/product/vendors.md`.

## The agent

Lives in `services/agent`. Detailed design in `docs/product/agent.md`.

**Scope (v1)**
- Custom-trip planning only. Fixed departures don't use the agent.
- Capabilities: discovery and qualification, dynamic itinerary construction, quote generation and revision, information lookup, expectation setting (verbatim quoting of operator content), booking initiation, light post-booking modification handling, handoff to humans.
- Out of scope: in-trip support, complaint handling, refund authorization, vendor communication, modifying operator-configured content.

**Architecture**
- Tool-calling loop with planning. Each customer message can trigger multiple tool calls before a response is sent.
- Tools are typed (Pydantic), tenant-scoped (every call filters by `operator_id` from the conversation context), and audit-logged.
- The LLM never writes raw SQL or accesses Postgres directly. All data access goes through tools.
- State (conversation, draft itinerary, quote) persisted in Postgres after every turn.

**Per-operator configuration**
- The agent's system prompt is layered: Hakido-owned safety/protocol layer + operator-configured persona/content/rules layer + conversation-specific context layer.
- Operators configure via structured fields in the dashboard (persona, voice samples, expectation statements, policies, allowed components). They do not write raw prompts.

**Hard rules (enforced in prompt AND in tools)**
- Never invent prices, dates, availability, sighting probabilities, or vendor commitments.
- Never quote a closed park or unavailable inventory.
- Quote operator-authored content verbatim. Never paraphrase policies or expectation statements.
- Never modify operator-configured content from runtime.
- Never share data across operators.
- Never collect payment card details (Stripe handles).
- Always escalate on: explicit request, complaint about past booking, repeated tool failures, complex group bookings beyond configured limits, anything legal-sounding.

**Handoff**
- Bidirectional. Conversation has a `current_handler` field: either `agent` or a specific operator user.
- Customer-facing presentation is a consistent operator persona; handoff is framed as "bringing in a colleague," not "switching from AI to human."
- Operator picks up with full conversation context, draft itinerary, customer profile, handoff reason.

**Memory**
- Conversation-level: full history including tool calls, in LLM context window with truncation/summarization for long sessions.
- Customer-level: profile, past trips, saved travelers, preferences, documents, internal notes. Loaded on returning customer conversation start.
- No cross-customer learning, no runtime self-modification. The agent gets smarter through operator configuration changes, not opaque memory.

## The task model

Every confirmed booking generates an upfront list of tasks based on the trip's shape (parks, vendors, dates, travelers). Reminders against those tasks are dynamic.

Detailed design in `docs/product/task-system.md`.

**Generation**
- At booking confirmation, the system instantiates a task list from operator-configured templates per trip type.
- Each task has type, due date (computed from trip dates), assignee (per assignment rules), status, dependencies.
- The full list is visible from day one. Tasks become "active" as their due dates approach.
- Operators can manually add ad-hoc tasks during the booking lifecycle.

**Reminders**
- Computed dynamically against current task state. Cancelled when task completes; rescheduled when due date changes.
- Operator reminders: 2 days before due, day of due, daily after due. Channels: in-dashboard, email, optional WhatsApp.
- Customer reminders for tasks needing customer input (passport, flight details): 7 days, 3 days, 1 day before due. Channels: WhatsApp + email.
- Owner escalations: tasks overdue 3+ days appear on the owner's "needs attention" view.

**Assignment**
- Operator-configured rules: "all permit tasks → Ramesh," "everything → me," etc. Small operators bulk-assign; larger ones specialize.
- Assignment is rules-based at task generation but tasks can be reassigned manually any time.

**Why tasks are first-class**
- They're the workflow spine. Operator's home page is "my tasks today," not a chart dashboard.
- They're the bridge between the agent and the operator. Agent confirms a booking → tasks generate → operator's task list updates in real time (Supabase Realtime).
- They make implicit knowledge explicit, which is the v1 product win for operators currently running on WhatsApp and memory.

## Multi-tenancy and white-labeling

**Tenant isolation**: shared database, shared schema, RLS by `operator_id`. See `docs/adr/0001-tenant-data-isolation.md` for the full reasoning.

- Every tenant-scoped table has `operator_id` and an RLS policy.
- Every authenticated request sets `app.operator_id` from JWT claims before any query.
- API layer also filters by `operator_id` from the session — defense in depth, never trust client input.
- Cross-operator queries are forbidden in product code paths and confined to a separate `platform-admin` module used by Hakido staff.
- Automated tests assert that operator A's session cannot read or write operator B's data.

**White-labeling**: each operator at `{operator}.travelhub.hakido.co`.

- Subdomain routing identifies the operator at the edge; the rest of the stack reads `operator_id` from the resolved tenant.
- Per-operator branding: logo, colors, fonts, agent name, agent persona, hero copy, contact info.
- Per-operator configuration: trip catalog, components, expectation statements, policies, WhatsApp number, Stripe account, supported currencies.
- Custom domains (operator's own URL via CNAME) are v2.

## Always do / Never do

**Always**
- Filter by `operator_id` at the API layer in addition to RLS.
- Quote operator-authored content verbatim from the database; never paraphrase policies or expectation statements.
- Use typed tools for all agent data access. The LLM never sees raw query results.
- Persist conversation state (messages, tool calls, results, draft state) after every turn.
- Use idempotency keys on state-changing operations.
- Audit-log every agent tool call: name, args, result, latency, conversation ID.
- Represent money as `{amount_minor, currency_code}`. Never bare numbers, never floats.
- Validate proposed itineraries against operator constraints (closures, permits, capacity, party-size rules) before pricing.
- Auto-save drafts (quotes, itineraries, messages). Operators are interrupt-driven and lose context constantly.
- Generate tasks upfront at booking confirmation; recompute reminders dynamically.
- Show clear handoff state to operators: who owns this conversation right now.
- For vendor reads, always join `canonical_vendors` + the operator's `operator_vendors` row. Never expose canonical-only data without the operator's overlay where one exists.

**Never**
- Invent prices, dates, availability, sighting probabilities, or vendor commitments.
- Modify operator-configured content (catalog, policies, prices, expectations) from agent runtime.
- Cross-tenant query in product code paths. Use `platform-admin` module for cross-operator analytics.
- Store payment card details. Stripe handles, the system never sees PAN/CVV.
- Quote closed parks, unavailable inventory, or out-of-policy dates.
- Build long multi-step wizard forms. Prefer many small actions over a few big ones.
- Hardcode operator-specific assumptions. Everything that varies between operators is configuration.
- Generate UI mocks instead of real implementations. Real data, real types, real backend integration from day one.
- Use floats for money. `amount_minor` is an integer in the smallest currency unit (paise, cents).
- Promise tiger sightings, weather, or any outcome the operator hasn't explicitly authorized via expectation statements.
- Write to `canonical_vendors` from operator-facing code paths. Operator writes go to `operator_vendors`; canonical edits are platform-admin only.

## Where to find more

**Product specs** (long-form, loaded on demand)
- `docs/product/travelers.md` — capabilities, behaviors, the three entry paths
- `docs/product/operators.md` — capabilities by pillar, behaviors, RBAC model
- `docs/product/vendors.md` — managed-entity model, contracts, feedback loop
- `docs/product/agent.md` — capabilities, tools, boundaries, configuration, handoff, memory, observability
- `docs/product/task-system.md` — generation, reminders, templates, assignment

**Architecture**
- `docs/architecture/topology.md` — deployment topology with the cross-cloud Supabase ↔ AWS line
- `docs/architecture/data-model.md` — schema for tenants, bookings, conversations, vendors, tasks, etc.
- `docs/architecture/agent-runtime.md` — how the agent service runs (planning loop, tool calls, prompt assembly)

**Decisions**
- `docs/adr/0001-tenant-data-isolation.md` — why shared DB with RLS, not per-operator databases
- `docs/adr/0002-vendor-data-model.md` — why canonical vendors + operator-scoped relationships, not duplicated per-operator
- More ADRs added as decisions firm up during build

**Per-area rules**
- `apps/mobile/CLAUDE.md` — mobile-specific patterns, navigation, offline rules
- `services/agent/CLAUDE.md` — agent runtime conventions, prompt assembly, tool-writing patterns

If you're working in an area without a nested `CLAUDE.md`, the rules in this file apply.
