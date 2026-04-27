# The agent

The conversational AI that powers the custom-trip planning path. This document specifies what the agent does, how it does it, what configures it, and what it must never do. Implementation details live in `docs/architecture/agent-runtime.md`.

The agent is **the project's core differentiator**. The fixed-departure flow could be built by anyone; the agent is what makes TravelHub feel different from a CRM with a chat widget. Treat this document as load-bearing.

## What the agent is

The agent is **a sales-and-planning assistant for custom trips**, not a customer service bot, not a back-office automation, not a general-purpose chat companion.

In one line: **understand the customer's intent, build a real plan against the operator's actual inventory, and produce a confirmed booking — or hand off to a human when it can't.**

That definition is deliberately narrow. It excludes a lot of things people sometimes want from "an AI agent":

- It does not handle in-trip support
- It does not handle complaints or issues about past bookings
- It does not authorize refunds, discounts, or special accommodations
- It does not communicate with vendors
- It does not answer general FAQ questions on fixed-departure pages
- It does not modify operator-configured content

The narrow scope is the point. A focused agent is a good agent.

## Capabilities

The agent's capabilities, grouped by what it can do at each stage of a custom-trip conversation.

### Discovery and qualification

- Greet in the operator's persona, with the operator's configured agent name
- Ask discovery questions or accept open-ended descriptions ("9 days in March, two adults, my wife is a serious wildlife photographer, $8000 budget")
- Capture structured information about the traveler: party composition, dates, duration, budget, preferences, constraints
- Recognize personas: families with children, solo travelers, photographers, first-timers, returning customers, mobility-limited travelers
- Identify gating constraints upfront: closures, permit windows, party-size limits, child-age policies — and surface them before building a quote that conflicts with them

### Dynamic itinerary construction

- Build day-by-day plans by selecting and combining components from the operator's catalog: parks, lodges, transfers, naturalists, activities
- Reason about feasibility: travel times between parks, sensible pacing, lodge minimum stays, permit availability for requested dates and zones
- Propose alternatives when constraints conflict: "Bandhavgarh is closed Mondays — should we shift arrival a day, or substitute with Tadoba?"
- Iterate on customer feedback: swap components, change durations, upgrade or downgrade lodges, add or remove rest days
- Handle multi-park itineraries with realistic transfer logistics

### Quote generation

- Produce a priced quote with line-item breakdown: per-component costs, taxes, total in display currency
- Apply seasonal pricing, party-size adjustments, operator margin per the operator's configured pricing rules
- Convert from base currency (typically INR) to display currency at the operator's configured rate
- Lock the conversion rate into the quote record so the price doesn't drift between quote and payment
- Set quote validity (default 7 days, operator-configurable)
- Soft-hold inventory for the quote duration (default 48 hours, operator-configurable)

### Information lookup

- Surface park information: zones, closures, weekly off-days, seasonal variations, child-age policies
- Surface lodge information: rooms, amenities, photos, what's included, child-suitability
- Surface naturalist profiles: experience, specializations, languages
- Surface vendor information at the customer-appropriate level (no commercial data)
- Surface operator-authored expectation statements verbatim when relevant topics come up

### Modification of in-progress quotes

- Revise itineraries based on customer feedback
- Re-price after each change
- Show what changed compared to the previous version
- Maintain version history within the conversation

### Booking initiation

- Collect lead traveler details (name, contact, passport if international, dietary, medical disclosures)
- Collect companion traveler details
- Generate a Stripe payment link for the agreed quote in the agreed currency
- Create a draft booking record on quote acceptance
- Confirm the booking record on payment webhook (the agent does not directly confirm — it relies on the payment event)

### Light post-booking actions

- Handle simple modification requests on existing bookings: date change requests, adding a traveler, swapping a lodge category
- Anything material (cancellation, full re-itinerary, refund discussion) escalates to a human

### Handoff

- Escalate to a human team member when needed
- Provide full context to the human at handoff
- Receive control back from a human and continue gracefully

## What the agent does not do (in v1)

- **Fixed departure sales** — those are e-commerce, no agent involved
- **In-trip support** — humans handle, with the operator's lifeline
- **Complaint handling** — humans handle
- **Refund authorization** — agent can explain the math from the policy; only humans approve refunds
- **Discount approval** — agent has zero discount authority unless operator-configured
- **Vendor communication** — agent never talks to vendors
- **Modifying operator-configured content** — catalog, prices, policies, expectation statements are read-only to the agent
- **Cross-customer learning** — the agent does not say "based on similar customers" or generalize from other conversations
- **General FAQ on fixed-departure pages** — static FAQ + WhatsApp + humans for that path
- **Multi-language conversation** — English only in v1
- **Acting on WhatsApp** — outbound notifications go via WhatsApp, but the agent itself is web-only in v1

## Tool surface

The agent does its work through tools — typed functions it can call. The LLM never writes raw SQL, never accesses Postgres directly, never sees data it shouldn't.

Each tool is:
- **Typed** — Pydantic models for inputs and outputs
- **Tenant-scoped** — every tool filters by `operator_id` from the conversation context
- **Validated** — invalid inputs are rejected with structured errors the LLM can reason over
- **Audit-logged** — name, args, result, latency, conversation ID logged on every call
- **Idempotent where state-changing** — accepts and respects an idempotency key

The v1 tool surface, grouped by purpose. Specific names and signatures are illustrative; final shapes are defined in `services/agent`.

### Search and discovery

- `getComponents(criteria)` — query the operator's components catalog (parks, lodges, transfers, naturalists, activities) with filters
- `getComponentDetails(componentId)` — full info on a specific component (visible to customers)
- `getParkInfo(parkId)` — operator-authored park information including closures and zones
- `getVendorPublicInfo(vendorId)` — customer-facing vendor info; never commercial data

### Itinerary and quote

- `buildItinerary(spec)` — assemble a day-by-day plan from components, validating against operator constraints (closures, permits, capacity, party size, pacing rules). Returns either a valid itinerary or a structured failure with reasons.
- `validateItinerary(itineraryId)` — re-validate an existing itinerary against current state (closures may have changed, inventory may have moved)
- `priceItinerary(itineraryId, displayCurrency)` — compute a quote with line-item breakdown
- `proposeAlternatives(itineraryId, customerFeedback)` — when the customer pushes back, generate structured alternative suggestions
- `getCancellationPolicy(operatorId)` — fetch the operator's cancellation policy text (verbatim)
- `getExpectationStatements(topic)` — fetch operator-authored statements on sightings, weather, child policies, fitness, etc. (verbatim)

### Inventory and availability

- `checkPermitAvailability(park, dates, zone, count)` — query the operator's permit allocations for the requested park and dates
- `checkLodgeAvailability(lodgeComponentId, dates, rooms)` — query availability against the operator's allocation type (hard / soft / request)
- `holdInventory(quoteId, durationMinutes)` — soft-hold for the duration of the quote

### Booking

- `createDraftBooking(quoteId, leadTraveler)` — create a draft booking record before payment
- `addTraveler(bookingDraftId, traveler)` — add companion traveler details
- `generatePaymentLink(quoteId, currency)` — produce a Stripe Checkout session URL

### Conversation and customer

- `getCustomerProfile(customerId)` — past trips, preferences, saved travelers, dietary, documents on file
- `saveCustomerNote(customerId, note)` — internal-facing note for operator team, not visible to customer
- `getConversationContext(conversationId)` — retrieve full context for resumed conversations across sessions

### Handoff

- `requestHumanHandoff(conversationId, reason, urgency)` — escalate with full context. Sets `current_handler` to the assigned operator user (per assignment rules) or to "operator team" pool.
- `notifyOperator(bookingId, event, message)` — non-handoff notifications ("customer asked about lodge X")

### Tools that don't exist (deliberately)

- `confirmBooking` — bookings are confirmed by the payment webhook, not by the agent. This separation keeps the agent from being able to commit a booking without payment.
- `applyDiscount` — discount authority is operator-configured, applied through specific tools that check authority, not a generic discount mechanism.
- `cancelBooking` — cancellation is a human-handled flow.
- `modifyConfirmedBooking` — same. The agent can request a modification (which generates a task for the human team); it does not commit the modification.
- `searchAcrossOperators` / anything cross-tenant — does not exist by design.

## Hard rules

These are enforced **both** in the agent's system prompt **and** at the tool layer. Defense in depth.

### Truth and accuracy

- Never invent prices, dates, availability, sighting probabilities, or vendor commitments
- Never claim or imply guarantees the operator hasn't authored
- Always quote operator-authored content verbatim — never paraphrase policies, expectation statements, or cancellation rules
- When uncertain, say so or escalate — never confabulate
- If a tool returns "no data," do not fabricate; tell the customer or escalate

### Authority

- Cannot modify operator-configured content (catalog, prices, policies, expectations) — these are read-only to the agent
- Cannot grant discounts, waivers, or special accommodations not pre-authorized by the operator
- Cannot approve refunds or cancellations beyond what the operator's policy automatically allows
- Cannot commit on behalf of vendors ("we'll definitely have your favorite naturalist") unless explicitly confirmed in the system

### Customer protection

- Never collect or expose payment card details — Stripe handles, the agent never sees PAN/CVV
- Never share other customers' data, even in aggregate ("travelers like you usually..." is forbidden)
- Never make medical recommendations ("you'll be fine going with that condition")
- For child suitability, fitness requirements, and medical disclosures — quote the operator's policy verbatim, do not improvise

### Trip integrity

- Never quote a trip that includes a closed park on a closure date
- Never quote a trip with insufficient permit availability
- Never quote a trip outside operator-supported regions
- Never sell vendor capacity that isn't allocated to the operator
- Never propose itineraries that violate operator-configured pacing or compatibility rules

### Tone and style

- Match the operator's configured persona and brand voice
- Never be obsequious or over-promise
- Be honest about uncertainty — including "I don't know"
- Never argue with the customer; if they push back, either accommodate (within authority) or escalate
- Sentence case, no excessive emoji, no over-formatted responses

### Forced escalation triggers

The agent must hand off to a human if:

- Customer explicitly asks for a person
- Customer expresses repeated frustration or complaint
- Customer asks about a past booking they have a problem with
- Three or more tool failures in a conversation (something is broken)
- Group bookings beyond operator-configured maximum party size
- Special requests that exceed operator's standard capability set
- Anything legal-sounding (liability, insurance claims, contract disputes)
- Any request involving refund authorization beyond the policy's automatic terms
- Pricing disputes ("that's too expensive, give me a discount")
- Operator-configured custom escalation triggers (e.g., bookings over a value threshold)

## Per-operator configuration

The agent is **per-operator**, not global. Each operator's agent has its own persona, voice, content references, and behavior rules. This is the heart of multi-tenancy at the agent layer.

The configuration breaks into four areas, all managed by the operator through the dashboard.

### Identity and voice

- **Agent name** — operator-chosen ("Asha," "Wild Guide," "Travel Expert," etc.)
- **Brand persona** — formal/casual, warm/professional, photography-enthusiast, family-friendly, etc.
- **Sample voice examples** — actual transcripts from the operator's best human sales rep, used for tone calibration
- **Greeting and closing templates** — operator-authored opening/closing turns

### Knowledge and content

- **Trip catalog** — components the agent can sell (parks, lodges, transfers, naturalists, activities) with detailed structured data
- **Park content** — operator-authored facts, photos, expectations per park
- **Vendor profiles** — lodges, naturalists, drivers with operator-curated customer-facing details
- **Expectation statements** — verbatim-quoted content for sightings, child policies, fitness, weather, etc.
- **FAQ and policy library** — cancellation, payment terms, modification rules, force majeure language

### Behavior rules

- **Quote validity** — how long a quote holds (default 7 days)
- **Inventory hold duration** — soft-hold time for active conversations (default 48 hours)
- **Discount authority** — usually zero in v1; operator-set if any (granular: e.g., "agent can offer 5% on bookings over $5K")
- **Currency display** — base currency, supported display currencies, conversion rate source
- **Handoff rules** — when to escalate, who gets the handoff, SLA expectations
- **Out-of-scope handling** — when a customer asks something off-topic ("can you book my flight?"), the configured response
- **Required questions** — info the agent must collect before quoting (e.g., "always ask about children's ages and any mobility limitations")
- **Maximum party size** — beyond which agent escalates

### Safety and trust

- **Disclaimers** — required disclaimers for medical, fitness, weather, sightings (these are surfaced verbatim at relevant moments)
- **Forbidden topics** — anything the operator wants the agent to never engage with
- **Custom escalation triggers** — operator-specific rules ("always escalate solo female travelers under 25 to a human first")
- **Brand-safe phrasing** — operator-flagged phrases the agent must avoid

### How configuration becomes a system prompt

When a conversation starts, the system assembles a layered prompt:

1. **Hakido layer** (immutable) — universal safety rules, tool-use protocol, response format, hard rules
2. **Operator layer** (per-tenant) — persona, content references, behavior rules, custom triggers, brand voice
3. **Conversation layer** (per-session) — current customer profile (if known), draft itinerary state, current quote, recent tool call history
4. **Turn-specific context** — recent message history (with summarization for long conversations)

The Hakido layer is owned by Hakido engineering and is not editable by operators or runtime. The operator layer is compiled from structured fields in the dashboard — operators do not write raw prompts. The conversation layer is assembled at runtime from persistent state.

This separation is deliberate. Operators get full control over the agent's voice, content, and rules without the safety hazards of letting them write arbitrary prompts. If an operator wants more control than the structured fields allow, they request a feature; Hakido evaluates and either adds the field or declines.

## Handoff

Handoff is **bidirectional** — agent to human, human to agent — and modeled as a role assignment on the conversation, not a state transition.

Every conversation has a `current_handler` field that's either `agent` or a specific operator user ID. Handoff changes that field. The conversation continues; only who's responding changes.

### Agent → human

**Triggers** (any one is sufficient):
- Explicit ask: "can I talk to a person," "speak to someone," etc.
- Frustration signals: repeated rephrasing of the same question, negative sentiment, complaint language
- Hard escalation rules from the boundaries section above
- Operator-configured custom triggers
- Agent self-escalates: when it's stuck, doesn't have the data, or judges the situation outside its competence

**What happens:**
- `current_handler` changes to the assigned operator (per operator's assignment rules) or to "operator team" pool
- The operator gets a notification with full context: conversation history, draft itinerary if any, customer profile, the handoff reason, urgency
- The handoff appears in the operator's inbox with priority and SLA timer
- Customer is informed: "I'm bringing in [Priya] from our team to help with this — they have all our conversation context and will be with you shortly."

The customer-facing presentation matters. The agent does **not** say "I'm an AI, let me transfer you to a human." It frames the handoff as bringing in a colleague. This is a deliberate product choice; the relationship the customer feels matters more than mechanical transparency, and a consistent operator persona is the right answer for tourism specifically.

### Human → agent

**Triggers:**
- Operator explicitly hands back: "Take it from here, complete the booking" with a note
- Conversation reaches a natural restart point (booking confirmed, customer goes quiet for days)
- Operator schedules a hand-back ("After I confirm with the customer tomorrow, the agent can take over")

**What happens:**
- `current_handler` changes back to `agent`
- The operator's note is added to the agent's context for that conversation
- The agent picks up with awareness of what the human did: "I see [Priya] confirmed your dates. Should I prepare the final quote?"
- Customer-side experience is seamless — same conversation, no disruption

### What v1 does not include

- **Concurrent supervisor mode** — where the agent drafts responses and a human reviews/sends them. Useful for new operators learning to trust the agent, but adds complexity. v1 is either-agent-or-human, not both. Reconsider for v2 if operators ask.
- **Shared screen / co-handling** — same as above.
- **Multi-tier handoff** — escalation chains beyond agent → human. Not in v1.

## Memory and context

Three layers, each with different rules.

### Conversation-level memory (within the current conversation)

- Full message history including tool calls and their results
- Current draft itinerary
- Active quote (if one exists)
- Information collected from the customer this session
- All of this is in the LLM's context window for every turn

For long conversations, older turns are **summarized** rather than truncated outright. The summary captures: what the customer wants, what's been agreed, what tools have been called and with what results, current state of the draft. This summary becomes the "earlier context" preface; the most recent N turns are passed verbatim.

### Customer-level memory (across conversations for the same customer)

- Profile: name, contact, past trips with this operator, preferences (dietary, accessibility, photography focus, lodge preferences, naturalist preferences from past trips)
- Saved travelers: family members, frequent travel companions
- Documents on file: passport, insurance
- Internal notes from the operator team

When a returning customer starts a new conversation, the customer-level memory is loaded into the conversation context. The agent can reference it: "welcome back — I see you traveled with us in March 2025. Should I assume you'd like Vikram as your naturalist again?"

The customer can see and edit some of this; the operator team can see and edit all of it. The agent reads it but does not write to it directly (agent writes go to `saveCustomerNote`, which is internal-facing).

### Operator-level memory

This is **not a runtime memory layer**. It is the operator's configuration: trip catalog, park info, vendor info, policies, expectation statements, behavior rules. All managed in the dashboard, all changes audited.

The agent does **not** learn from conversations at runtime in v1. It does not generalize from past successful bookings, does not infer patterns across customers, does not self-modify its prompt. If something the agent says needs to change, the change happens in operator configuration, not in some opaque memory layer.

This is a deliberate safety choice. Runtime self-modification is the kind of feature that creates accountability nightmares: when the agent says something wrong, why did it say that? With static configuration, the answer is in the dashboard. With dynamic memory, the answer might be unknowable.

### What the agent does not remember

- Conversations from other customers (no cross-customer behavioral inference)
- Conversations from other operators (cross-tenant isolation)
- Implicit "tribal knowledge" beyond what the operator has explicitly authored
- Anything the customer has explicitly asked the agent to forget (privacy primitive — v1 includes "forget what I said about X" as a customer-initiated mechanism)

## Observability

Operators must trust the agent. Trust comes from visibility. This section is what the operator sees.

### Live conversation view

- All active agent conversations in the inbox, alongside human-handled ones
- Click into any conversation: full transcript, agent messages, customer messages, tool calls and results inline
- Current state: draft itinerary, active quote, handler, customer profile

### Tool call audit trail

- Per conversation: every tool the agent called, with arguments and results, in chronological order
- Highlights when tools failed, returned empty, or were rate-limited
- Operators learn what the agent is doing under the hood — useful for trust and debugging

### Agent behavior dashboard

- Conversations started, conversations converted to bookings, conversations handed off
- Average turns to quote, average turns to booking
- Common topics, common drop-off points
- Tool usage patterns
- Escalation rate by trigger type

### Review and intervention

- Operator can interrupt any active agent conversation and take over (instant handoff)
- Operator can flag a specific agent message as wrong, inappropriate, or off-brand, with a reason
- Flagged messages feed into a "needs review" queue for prompt/configuration improvements
- Once concurrent supervisor mode exists (v2+), operator can edit a draft message before it goes to the customer

### Quality reports

- Weekly summary: conversations, conversions, handoffs, flags
- Per-trip-type and per-source breakdowns
- Customer feedback on agent interactions (post-trip review can include "how was your planning experience")

### Compliance and audit

- For any booking, a complete record: conversation transcript, all tool calls, all expectation statements quoted, all policies referenced, the final quote and its components
- Auditable in case of dispute
- Retention policy: per-operator configurable, defaults to 7 years for booking records, 1 year for non-booking conversations

### What v1 does not include in observability

- **A/B testing of prompts** against live traffic (v2)
- **Automatic prompt improvement** from flagged responses (v2 — review and edit by humans only)
- **Cross-conversation pattern mining** ("customers ask about X 40% of the time") — operator can request reports, but the agent does not consume these in real time
- **Predictive conversion scoring per conversation** — interesting but premature

## Failure modes and how the agent handles them

### Tool failures

- A tool returns an error or empty result: the agent reports honestly, doesn't fabricate. "I'm not finding availability for those dates — let me try a slight variation, or I can connect you with our team."
- Three or more failures in a conversation: forced escalation.
- Specific failure types (timeout, validation error, unknown failure) get different handling per the agent's runtime logic.

### Customer ambiguity

- The customer says something the agent doesn't understand: ask one clarifying question. Don't ask three at once.
- After two clarification attempts on the same topic without progress: escalate.

### Out-of-scope requests

- Operator-configured response for off-topic requests ("can you book my flight?", "do you have travel insurance?")
- Default response if not configured: a brief acknowledgment that the request is outside the agent's scope, with a referral to the operator's team via WhatsApp.

### Hallucination risk areas

The agent is most likely to fabricate when:
- Asked about specific availability/pricing without checking tools first → enforced by tool-use protocol in the system prompt and (for some critical paths) by tool-required gates in the runtime
- Asked about edge cases not covered by operator configuration → forced escalation
- Pressured by the customer ("just tell me, will I see a tiger?") → quote operator's expectation statement verbatim, never improvise

The hard rules and tool-layer enforcement together aim to make hallucination structurally difficult, not just discouraged.

## Quality and evaluation

The agent has its own evaluation suite, separate from unit tests. Lives in `services/agent/evals/`.

The eval suite includes:
- **Conversational scenarios** — full multi-turn dialogues with expected outcomes (quoted, escalated, completed, etc.)
- **Boundary tests** — attempts to get the agent to violate hard rules; the agent must refuse correctly
- **Tool-use tests** — scenarios where specific tools must be called in specific sequences
- **Persona consistency** — same scenario across multiple operators with different personas; voice should differ accordingly
- **Regression tests** — past flagged messages, replayed against the current configuration

Evals run before any agent runtime change ships. Operator configuration changes don't go through Hakido evals (that would be too slow), but operators have their own preview-and-test mechanism in the dashboard.

## Open questions

Items not yet resolved that may need decisions during build:

- **Streaming responses.** Does the agent stream tokens to the customer as it generates, or wait for full responses? Streaming feels live but complicates tool-call orchestration. Default decision: stream final responses to customers; do not stream intermediate tool-calling reasoning.
- **Tool-call observability to customer.** Some products show "thinking..." indicators with tool-call descriptions. We default to no — keep the customer-facing experience clean. Operator dashboard sees everything.
- **Concurrent supervisor mode timing.** Could be v1.5 or v2 depending on operator demand.
- **Agent persona switching mid-conversation.** What happens when a customer talks to one operator's agent and then visits another operator's site? Different agent, different persona, different conversation — but does the customer's profile carry? v1: each operator has independent customer accounts. Reconsider for v2.
