# Vendors

Lodges, transport companies, drivers, naturalists, guides — the service providers that operators use to deliver trips. **Vendors are not system users in v1.** They're managed entities that operators track, contract, and communicate with outside the platform.

The model is **two-tier**: shared canonical entities at the platform level, plus operator-scoped relationship records. The same physical vendor (e.g., "Sanjay Dugri Lodge in Bandhavgarh") is one canonical record that many operators link to. Each operator has their own private commercial relationship with the vendor.

See `docs/adr/0002-vendor-data-model.md` for the architectural reasoning.

## Vendor types

The system recognizes these vendor types in v1:

- **Lodge** — accommodation: lodges, resorts, hotels, guest houses, tented camps
- **Transport** — transport companies that provide vehicles for transfers and inter-park travel
- **Driver** — individual drivers (often associated with a transport company but tracked separately because the customer experience depends on the specific driver)
- **Naturalist** — wildlife guides / experts, often the customer's most-remembered touchpoint
- **Guide** — non-naturalist guides for heritage tours, treks, etc.

Each type has type-specific fields and behaviors. New types can be added via configuration as the platform expands beyond wilderness operators.

## Two-tier model

### Canonical vendors

A platform-level table (`canonical_vendors`) holds the **structural facts** about each vendor, shared across all operators who work with them.

Fields:
- **id** — uuid
- **type** — `lodge`, `transport`, `driver`, `naturalist`, `guide`
- **name** — canonical name ("Sanjay Dugri Lodge")
- **location** — PostGIS point with region tagging
- **region** — coarser geographic tag ("Bandhavgarh, MP, India")
- **structural_data** — jsonb, type-specific:
  - Lodges: room categories, room counts per category, amenities, child-suitability, accessibility, restaurant info, public photos
  - Transport: fleet types and counts, capacity per vehicle type
  - Drivers: license info if public, languages spoken, years of experience
  - Naturalists: specializations (big cats, birds, etc.), languages, years of experience, certifications
  - Guides: specializations, languages, certifications
- **photos** — array of public photo URLs (operator-uploaded, shared)
- **public_contact** — general phone, email, website if the vendor publishes them
- **created_by_operator_id** — provenance, not ownership
- **is_verified** — bool: false for operator-submitted, true after Hakido verification (manual in v1)
- **created_at**, **updated_at**

Canonical vendor records are **read-only to operators** through normal application code paths. Only Hakido staff can edit canonical records, via a separate platform-admin code path.

### Operator-vendor relationships

A tenant-scoped table (`operator_vendors`) holds **the operator's private relationship** with each canonical vendor they work with.

Fields:
- **id** — uuid
- **operator_id** — tenant scope (RLS)
- **canonical_vendor_id** — references `canonical_vendors`
- **display_name** — optional override if the operator calls the vendor differently
- **status** — `active`, `paused`, `blacklisted`
- **primary_contact** — operator's specific contact at this vendor (jsonb: name, phone, WhatsApp, email, role)
- **secondary_contacts** — jsonb array of additional contacts
- **internal_notes** — free-text notes for the team
- **performance_summary** — jsonb cached aggregate (recent rating averages, complaint tags, booking count) — refreshed on a schedule from `vendor_feedback`
- **created_at**, **updated_at**
- Unique constraint on `(operator_id, canonical_vendor_id)`
- RLS policy: `using (operator_id = current_setting('app.operator_id')::uuid)`

### Contracts and rate cards

Operators have **contracts** with vendors — validity periods, payment terms, cancellation terms — and **rate cards** under those contracts.

`vendor_contracts`:
- **id** — uuid
- **operator_vendor_id** — references `operator_vendors`
- **valid_from**, **valid_until** — date range
- **payment_terms** — text or structured (e.g., "30% advance, balance 7 days before arrival")
- **cancellation_terms** — text
- **documents** — jsonb array of attached documents (S3 references)
- **status** — `draft`, `active`, `expired`, `terminated`
- **created_at**, **updated_at**
- RLS inherited via parent

`vendor_rate_cards`:
- **id** — uuid
- **operator_vendor_id** — references `operator_vendors`
- **contract_id** — references `vendor_contracts` (optional; rate cards may exist without formal contracts)
- **season_start**, **season_end** — date range (e.g., Oct 1 – June 30 for safari peak)
- **category** — type-specific (e.g., room category for lodges, vehicle type for transport)
- **rate_amount_minor** — integer, in smallest currency unit
- **rate_currency_code** — ISO 4217
- **inclusions** — jsonb array (breakfast, dinner, park fees, etc.)
- **min_nights** — applicable to lodges
- **child_pricing** — jsonb (different rates per age band)
- **notes** — free text
- **created_at**, **updated_at**
- RLS inherited

A lodge in peak season vs. shoulder vs. monsoon will typically have multiple rate cards covering each period. The pricing engine selects the applicable rate based on trip dates.

### Vendor feedback

Per (operator_vendor, booking) pair, ratings and notes captured post-trip from travelers.

`vendor_feedback`:
- **id** — uuid
- **operator_vendor_id** — references `operator_vendors`
- **booking_id** — references `bookings`
- **traveler_id** — references travelers (the specific person who left feedback)
- **rating** — integer 1-5
- **category** — type-specific (e.g., for lodges: `food`, `cleanliness`, `service`, `room_quality`, `location`; for naturalists: `knowledge`, `communication`, `attention`)
- **notes** — free text
- **created_at**
- RLS inherited

The aggregation feeds `operator_vendors.performance_summary`, refreshed on a schedule (nightly or after each new feedback record).

## Operator workflows

### Adding a vendor

1. Operator goes to "Add a vendor" in their dashboard
2. They search by name, type, region
3. System suggests canonical matches via fuzzy matching (name + region similarity)
4. Operator confirms a match → creates an `operator_vendors` row linking to the existing canonical record
5. If no match fits, operator creates a new canonical record (auto-approved with `is_verified=false`) and the operator-vendor relationship in the same transaction
6. Operator fills in their private fields: contract, rates, contact, notes

### Editing a vendor

- **Operator-private fields** (display_name, status, contacts, notes, contracts, rates, feedback) — operator can edit freely
- **Canonical structural fields** (name, location, room categories, photos) — operator cannot edit. They can submit a correction request through a dashboard form; Hakido staff review and apply if appropriate.

### Vendor reservations

Each booking generates **vendor reservations** as part of task generation. A `vendor_reservation` record represents "this booking has booked this lodge for these dates" or "this booking has booked this naturalist for these mornings."

`vendor_reservations`:
- **id** — uuid
- **operator_id** — tenant scope (RLS)
- **booking_id** — references `bookings`
- **operator_vendor_id** — references `operator_vendors`
- **service_type** — type-specific (room category, vehicle type, naturalist day, etc.)
- **start_date**, **end_date**
- **quantity** — number of rooms, vehicles, days, etc.
- **status** — `requested`, `confirmed`, `paid`, `cancelled`, `completed`
- **rate_card_id** — references the rate card used for pricing
- **cost_amount_minor**, **cost_currency_code** — what the operator owes the vendor for this reservation
- **vendor_reference** — vendor's own booking reference if they provide one
- **notes** — free text
- **created_at**, **updated_at**
- Status transitions are tracked in the booking's audit log

The vendor reservation moves through statuses as the operator confirms with the vendor (typically via WhatsApp or call — outside the system in v1). The operator updates status manually as confirmations come in. Confirmation tasks remind them to do so.

### Communicating with vendors

Outbound messages to vendors:
- Booking-request templates ("can you confirm 3 nights from March 15-18 for 4 guests, 2 rooms")
- Confirmation reminders for tasks that haven't moved
- Day-before-arrival heads-up
- Post-trip thank-you / feedback request

These are sent via WhatsApp, SMS, or email per the vendor's primary contact preference. WhatsApp uses the operator's verified business number; SMS and email use operator-configured senders.

In v1, **vendors do not reply through the system**. Their replies arrive via WhatsApp on the operator's phone, email, or phone calls. Operators manually update the vendor reservation status based on what they hear back.

In v1.5+, the WhatsApp inbound webhook may also capture vendor replies into the dashboard timeline (alongside customer replies), but this requires distinguishing vendor numbers from customer numbers — a moderate complexity that's deferred.

### Performance tracking

Post-trip feedback aggregates per (operator_vendor) pair. The operator sees:

- Average ratings over time
- Recurring complaint themes (tagged categories)
- Booking volume and revenue
- Last booking date, trend over recent months

This drives operator decisions: rotate underperformers out, send more business to top performers, raise issues with vendors whose service is slipping.

The operator owns this performance data. It is **not** shared with other operators in v1, even though all operators link to the same canonical vendor. Operator A's experience with Sanjay Dugri Lodge is not visible to Operator B.

### Vendor payouts

The system tracks what the operator owes each vendor per booking (via `vendor_reservations.cost_amount_minor`) and aggregates into a payables view.

The operator can:
- See total payable per vendor per period
- See per-booking breakdown
- Mark payments made (date, amount, method, reference) — a free-form payment record
- Generate vendor settlement reports for export to Tally or accountant

The system **does not pay vendors** in v1. Payment happens through the operator's bank, UPI, or cheque outside the platform. The system is a tracking and reporting tool for payables.

## Inventory ownership models

Different vendors have different relationships with the operator regarding inventory:

- **Hard allocation** — the vendor has reserved capacity for the operator (e.g., "5 rooms always held for you, you sell, we honor"). Operator can sell against this without per-booking confirmation.
- **Soft allocation** — the vendor honors operator bookings but on a first-come-first-served basis against their general inventory. Operator should request before committing to customer.
- **Request-based** — every booking requires explicit vendor confirmation before the operator commits.

These models are configured per (operator_vendor, service_type) — a lodge might give hard allocation on standard rooms but soft allocation on premium suites. The configuration affects the agent's behavior:

- For hard allocation: agent can confirm bookings without waiting for vendor confirmation. The vendor reservation is created `confirmed` automatically.
- For soft allocation: agent can quote, but the booking creates the reservation as `requested`. An operator task is generated to confirm with the vendor.
- For request-based: agent quotes with a "subject to vendor confirmation" caveat. Booking and reservation both held until operator confirms.

In v1, the system supports all three modes but with **simple configuration**. Sophisticated rules ("hard allocation Oct-March, soft April-June, request-based monsoon") are operator-managed via multiple configurations rather than complex rule trees.

## Conflict detection

The system surfaces basic conflicts in v1:

- Same lodge, same date, same room category over-allocated across bookings (vendor reservation overlap)
- Same naturalist booked across overlapping trips
- Same driver/vehicle across overlapping trips
- A vendor with status `paused` or `blacklisted` being used in a new booking

Conflicts surface in:
- The operator's "needs attention" view
- The relevant booking's detail page
- The agent's `validateItinerary` tool — proposed itineraries with conflicts return as invalid with explanations

Sophisticated conflict resolution (suggesting alternatives, re-allocating across bookings) is v2.

## Reissue and replan

When a vendor cancels — lodge over-booked, driver sick, naturalist unavailable — the operator needs to:
1. Find a replacement
2. Update the booking's vendor reservation
3. Inform the customer
4. Possibly issue a credit, upgrade, or refund

In v1, this is a **human-driven workflow** with system support:
- Operator marks the existing vendor reservation as `cancelled` with a reason
- A task is auto-generated: "Find replacement for [vendor] in [booking]"
- Operator manually finds and books a replacement (creates a new vendor reservation)
- Operator decides on customer compensation per their policy
- A communication task is generated to inform the customer

The system tracks the change history but doesn't automate the recovery. Sophisticated replan workflows ("automatically suggest a comparable lodge in the same region") are v2+.

## Hakido's role with the canonical catalog

The canonical vendor catalog is a platform-level asset that Hakido owns and curates. Responsibilities:

- **Verification** — staff review of operator-submitted vendors, marking `is_verified=true` when validated
- **Deduplication** — when two operators add what appears to be the same vendor under slightly different names, Hakido staff merge the records (the "Sanjay Dugri Lodge" / "Sanjay Dugri Resort" problem)
- **Updates** — when structural data changes (vendor renovates, adds a category), Hakido updates canonical records
- **Photo curation** — operator-uploaded photos enter a moderation queue before being added to canonical records
- **Correction requests** — handle operator submissions for canonical edits

In v1, this is a **manual lightweight process** by Hakido staff. Tooling for this lives in the platform-admin module, separate from the operator dashboard. As the catalog grows, automation grows with it (v1.5+: formal review queues, automated dedup suggestions, etc.).

## Privacy and isolation guarantees

Restating from ADR-0002 because it bears repeating:

1. **Canonical vendor records** are shared, read-only-to-operators. Operators see them but never see who else is using them or any commercial data attached by other operators.
2. **Operator-vendor relationships** are strictly tenant-scoped. Operator A cannot read Operator B's relationships, contracts, rates, internal notes, or feedback under any condition.
3. **Cross-tenant query patterns** are forbidden in product code. Aggregated platform-level statistics (e.g., "this lodge has 12 operator relationships") are computed in the platform-admin code path and exposed only to Hakido staff in v1.
4. **The application layer enforces these boundaries** in addition to RLS, per the defense-in-depth principle.

## What's out of v1 scope

- Vendor portal or vendor-side login
- Vendor-managed availability calendars
- Vendor-side acceptance/rejection of booking requests
- Automated vendor payment processing
- Inbound WhatsApp ingestion specifically for vendor replies (v1 ingests inbound WhatsApp generally; differentiating vendor numbers from customer numbers is v1.5+)
- Cross-operator vendor performance sharing (each operator's data stays private)
- Public review aggregation displayed on customer-facing site (v2)
- Sophisticated availability matching across multiple lodges to suggest alternatives
- Vendor invoicing tools (operators handle invoices through Tally / their accountant)
- Per-vendor SLA tracking and enforcement

## Open questions

- **Naturalist scheduling granularity.** Some operators schedule naturalists by morning/afternoon safari (sub-day), others by day. v1 supports day-level; sub-day requires more nuanced scheduling. Re-evaluate after design partner feedback.
- **Driver vs. transport company.** Currently modeled as separate vendor types. Some operators might want them merged ("the driver is just an attribute of the transport reservation"). Configurable per operator could be future enhancement.
- **Vendor capacity tracking.** Today: lodge has N rooms in category X. Tomorrow: how do we know how many are sold to other operators (we don't, by design — that's their private business). The hard/soft allocation model is the workaround. Whether this is sufficient at scale is TBD.
