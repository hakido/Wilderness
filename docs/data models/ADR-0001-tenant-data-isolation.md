# ADR-0001: Tenant data isolation strategy

- **Status**: Accepted
- **Date**: 2026-04-27
- **Deciders**: Hakido engineering
- **Project**: TravelHub

## Context

TravelHub is multi-tenant SaaS sold to tour operators. Each operator (tenant) runs their own white-labeled customer-facing site at `{operator}.travelhub.hakido.co` and their own operator dashboard. Tenants must not be able to see each other's data — bookings, customers, vendors, conversations, financials.

Three architectural patterns were considered for tenant data isolation:

1. **Shared database, shared schema** — one Postgres database, every table carries an `operator_id` column, Postgres row-level security (RLS) enforces isolation.
2. **Shared database, separate schemas** — one Postgres instance, each operator gets their own schema namespace.
3. **Separate database per operator** — each operator gets their own Postgres database (or instance).

The question came up at the start of the project, before the data model was written, so we have the option to pick any of these without migration cost.

## Decision

**We will use shared database, shared schema, with RLS keyed on `operator_id`.**

Specifics:

- Every tenant-scoped table has a non-null `operator_id uuid references operators(id)` column.
- Every tenant-scoped table has RLS enabled with a policy of the form:
  ```sql
  using (operator_id = current_setting('app.operator_id')::uuid)
  ```
- Every authenticated request sets `app.operator_id` from the JWT claims before any query runs.
- The application API layer also filters by `operator_id` from the session as defense in depth — never trusting URL params, body params, or client-supplied IDs.
- Automated integration tests assert that a request authenticated as operator A cannot read or write operator B's data, on every protected table.
- A small set of platform-admin tables (used by Hakido staff for cross-tenant operations like billing, analytics, and support) live outside the RLS regime and are accessed only via a separate, audited code path.

We will design the data access layer so a future migration to dedicated databases for specific enterprise customers is *possible* but not *required*. Practically:

- All data access goes through a `getDbForOperator(operatorId)` abstraction. Today this returns a single shared pool; tomorrow it can return a per-tenant pool for specific operators.
- Cross-operator queries are forbidden in product code paths and only allowed in a separate, clearly-marked platform-admin module.
- Operator-specific configuration (branding, agent system prompts, integration credentials, WhatsApp numbers) lives in tables that could be relocated with the operator's data if they ever moved to a dedicated database.

## Alternatives considered

### Separate database per operator

Rejected for v1. The benefits — hard isolation, per-tenant backup/restore, performance isolation, easy data residency, simple offboarding — are real but disproportionate to our current needs.

The costs are substantial and grow with operator count:

- Schema migrations must run against N databases. Tooling exists (Atlas, Sqitch) but it is never as cheap as a single migration. At 50+ operators this becomes meaningful engineering overhead.
- Operational surface scales with operator count: connection pools, monitoring, slow-query logs, backup configs, vacuum schedules.
- Cost: Supabase projects and RDS instances each have a base cost. Cheap consolidation defeats the isolation benefit.
- Cross-tenant queries — needed for Hakido-internal analytics and platform-level reporting — become application-layer aggregations across N databases.
- Realtime subscriptions, our reason for choosing Supabase, are scoped per-project. Per-tenant Supabase projects multiply realtime config and cost.
- Connection management at the agent service and API service must be tenant-aware, increasing memory footprint per service instance.

The operators we will serve in v1 are not regulated entities (no HIPAA, no PCI-Level-1, no banking compliance). Tour operators do not currently demand data residency or hard isolation as a contractual term. The risk of cross-tenant data leakage via RLS bug is real but mitigable with defense-in-depth (API-layer filtering, automated isolation tests, audit logging).

The industry pattern for modern multi-tenant SaaS — Notion, Linear, Vercel, Resend, Posthog and others — is shared DB with RLS. The companies that started with per-tenant databases and grew have generally regretted it; migrating to shared later is expensive.

### Shared database, separate schemas

Rejected. It captures most of the operational cost of separate DBs (per-schema migrations, per-schema configs) without the strongest benefit (true backup/restore isolation). Postgres RLS on a shared schema gives us logical isolation more cleanly with less ceremony.

## Consequences

### Positive

- Single migration command applies to all operators.
- Operational simplicity: one database to monitor, back up, tune.
- Cost-efficient: a single Supabase project covers all tenants until we need horizontal scale.
- Realtime, search, and analytics are straightforward — they query one source of truth.
- Onboarding a new operator is a row insert plus configuration, not infrastructure provisioning.
- Hakido-internal analytics (cross-tenant insights for the platform team) are SQL queries, not multi-database aggregations.

### Negative

- Tenant isolation depends on RLS correctness and on every API path correctly setting `app.operator_id`. A bug here can leak data across tenants. Mitigated by defense-in-depth and automated tests, but the failure mode is real and catastrophic.
- All operators share database performance. A heavy query from one operator can affect others. Mitigated by query timeouts, connection limits, and read replicas if needed.
- We cannot offer hard-isolation as a contractual term to enterprise customers without doing additional work.
- Backup and restore are all-or-nothing at the database level. Restoring one operator's data without affecting others requires careful row-level operations.
- Data residency is global to the database. Cannot offer per-operator regional storage without architectural change.

### Neutral

- The `getDbForOperator` abstraction adds a small amount of indirection for an option we may not exercise.

## Revisit triggers

We should reopen this decision if any of the following become true:

- An enterprise operator contractually requires hard data isolation or specific data residency.
- A regulated vertical (e.g. medical tourism with HIPAA implications) becomes a target market.
- We hit performance ceilings on a single Postgres instance that read replicas and partitioning cannot resolve.
- An operator's data volume is large enough that backup, restore, or vacuum on the shared database materially affects other operators.
- We discover an RLS bug in production that leaks tenant data, suggesting the defense-in-depth approach is insufficient.

In any of these cases, the likely move is *not* to migrate everyone to per-tenant databases, but to introduce a "dedicated tier" where specific operators get their own database while the rest remain on shared infrastructure.

## References

- Supabase row-level security: https://supabase.com/docs/guides/database/postgres/row-level-security
- Multi-tenancy patterns at scale (informal industry write-ups from Notion, Figma, Linear engineering blogs)
- ADR-0002 (forthcoming): API-layer authorization and defense-in-depth
- ADR-0003 (forthcoming): Cross-tenant analytics access pattern
