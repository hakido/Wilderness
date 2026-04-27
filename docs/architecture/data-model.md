# Data model

First-pass schema for TravelHub. This is the v1 design target — concrete enough to build against, abstract enough that detailed field decisions during implementation refine it without breaking the shape.

The schema is organized into:
- **Platform tables** — outside the regular RLS regime, owned by Hakido
- **Tenant tables** — RLS-scoped by `operator_id`
- **Reference / lookup tables** — small, mostly static, read-only to operators

All tables follow conventions in `CLAUDE.md`: snake_case names, plural tables, `created_at`/`updated_at` as `timestamptz`, foreign keys as `<entity>_id`, money as `{amount_minor, currency_code}`.

## Platform tables

### operators

The tenants. One row per operator (tour management company).

```
operators
  id uuid pk
  name text                          -- "Wilderness Co"
  slug text unique                   -- "wildernessco" → subdomain
  base_currency_code text            -- ISO 4217, e.g., "INR"
  display_currencies text[]          -- e.g., ["INR", "USD", "EUR"]
  status enum                        -- active, suspended, churned
  branding jsonb                     -- logo url, colors, fonts
  agent_config jsonb                 -- compiled agent configuration
  whatsapp_business_number text
  whatsapp_business_id text
  stripe_account_id text             -- Stripe Connect account
  email_sender_domain text
  gst_number text                    -- for Indian operators
  legal_address jsonb
  created_at, updated_at
```

No `operator_id` (this *is* the operator). Read access via subdomain resolution at the edge.

### canonical_vendors

Shared vendor catalog. See `vendors.md` and ADR-0002.

```
canonical_vendors
  id uuid pk
  type vendor_type                   -- enum: lodge, transport, driver, naturalist, guide
  name text
  location geography(point)          -- PostGIS
  region text
  structural_data jsonb              -- type-specific fields
  photos text[]
  public_contact jsonb
  created_by_operator_id uuid        -- provenance, not ownership
  is_verified boolean default false
  created_at, updated_at
```

No RLS. Read-only to operators via application code; writes only through platform-admin paths.

### parks

Reference data for parks, treated as platform-level given it's domain-shared knowledge that operators consume but don't define.

```
parks
  id uuid pk
  name text                          -- "Bandhavgarh"
  state text                         -- "Madhya Pradesh"
  country text                       -- "India"
  location geography(point)
  zones jsonb                        -- array of zones with names, descriptions
  default_closures jsonb             -- weekly closures, seasonal closures (rough; operator-overridable)
  metadata jsonb                     -- coarse facts (size, primary species, etc.)
  created_at, updated_at
```

Operators configure park-specific overrides in their own tables (closures specific to their permits, expectation statements, etc.) but the base park data is shared.

### currencies

Reference data.

```
currencies
  code text pk                       -- ISO 4217: USD, INR, EUR, GBP, AUD
  name text                          -- "Indian Rupee"
  minor_unit_factor integer          -- 100 for INR/USD/EUR, 1000 for some others
  symbol text                        -- "₹", "$", "€"
```

### fx_rates

Daily-refreshed exchange rates. Workers update; agents read.

```
fx_rates
  id uuid pk
  from_currency text                 -- references currencies.code
  to_currency text                   -- references currencies.code
  rate numeric(20, 10)
  source text                        -- "openexchangerates.org" etc.
  fetched_at timestamptz
  unique (from_currency, to_currency, fetched_at::date)
```

## Tenant tables

All have `operator_id uuid not null references operators(id)` and RLS policy `using (operator_id = current_setting('app.operator_id')::uuid)`. Indices on `operator_id` for query performance.

### users

Operator team members. Distinct from customers.

```
users
  id uuid pk
  operator_id uuid not null
  email text not null
  name text
  phone text
  whatsapp text
  status enum                        -- active, invited, suspended, removed
  out_of_office_until date
  out_of_office_delegate_user_id uuid
  notification_preferences jsonb
  created_at, updated_at
  unique (operator_id, email)
```

### roles

```
roles
  id uuid pk
  operator_id uuid not null
  name text                          -- "Owner", "Sales", custom names
  description text
  permissions text[]                 -- array of permission strings
  is_system_template boolean         -- true for the built-in templates
  created_at, updated_at
```

### user_roles

Many-to-many.

```
user_roles
  id uuid pk
  operator_id uuid not null
  user_id uuid not null references users(id)
  role_id uuid not null references roles(id)
  created_at
  unique (user_id, role_id)
```

### customers

End-customer records. Per-operator (no cross-operator customer identity in v1).

```
customers
  id uuid pk
  operator_id uuid not null
  email text
  phone text
  whatsapp text
  name text
  preferred_language text default 'en'
  preferred_currency_code text
  preferences jsonb                  -- dietary, accessibility, photography_focus, etc.
  vip_flag boolean default false
  total_bookings_count integer default 0
  total_spend_minor bigint default 0
  total_spend_currency_code text
  first_booked_at timestamptz
  last_booked_at timestamptz
  created_at, updated_at
  unique (operator_id, email)
```

### travelers

Individual travelers within a booking. Related to customers but distinct (a customer may book for many travelers, including children without their own accounts).

```
travelers
  id uuid pk
  operator_id uuid not null
  customer_id uuid                   -- the booking-account-holder; nullable if traveler is a companion
  first_name, last_name text
  date_of_birth date
  gender text
  passport_number text               -- encrypted at column level
  passport_country text
  passport_expiry date
  nationality text
  dietary text
  medical_disclosures text
  emergency_contact jsonb
  insurance_policy_number text
  insurance_expiry date
  is_lead_traveler boolean
  created_at, updated_at
```

### saved_travelers

Travelers a customer has saved for reuse across bookings (family members, frequent companions).

```
saved_travelers
  id uuid pk
  operator_id uuid not null
  customer_id uuid not null
  first_name, last_name, date_of_birth, gender
  relationship text                  -- "spouse", "child", "friend"
  -- subset of travelers fields, no booking link
  created_at, updated_at
```

### conversations

Chat conversations between customers and the agent (or operator team).

```
conversations
  id uuid pk
  operator_id uuid not null
  customer_id uuid                   -- nullable until customer is identified
  current_handler text               -- 'agent' or user_id
  status enum                        -- active, dormant, archived
  channel text                       -- 'web', 'mobile', 'whatsapp', 'email'
  source text                        -- where this conversation originated
  draft_itinerary_id uuid            -- references current draft
  active_quote_id uuid               -- references current quote
  metadata jsonb                     -- last message at, message count, etc.
  created_at, updated_at
```

### messages

Individual messages in conversations. Includes both customer-agent dialog and tool calls.

```
messages
  id uuid pk
  operator_id uuid not null
  conversation_id uuid not null references conversations(id)
  role enum                          -- customer, agent, operator_user, system
  content_type enum                  -- text, structured (cards, comparisons), tool_call, tool_result
  content jsonb                      -- payload depends on content_type
  sender_user_id uuid                -- if role = operator_user
  channel text                       -- 'web', 'whatsapp', 'email'
  external_message_id text           -- WhatsApp message ID, email message ID for ingested messages
  reply_to_message_id uuid
  created_at
```

### tool_calls

Audit trail of agent tool calls. Linked to messages but separate for query efficiency.

```
tool_calls
  id uuid pk
  operator_id uuid not null
  conversation_id uuid not null references conversations(id)
  message_id uuid                    -- the agent message this tool call belongs to
  tool_name text                     -- 'getComponents', 'priceItinerary', etc.
  arguments jsonb
  result jsonb
  status enum                        -- success, error, timeout
  error_message text
  latency_ms integer
  created_at
```

### itineraries

Draft and confirmed itineraries.

```
itineraries
  id uuid pk
  operator_id uuid not null
  customer_id uuid
  conversation_id uuid               -- nullable; for agent-built itineraries
  type enum                          -- fixed_departure, custom
  fixed_departure_id uuid            -- references fixed_departures if type=fixed
  status enum                        -- draft, quoted, accepted, expired, abandoned, booked
  party_size integer
  trip_start_date date
  trip_end_date date
  components jsonb                   -- array of {component_id, dates, quantity, ...}
  day_by_day jsonb                   -- structured day-by-day plan
  notes text
  parent_itinerary_id uuid           -- if this is a revision
  version integer
  created_at, updated_at
```

### quotes

Priced versions of itineraries.

```
quotes
  id uuid pk
  operator_id uuid not null
  itinerary_id uuid not null references itineraries(id)
  conversation_id uuid
  customer_id uuid
  status enum                        -- draft, sent, viewed, accepted, expired, rejected
  total_amount_minor bigint
  base_currency_code text            -- the operator's base currency
  display_amount_minor bigint
  display_currency_code text         -- the customer's chosen display currency
  fx_rate numeric(20, 10)            -- locked at quote time
  fx_rate_fetched_at timestamptz
  line_items jsonb                   -- breakdown
  taxes jsonb                        -- GST etc.
  cancellation_policy_text text      -- snapshot of operator's policy at quote time
  expectation_statements jsonb       -- snapshot of relevant statements
  valid_from timestamptz
  valid_until timestamptz
  inventory_held_until timestamptz
  shareable_link_token text          -- for "send to my wife"
  parent_quote_id uuid               -- for versions
  version integer
  created_at, updated_at
```

### bookings

Confirmed sales.

```
bookings
  id uuid pk
  operator_id uuid not null
  customer_id uuid not null
  quote_id uuid                      -- the quote that became this booking
  itinerary_id uuid                  -- final itinerary
  type enum                          -- fixed_departure, custom
  status enum                        -- confirmed, in_progress, completed, cancelled
  trip_start_date date
  trip_end_date date
  party_size integer
  primary_owner_user_id uuid         -- operator team member who owns this booking
  total_amount_minor bigint
  currency_code text
  amount_paid_minor bigint
  amount_due_minor bigint
  payment_status enum                -- unpaid, deposit_paid, paid, refunded, partial_refund
  cancellation_at timestamptz
  cancellation_reason text
  cancellation_policy_snapshot jsonb -- frozen at booking time
  created_at, updated_at
```

### fixed_departures

Operator-published trip products. Distinct from generic templates (a trip template can have many specific dated fixed_departures).

```
fixed_departure_templates
  id uuid pk
  operator_id uuid not null
  name text                          -- "Bandhavgarh Tigers - 5 Days"
  description text
  duration_days integer
  components jsonb                   -- the standard component set
  base_price_minor bigint
  base_currency_code text
  inclusions text[]
  exclusions text[]
  photos text[]
  status enum                        -- draft, published, archived
  created_at, updated_at

fixed_departures
  id uuid pk
  operator_id uuid not null
  template_id uuid not null references fixed_departure_templates(id)
  start_date date
  end_date date
  capacity integer
  bookings_count integer default 0
  price_overrides jsonb              -- if this specific departure has different pricing
  status enum                        -- open, full, departed, cancelled
  created_at, updated_at
```

### components_catalog

The components the agent can use to build custom trips. Polymorphic by type.

```
components
  id uuid pk
  operator_id uuid not null
  type vendor_type                   -- lodge, transport, naturalist, etc.
  operator_vendor_id uuid            -- references operator_vendors; the underlying entity
  name text                          -- operator-curated display name (may differ from canonical)
  description text
  customer_facing_data jsonb         -- photos, descriptions, what's included shown to customers
  internal_data jsonb                -- operator-only data (not shown to customers)
  pricing_rules jsonb                -- structured rules for pricing
  availability_rules jsonb           -- minimum nights, party size constraints, etc.
  status enum                        -- active, paused, archived
  created_at, updated_at
```

### parks_operator_overrides

Operators can override or extend the platform park data with their own permits, closures, expectations.

```
parks_operator_data
  id uuid pk
  operator_id uuid not null
  park_id uuid not null references parks(id)
  custom_closures jsonb              -- operator-specific closures
  expectation_statements jsonb       -- per-park expectation statements
  child_age_policy jsonb
  fitness_requirements jsonb
  internal_notes text
  created_at, updated_at
  unique (operator_id, park_id)
```

### permits

Permit allocations and assignments.

```
permits
  id uuid pk
  operator_id uuid not null
  park_id uuid not null references parks(id)
  zone text
  date date
  session enum                       -- morning, afternoon, full_day
  vehicle_type text                  -- gypsy, canter
  capacity integer                   -- per-vehicle capacity
  allocated_count integer            -- number assigned to bookings
  status enum                        -- available, applied, confirmed, cancelled
  application_reference text
  cost_amount_minor bigint
  cost_currency_code text
  released_on date                   -- when the park released these permits
  applied_on date
  confirmed_on date
  created_at, updated_at

permit_assignments
  id uuid pk
  operator_id uuid not null
  permit_id uuid not null references permits(id)
  booking_id uuid not null references bookings(id)
  traveler_count integer
  traveler_names text[]              -- for the named-permit requirement
  created_at
```

### operator_vendors

The operator's relationship with canonical vendors. See `vendors.md`.

```
operator_vendors
  id uuid pk
  operator_id uuid not null
  canonical_vendor_id uuid not null references canonical_vendors(id)
  display_name text
  status enum                        -- active, paused, blacklisted
  primary_contact jsonb
  secondary_contacts jsonb
  internal_notes text
  performance_summary jsonb          -- cached aggregate
  inventory_model enum               -- hard_allocation, soft_allocation, request_based
  created_at, updated_at
  unique (operator_id, canonical_vendor_id)
```

### vendor_contracts, vendor_rate_cards, vendor_feedback, vendor_reservations

See `vendors.md` for full field lists.

### tasks

Operator workflow tasks. See `task-system.md`.

```
tasks
  id uuid pk
  operator_id uuid not null
  booking_id uuid                    -- nullable for non-booking tasks
  template_id uuid                   -- nullable for manual tasks
  type text
  title text
  description text
  assignee_user_id uuid
  due_at timestamptz
  status enum                        -- pending, in_progress, blocked, completed, cancelled
  priority enum                      -- low, normal, high, critical
  source enum                        -- auto, manual
  dependencies uuid[]                -- references to other task ids
  completed_at timestamptz
  completed_by_user_id uuid
  completion_notes text
  metadata jsonb
  created_at, updated_at
```

### task_templates

```
task_templates
  id uuid pk
  operator_id uuid not null
  type text
  applies_to jsonb                   -- conditions for when this template generates tasks
  title_template text
  description_template text
  due_relative_to enum               -- booking_confirmed, trip_start, trip_end, payment_due
  due_offset_days integer            -- negative for "before"
  default_priority enum
  assignment_rule jsonb              -- how to assign generated tasks
  dependency_template_ids uuid[]
  reminder_schedule jsonb
  is_system_template boolean
  status enum                        -- active, paused
  created_at, updated_at
```

### reminders_log

History of fired reminders. Useful for debugging and reporting.

```
reminders_log
  id uuid pk
  operator_id uuid not null
  task_id uuid not null references tasks(id)
  reminder_type text                 -- "operator_2_days_before", "customer_7_days_before", etc.
  channel text                       -- in_dashboard, email, whatsapp, push
  recipient_user_id uuid             -- for operator reminders
  recipient_customer_id uuid         -- for customer reminders
  status enum                        -- sent, delivered, failed
  delivery_metadata jsonb
  fired_at timestamptz
```

### issues

Mid-trip and operational issues.

```
issues
  id uuid pk
  operator_id uuid not null
  booking_id uuid
  reported_by enum                   -- traveler, operator_user, vendor
  reported_by_traveler_id uuid
  reported_by_user_id uuid
  category text                      -- room, food, vehicle, driver, naturalist, weather, etc.
  severity enum                      -- low, medium, high, critical
  status enum                        -- open, in_progress, resolved, closed
  description text
  resolution_notes text
  assigned_to_user_id uuid
  resolved_at timestamptz
  resolved_by_user_id uuid
  created_at, updated_at
```

### communications_templates

Operator-defined message templates for WhatsApp, email, SMS.

```
communications_templates
  id uuid pk
  operator_id uuid not null
  channel enum                       -- whatsapp, email, sms
  trigger text                       -- 'booking_confirmed', 'payment_reminder', 'pre_trip_briefing'
  recipient_type enum                -- customer, vendor, operator_user
  language text default 'en'
  whatsapp_template_id text          -- Meta-approved template ID for WhatsApp
  subject text                       -- for email
  body_template text
  variables jsonb                    -- list of expected variables
  status enum                        -- draft, active, deprecated
  created_at, updated_at
```

### communications_log

Outbound message history.

```
communications_log
  id uuid pk
  operator_id uuid not null
  template_id uuid references communications_templates(id)
  channel enum
  recipient_type enum
  recipient_id uuid                  -- customer_id, vendor_id, or user_id depending on type
  recipient_address text             -- email address, phone number
  subject text
  body text                          -- compiled, ready-to-send content
  status enum                        -- queued, sent, delivered, failed, bounced
  delivery_metadata jsonb
  external_message_id text
  related_booking_id uuid
  related_task_id uuid
  created_at, sent_at
```

### audit_log

Catch-all audit log for sensitive actions (role changes, refund approvals, contract edits, etc.).

```
audit_log
  id uuid pk
  operator_id uuid not null
  user_id uuid                       -- who did it
  action text                        -- "user_role_assigned", "vendor_contract_edited"
  entity_type text
  entity_id uuid
  before jsonb
  after jsonb
  metadata jsonb
  created_at
```

### platform_admin_audit_log

Separate audit log for Hakido platform-admin actions, keyed differently.

```
platform_admin_audit_log
  id uuid pk
  hakido_user_id uuid
  action text
  affected_operator_id uuid          -- which operator's data was touched
  entity_type text
  entity_id uuid
  before, after jsonb
  metadata jsonb
  created_at
```

## Indexes (key ones)

Beyond primary keys and `operator_id` indexes, query-driven indexes:

- `bookings (operator_id, trip_start_date)` — daily ops view
- `bookings (operator_id, status)` — pipeline summaries
- `tasks (operator_id, assignee_user_id, status, due_at)` — "my today" queries
- `tasks (operator_id, status, due_at) where status in ('pending', 'in_progress')` — overdue scans
- `messages (conversation_id, created_at desc)` — conversation timeline
- `conversations (operator_id, current_handler, status)` — operator inbox
- `customers (operator_id, email)` — customer lookup
- `vendor_reservations (operator_id, booking_id)` — booking detail page
- `permits (operator_id, park_id, date)` — permit calendar
- `audit_log (operator_id, entity_type, entity_id, created_at desc)` — audit trail per entity

## Constraints and policies

### RLS pattern

Every tenant-scoped table has identical policy:

```sql
alter table <table_name> enable row level security;
create policy tenant_isolation on <table_name>
  using (operator_id = current_setting('app.operator_id')::uuid);
```

`app.operator_id` is set per-connection from the JWT or session context before any query runs. The setting is reset between connections via pgbouncer's transaction pooling.

### Defense in depth

Application code never trusts client-supplied `operator_id`. The value used in `app.operator_id` and in API-layer filtering comes from the authenticated session (JWT claim, set during auth), not from request bodies or URL parameters.

### Soft delete

Most tables don't use soft delete; bookings, customers, conversations are kept indefinitely (with separate retention policies for archival). Tables that do use soft delete add a `deleted_at timestamptz` column and exclude deleted rows from queries via view layer.

### Encryption at rest

- Postgres: full-database encryption via Supabase
- Sensitive columns (passport_number, credit card data — though we don't store the latter): column-level encryption via pgcrypto for additional protection

### PII handling

- Passport numbers, dates of birth, medical disclosures are PII
- Customer requests for deletion (under GDPR or similar) require: anonymizing personal fields while preserving booking records for accounting (auditable transactions can't disappear)
- The system has a `privacy_anonymized_at` flag on customers and travelers to mark anonymization

## Migration strategy

- Migrations live in `supabase/migrations/` as numbered SQL files
- Forward-only migrations (no down migrations); rollback is a new forward migration
- Supabase CLI manages application
- All migrations tested in staging before prod
- For destructive changes (drop column, drop table), a multi-step migration: deploy code that doesn't use the column, then drop in a follow-up migration

## What's not in this document

- Specific column types for every nuanced field (jsonb shapes inside structural_data, components, etc.) — those evolve with implementation
- Indexes for performance tuning beyond the obvious — added during build based on query patterns
- Specific check constraints — added during implementation
- The platform-admin schema for Hakido staff tooling — separate from this doc, lives in `docs/architecture/platform-admin.md` (forthcoming)
- Specific enum values — written explicitly in migrations

This is the *shape* of the data; the *details* arrive with the migrations.
