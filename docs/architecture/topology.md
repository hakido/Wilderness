    # Deployment topology

How TravelHub is deployed across services and clouds, what each piece is responsible for, and where the seams are.

The architecture is deliberately split: **AWS for compute, AI, and async work; Supabase for application data and realtime; Stripe for payments; Meta for WhatsApp**. The cost is one cross-cloud line; the benefit is each part using the platform best suited to it.

## Deployment topology diagram

```
┌────────────────┐  ┌──────────────────┐  ┌─────────────────┐
│ Mobile (Expo)  │  │ Customer web     │  │ Operator web    │
│ Customer + agent│  │ Booking + agent  │  │ CRM dashboard   │
└────────┬───────┘  └────────┬─────────┘  └────────┬────────┘
         │                   │                     │
         │                   ▼                     ▼
         │          ┌────────────────────────────────────┐
         │          │ AWS account                        │
         │          │  ┌────────────────────────────┐    │
         │          │  │ Agent service (Fargate)    │    │
         │          │  │ Tool-calling loop, Python  │    │
         │          │  └─────┬─────────────┬────────┘    │
         │          │        ▼             ▼             │
         │          │ ┌──────────┐  ┌────────────┐      │
         │          │ │ Bedrock  │  │ Workers    │      │
         │          │ │ Claude   │  │ SQS+Lambda │      │
         │          │ └──────────┘  └─────┬──────┘      │
         │          └────────────────────┼──────────────┘
         │                               ▼
         │                      ┌────────────────┐
         │                      │ Stripe         │
         │                      │ Payments       │
         │                      └────────────────┘
         ▼
┌──────────────────────────────────────┐
│ Supabase                             │
│ ┌──────────────────────────────┐     │
│ │ Postgres (pgvector, PostGIS) │     │
│ └──────────────────────────────┘     │
│ ┌──────────────────────────────┐     │
│ │ Auth                         │     │
│ └──────────────────────────────┘     │
│ ┌──────────────────────────────┐     │
│ │ Realtime                     │     │
│ └──────────────────────────────┘     │
│ ┌──────────────────────────────┐     │
│ │ Storage                      │     │
│ └──────────────────────────────┘     │
└──────────────────────────────────────┘

Cross-cloud line: Agent service (AWS) ↔ Postgres (Supabase)
```

## The two halves

**AWS hosts the compute and AI side**

The agent service runs on AWS Fargate as a Python FastAPI application. It calls Bedrock for Claude, persists conversation state to Postgres (via the cross-cloud line), and queues async work to SQS. Workers run as Lambda functions consuming from SQS — they handle email sending, WhatsApp template messaging, PDF generation, payment reconciliation, scheduled reminders, and other non-blocking operations.

This side also includes:
- Bedrock with Claude (with prompt caching enabled and PrivateLink so traffic stays in-VPC)
- S3 for large media (tour photos at higher resolution, traveler-uploaded passports, vendor contract documents)
- CloudFront in front of the customer-facing web app and the customer mobile app's downloadable assets
- SES for transactional email
- EventBridge for cross-service events (booking confirmed, payment received, vendor cancelled, etc.)
- Secrets Manager for runtime credentials (Stripe keys, WhatsApp API tokens, Supabase service role key)

**Supabase hosts the data and realtime side**

Postgres holds the entire application schema. The operator dashboard talks to Supabase directly through the JS client: Auth for sign-in (with operator-configured providers), Postgres for CRUD (with RLS by `operator_id` enforcing tenancy), Realtime for live updates when bookings come in or guides update the manifest, Storage for tour photos and customer documents.

The customer web and mobile apps also talk to Supabase for read-heavy paths (browsing fixed departures, viewing trip details). Write paths and authenticated reads go through application API routes (Next.js Route Handlers) that add server-side validation, authorization beyond RLS, and orchestration (e.g., creating a booking involves writing to multiple tables transactionally).

## The cross-cloud line

The horizontal arrow from Agent service to Postgres is the architectural seam that matters most. The agent's tools (`searchTours`, `holdSeat`, `quoteBooking`, `createBooking`, `requestHumanHandoff`) hit Postgres over a pooled connection.

This is the one place where the two clouds couple, and it deserves careful design:

- **Connection pooling** — pgbouncer in transaction-pooling mode in front of Postgres. Fargate task count × connections per task should fit within Postgres's max_connections limit.
- **Credentials** — the agent service authenticates with a dedicated Postgres role (not the service role). Credentials live in AWS Secrets Manager, rotated quarterly.
- **Network** — the connection goes over the public internet, encrypted via TLS. Supabase doesn't currently support PrivateLink to AWS VPCs. If/when it does, switch.
- **Timeouts** — tool calls have aggressive timeouts (5-10 seconds) so a slow query doesn't stall the agent's conversation. Postgres queries have their own statement timeouts to enforce this server-side.
- **Retries** — transient failures (connection reset, single timeout) retry once with jitter. Persistent failures fail the tool call with a structured error the LLM can reason over.
- **RLS context** — every connection sets `app.operator_id` from the conversation's tenant context before issuing any query. The setting is reset between connections (per pgbouncer's transaction pooling).

## Per-component responsibilities

### Customer web app (`apps/web-customer`)

Next.js 15 with App Router, deployed on Vercel or self-hosted on AWS via OpenNext. White-labeled per operator via subdomain.

Responsibilities:
- Marketing pages, fixed-departure browse and detail
- Customer agent chat UI (React component embedded in Next page)
- Booking flow (traveler details, payment redirect to Stripe, confirmation)
- Customer account management
- Pre-trip information forms
- Post-trip review submission
- During-trip access to booking details, vouchers, vendor contacts

Talks to:
- Supabase: Auth, Postgres reads (browse), Realtime (agent conversation updates)
- Application API: writes (booking creation, profile updates, form submission)
- Agent service: chat WebSocket or SSE for streaming agent responses
- Stripe: redirect for checkout

### Customer mobile app (`apps/mobile`)

Expo + React Native, distributed via App Store and Play Store, OTA updates via EAS Update.

Responsibilities:
- Same as customer web: browse, chat with agent, book, manage account, complete pre-trip
- Optimized for during-trip use: offline-capable booking access, vouchers, push notifications, lifeline (one-tap WhatsApp to operator)
- Photo upload (passport scans, traveler photos for permits)

Talks to:
- Supabase: same as web
- Application API: same as web
- Agent service: same as web
- Push notifications: APNs (iOS) and FCM (Android) via Expo

### Operator web dashboard (`apps/web-operator`)

Next.js 15, the operator team's primary interface. Login required, authenticated via Supabase Auth.

Responsibilities:
- Inbox / per-customer timeline (the hero feature)
- Sales pipeline view
- Booking management
- Task management ("my today")
- Vendor directory and reservations
- Permit calendar
- Trip catalog and components configuration
- Agent configuration
- Communications templates
- Team and RBAC management
- Reports and analytics

Talks to:
- Supabase: Auth, Postgres reads/writes (with RLS), Realtime (live updates), Storage (uploads)
- Application API: complex operations (booking modifications, refund flows, manifest generation)
- Agent service: read-only access to conversation state and tool call audit trail

### Agent service (`services/agent`)

Python FastAPI on AWS Fargate. Stateless (state lives in Postgres); horizontal scaling via Fargate task count.

Responsibilities:
- Run the agent loop: receive customer message, fetch context, plan, call tools, generate response
- Maintain conversation state in Postgres (messages, tool calls, draft itinerary, quote)
- Stream responses to the chat UI (web/mobile) via Server-Sent Events
- Audit-log every tool call
- Trigger handoffs: change `current_handler`, notify operator team
- Handle reconnection: a customer who closes their browser and returns picks up the same conversation

Talks to:
- Bedrock: Claude API for LLM completions, with prompt caching
- Postgres (cross-cloud): all reads and writes for tools and state
- SQS: enqueue async work (send WhatsApp confirmation, generate PDF voucher, etc.)
- Stripe (indirectly via API service): payment link generation

### Application API (`services/api` or Next.js Route Handlers)

TypeScript. v1 starts as Next.js Route Handlers in `apps/web-operator` and `apps/web-customer`; if it grows beyond comfort, breaks out into a standalone Node service.

Responsibilities:
- Validation, authorization, orchestration for non-trivial writes
- Booking creation transactions (multiple tables, payment intent setup, task generation trigger)
- Refund processing
- Manifest generation
- Report queries
- Webhook receivers (Stripe, WhatsApp, payment providers)
- Operator configuration writes

Talks to:
- Postgres: reads and writes
- SQS: enqueue async work
- Stripe API: charges, refunds, webhook verification
- WhatsApp API: send templated messages

### Workers (`services/workers`)

AWS Lambda functions consuming from SQS queues. Mix of Python and TypeScript per task affinity.

Examples:
- `send_whatsapp_template_lambda` — pull from queue, send via WhatsApp Business API
- `send_email_lambda` — pull from queue, send via SES
- `generate_voucher_pdf_lambda` — pull from queue, generate PDF, upload to S3, link in booking
- `reconcile_payment_lambda` — receive Stripe webhook, update booking status, trigger task generation
- `process_review_lambda` — process post-trip review submission, aggregate to vendor performance
- `scheduled_reminder_lambda` — periodic, scan tasks for ones needing reminders

Each worker is small, single-purpose, idempotent. They share a common `lib/` for tenant-aware Postgres access.

### Background scheduler

Periodic jobs run via EventBridge Rules triggering Lambda:
- Hourly: scan for overdue tasks, fire reminders
- Daily: refresh vendor performance summaries, send digest emails to operators
- Daily: refresh FX rates from configured source
- Weekly: send operator activity reports
- Per-trip-date: generate manifest, send day-before briefings, etc.

## Data flow examples

### Customer plans a custom trip

1. Customer opens chat on `wildernessco.travelhub.hakido.co`
2. Customer web app opens SSE connection to agent service
3. Customer sends message → agent service receives → fetches conversation state from Postgres → calls Bedrock with assembled prompt → receives tool-use response → executes tools (Postgres reads) → may iterate → sends final response back over SSE
4. Conversation state persisted to Postgres after each turn
5. Operator dashboard subscribes to Realtime on the conversation; sees turns appear live
6. Eventually customer accepts quote → agent calls `generatePaymentLink` → Stripe Checkout URL returned
7. Customer pays → Stripe webhook to API service → booking confirmed → SQS task: generate task list → SQS task: send confirmation WhatsApp → Realtime push to operator dashboard

### Operator confirms a vendor

1. Operator team member opens booking detail page
2. Sees task: "Confirm Lodge Tigergarh booking"
3. WhatsApps the lodge from their phone via the operator's business number
4. Lodge replies "confirmed" via WhatsApp
5. Inbound webhook from WhatsApp → API service → message ingested into conversation timeline
6. Operator clicks "Mark confirmed" on the task → Postgres update → Realtime update across all team members viewing this booking
7. Reminder for this task cancelled
8. If "Send vouchers" task depended on "Confirm all vendors," and this was the last one, dependent task becomes active

### Customer fills pre-trip form

1. Customer receives reminder via WhatsApp (T-7 days before due) with magic link
2. Customer clicks link → web form on customer site
3. Customer fills form, uploads passport photo
4. Submission to API service → validation → write to Postgres → upload to S3
5. Tasks for passport collection marked complete
6. Operator's dashboard shows the booking's pre-trip checklist updating in real time

## Multi-tenancy at the topology layer

Subdomain routing identifies the operator at the edge:

- Customer web: `{operator}.travelhub.hakido.co` resolves at edge, Next.js middleware reads the subdomain and resolves to `operator_id` (cached lookup), sets context for the request
- Operator web: `{operator}.travelhub.hakido.co/admin` (or a separate operator subdomain pattern) — same resolution
- Mobile app: operator selection happens in-app at first launch; the app stores `operator_id` and uses it for all requests
- Agent service: receives `operator_id` in the request context from the calling client; validates against the conversation's stored `operator_id`

Once `operator_id` is resolved, every backend operation is tenant-scoped. RLS on Postgres enforces this at the data layer; application-level filtering enforces it at the API layer (defense in depth, per ADR-0001).

## Custom domains

In v1, all operators are at `{operator}.travelhub.hakido.co`.

In v2, operators can configure custom domains (e.g., `safaris.wildernessco.com`) via CNAME. Implementation requires:
- Custom domain registration in the operator's dashboard
- DNS verification (CNAME pointing to a Hakido-managed subdomain)
- TLS certificate provisioning (via ACM or Let's Encrypt)
- Edge routing rules to map custom domain to operator_id

This is mechanically straightforward but requires operational tooling and is deferred.

## Environment topology

Three environments:

**Production (`prod`)** — production workloads, paying operators, real customers
**Staging (`staging`)** — release candidates, integration testing, mirrors prod data structure with sanitized data
**Development (`dev`)** — engineer per-branch environments, feature work, ephemeral

Each environment has its own:
- AWS account
- Supabase project
- Stripe account (test mode for dev/staging, live for prod)
- WhatsApp Business numbers (sandbox for dev/staging if available; per-operator real numbers in prod)
- Domain (`travelhub-dev.hakido.co`, `travelhub-staging.hakido.co`, `travelhub.hakido.co`)

Secrets are environment-scoped; nothing crosses environment boundaries. Production data never copies to staging or dev (per Indian and international privacy expectations); seed data is synthetic.

## Observability

Logs and metrics flow:

- **Application logs** (structured JSON) → CloudWatch
- **Postgres logs** → Supabase logging dashboard
- **Edge / CDN logs** → CloudFront access logs to S3
- **Exception tracking** → Sentry (separate projects for web-customer, web-operator, mobile, agent service, workers)
- **Agent traces** → LangSmith or Langfuse (decide during agent build); per-conversation trace with tool calls, latencies, costs
- **Metrics** → CloudWatch metrics (Fargate, Lambda) + custom application metrics
- **Alerting** → CloudWatch alarms + Sentry alerts → PagerDuty (or equivalent)

Per-operator analytics queries run against Postgres directly (operator dashboards) or through the platform-admin code path (Hakido's own analytics).

## Disaster recovery

- **Postgres** — Supabase point-in-time recovery (default 7 days, can extend). Backup strategy: daily logical backups exported to S3 in addition to Supabase's native backups.
- **S3 storage** — versioning enabled on critical buckets; cross-region replication for production.
- **Application code** — git-based deployment, rollback via redeploying previous tag.
- **Configuration** — operator config in Postgres, included in backups. Hakido-level configuration in IaC (Terraform/CDK).
- **Secrets** — backed up via AWS Secrets Manager replication.

RTO targets:
- Catastrophic failure: 4 hours to restore service
- Single-component failure: < 30 minutes (auto-failover where possible)

RPO targets:
- Postgres: < 1 hour data loss in catastrophic scenarios (point-in-time recovery)
- S3: zero (versioning + cross-region)

## Cost shape

Rough monthly cost expectations at v1 scale (5-20 operators, low thousands of bookings):

- **Supabase Pro tier** — $25 base + compute add-ons + storage = $50-200/mo
- **AWS Fargate (agent service)** — 2-4 tasks running 24/7 = $50-150/mo
- **AWS Bedrock (Claude)** — variable; with prompt caching and conservative usage, $200-1000/mo
- **AWS Lambda + SQS + supporting services** — $20-100/mo
- **AWS S3 + CloudFront** — $20-50/mo
- **AWS SES** — $1-5/mo (email is cheap)
- **Stripe** — variable transaction fees, 2.9% + 30¢ per transaction (international); 2% for India
- **WhatsApp Business API** — variable per template message + per session; budget $50-200/mo
- **Sentry, LangSmith/Langfuse** — $50-150/mo

Total: roughly $500-2000/mo at v1 scale, growing primarily with Bedrock usage and transaction volume.

## Where this topology breaks

Honest about failure modes worth designing for:

- **Supabase outage** — operator dashboard goes down, customer browsing degraded (cache helps), agent service can't read/write (conversations queue or error). Mitigation: aggressive caching at app layer for read paths; agent gracefully degrades to "we're having a system issue, let me get you to our team."
- **Bedrock outage** — agent unable to respond. Mitigation: graceful degradation to handoff with "we're having a system issue" message; retry with backoff; consider fallback to direct Anthropic API as v1.5 if Bedrock outages become a pattern.
- **WhatsApp delivery failure** — falls back to email + operator follow-up call task.
- **Stripe outage** — bookings can be quoted but not paid; fallback to manual payment processing instructions, with operator follow-up.
- **Cross-cloud network partition** — agent service can't reach Postgres. This is the highest-impact single failure. Mitigation: aggressive timeouts so failed tool calls return quickly; agent escalates rather than hangs; operations alert.

The topology is designed to keep blast radius small. A failure in one component doesn't cascade because the seams are clean.
