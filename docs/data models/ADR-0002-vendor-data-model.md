# ADR-0002: Vendor data model — shared canonical entities with operator-scoped relationships

- **Status**: Accepted
- **Date**: 2026-04-27
- **Deciders**: Hakido engineering
- **Project**: TravelHub
- **Related**: ADR-0001 (tenant data isolation)

## Context

Vendors in TravelHub are service providers used by operators: lodges, transport companies, drivers, naturalists, guides. In the real world, the same vendor — say, Sanjay Dugri Lodge in Bandhavgarh — is used by many tour operators simultaneously. Each operator has a separate commercial relationship with the same physical entity.

The data splits cleanly into two layers:

**Shared, structural** — true regardless of which operator is asking:
- The vendor exists, has a name, type, physical location, room categories or vehicle types, capacity, amenities, public contact details, photos.

**Per-operator, commercial and relational** — varies by operator:
- Contracts (different rates, validity periods, payment terms)
- Rate cards (seasonal pricing per operator)
- Internal notes ("Vikram handles bookings now, prefers WhatsApp")
- Performance data from this operator's customers
- Status (active / paused / blacklisted by this operator)
- Documents (signed contracts, MoUs)

Three modeling patterns were considered. The naive default — every operator has a fully-private copy of every vendor — duplicates structural data across operators, causes drift, makes Hakido-level features impossible, and fights the reality of the domain. The opposite — fully-shared records — leaks commercial data across competitors. The middle path is to split the model.

## Decision

**We will use a two-tier vendor data model: a shared canonical vendor catalog at the platform level, and operator-scoped relationship records linking each operator to the canonical vendors they use.**

### Schema shape

`canonical_vendors` (platform-level, no `operator_id`)
- id, type, name, location (PostGIS), region, structural_data (jsonb), photos, public_contact, created_by_operator_id, is_verified, timestamps
- Read-only to operators via the application layer; writable only through a curated submission flow
- Outside the regular RLS regime; accessed via a separate code path

`operator_vendors` (tenant-scoped, RLS by `operator_id`)
- id, operator_id, canonical_vendor_id, display_name (override), status, primary_contact, internal_notes, timestamps
- Unique constraint on (operator_id, canonical_vendor_id)
- Standard RLS policy

`vendor_contracts`, `vendor_rate_cards`, `vendor_feedback`
- All keyed to `operator_vendor_id`
- Inherit tenant scope via the parent `operator_vendors` row
- Standard RLS

### Onboarding flow for new vendors (v1)

1. Operator searches the canonical catalog when adding a vendor (by name, region, type).
2. The system suggests possible matches using fuzzy name + region matching.
3. Operator confirms a match and creates their `operator_vendors` row, or chooses "this is a new vendor" if no match fits.
4. If new, the operator's submitted vendor is auto-added to `canonical_vendors` with `is_verified=false`.
5. Hakido has read access to all `is_verified=false` records and can verify, edit, or merge duplicates as a background curation activity.
6. v1.5 may add a formal Hakido review queue with explicit approve/reject; v1 is auto-approve with manual cleanup.

### Read pattern

The agent and the operator dashboard fetch a vendor view that joins canonical structural data + this operator's relationship + applicable contracts and rate cards. RLS ensures only this operator's relational data is visible. The canonical join is the same canonical row across all operators, by design.

## Alternatives considered

### Pattern 1: Fully duplicated, per-operator vendors

Rejected. Every operator gets a private vendor record with no shared identity. Simple data model, trivial isolation, but:

- Structural data duplicates and drifts. A lodge renovation updates one operator's record; others go stale.
- Photos can't be shared. Each operator re-uploads.
- Hakido has no platform-level vendor knowledge. Cannot build network features (canonical vendor directory, market-rate intelligence, cross-operator vendor discovery) without rebuilding from scratch.
- Onboarding new operators is slower — they start with an empty vendor list and rebuild what other operators already have.

The duplication problem isn't theoretical. After 5-10 operators are onboarded, the same physical lodges exist as 5-10 inconsistent records with diverging data.

### Pattern 3: Hybrid — duplicated by default, optional canonical link

Rejected. Each operator owns their vendor record (Pattern 1) but can optionally link to a canonical entity. In practice this becomes the worst of both worlds: Hakido maintains canonical data for the operators who opt in, while operators who don't opt in still produce duplicates that drift. Most systems that started here ended up wishing they'd committed to Pattern 2 from the start.

### Operator-private clones of canonical records

Rejected. A pattern where each operator gets a "snapshot" of canonical data they can edit independently. Defeats the purpose of canonicalization and adds complexity (when does the snapshot refresh? what if canonical changes after the operator edited their copy?). Not chosen.

## Consequences

### Positive

- No structural duplication. One source of truth for "Sanjay Dugri Lodge has 24 rooms in 3 categories."
- Commercial and relational data is strictly isolated per operator, using the same RLS pattern as bookings, conversations, and quotes. No new isolation primitive.
- New operator onboarding is faster: they start with a populated catalog of vendors other operators have already added.
- Hakido accumulates a curated vendor graph across the Indian tourism industry — a real platform asset over time.
- Future features become possible: vendor discovery for operators ("find all lodges in Tadoba within 10km of the gate"), market-rate intelligence (with operator opt-in), cross-operator analytics, vendor verification badges.
- Photos and public structural data are shared assets, reducing storage and editorial cost.

### Negative

- More complex data model. Two-tier joins on every vendor read.
- Duplicate detection problem: when Operator B adds "Sanjay Dugri Resort" while Operator A already has "Sanjay Dugri Lodge," the system must surface a possible match. Fuzzy matching on name + region in v1; manual merge tooling for Hakido staff.
- Editorial responsibility: Hakido is now in the data-curation business. The canonical catalog has to be maintained, deduplicated, and (eventually) verified for accuracy.
- Operators may have opinions about how their preferred vendors are listed in the canonical catalog ("the lodge name is wrong," "those photos are old"). A correction-request flow is needed.
- Side-channel inference risk: a malicious operator cannot read another operator's commercial data via this design, but can observe canonical-layer changes ("Sanjay Dugri now lists 28 rooms instead of 24") and infer that some operator added information. Low-severity, accepted.
- Migration story for v0 / pre-v1 data: if any vendors were created in a flat model, they need migration into the two-tier structure. Mitigated by designing two-tier from day one.

### Neutral

- The `created_by_operator_id` column on `canonical_vendors` records provenance but does not grant ownership rights. Once a canonical record exists, all operators can link to it on equal terms. No "first operator wins" privilege.

## Privacy and isolation guarantees

To make the isolation explicit:

1. `canonical_vendors` rows are **shared, read-only-to-operators**. Operators see them but never see who else is using them, who created them (except via Hakido admin tools), or any commercial data attached by other operators.
2. `operator_vendors` rows are **strictly tenant-scoped via RLS**. Operator A cannot read Operator B's `operator_vendors` row, contracts, rate cards, internal notes, or feedback under any condition.
3. `vendor_contracts`, `vendor_rate_cards`, `vendor_feedback` inherit the operator scope of their parent `operator_vendors` row.
4. The application layer enforces these boundaries in addition to RLS, per ADR-0001's defense-in-depth principle.
5. Aggregated platform-level statistics (e.g., "this lodge has 12 operator relationships") are computed in the platform-admin code path and exposed only to Hakido staff, never to operators, in v1. Future operator-facing aggregates require explicit operator opt-in.

## Revisit triggers

We should reopen this decision if:

- Operators object to the canonical catalog model on competitive or privacy grounds severe enough to threaten adoption.
- The duplicate-detection problem becomes more expensive than the benefit (e.g., Hakido staff spending more than a few hours per week on catalog cleanup).
- A regulator-level concern emerges around aggregated vendor data (unlikely for tourism, but possible).
- An enterprise operator requires fully-private vendor data not linked to any canonical catalog. In that case, supporting both patterns side-by-side (private vendors flag) is more likely than abandoning canonicalization for everyone.

## References

- ADR-0001: Tenant data isolation strategy
- `docs/product/vendors.md` (forthcoming): full vendor management spec
- `docs/architecture/data-model.md` (forthcoming): complete schema with relationships
