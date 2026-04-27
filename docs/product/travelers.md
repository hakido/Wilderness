# Travelers

The end-customer side of TravelHub. Travelers are the people who book and go on trips. This document covers what they can do, how they actually behave, and the design implications that flow from both.

The **three entry paths** to a booking are the structural shape of the customer-facing product:

1. **Browse fixed departures** — pure e-commerce flow on the operator's site. List, filter, view, book, pay.
2. **Plan a custom trip** — conversational planning with the AI agent. Discovery, itinerary construction, quote, book, pay.
3. **FAQ + WhatsApp** — static FAQ answers common questions on fixed-departure pages; anything else routes to humans via WhatsApp.

Post-booking, all three converge: same booking record, same task system, same pre-trip flow, same post-trip review.

## Capabilities

Grouped by journey stage. Same capabilities apply regardless of how the booking was made unless noted.

### Discovery and planning

- Browse fixed departures with filtering by destination/park, dates, duration, price, activity type, difficulty
- View detailed trip pages: day-by-day itinerary, included/excluded, lodge details, naturalist info, sample photos, expectation statements
- Read static FAQ on fixed-departure pages and operator's general site
- Use WhatsApp to ask questions of the operator's human team
- Chat with the AI agent to plan a custom trip from scratch (the dynamic itinerary path)
- Save trips and itineraries to come back to later (account or magic-link based)
- Compare trip options side-by-side
- Share an itinerary or quote with a travel companion (link-based, no account required for the recipient)

### Booking and payment

- Select a fixed departure or accept a custom-trip quote
- Add traveler details for the lead and companions: names, ages (especially for children), passport (international), dietary, fitness disclosures, special requests
- See a clear price breakdown: components, taxes (GST for domestic INR), discounts (if any), total in display currency
- See the operator's cancellation policy and expectation statements before paying
- Accept terms, waiver, and any operator-required disclosures
- Pay deposit or full amount via Stripe in supported display currency
- Receive confirmation immediately after payment with booking reference

### Pre-trip

- Submit pre-trip information via a guided form: passport scans, emergency contacts, medical disclosures, dietary, gear sizes, arrival flight details
- Receive operator follow-up call if pre-trip form is incomplete by T-2 days
- Receive packing list, briefing document, weather updates, and other operator-prepared materials
- Modify the booking: date change request, swap traveler, add traveler (subject to operator approval), upgrade lodge, add a day
- Cancel and see refund computed against the operator's cancellation policy
- Provide proof of insurance if the operator requires it
- Pay balance amount when due (if booking was deposit-based)

### During trip

- Access daily plan, vendor contacts, vouchers, and permits in one place (mobile-optimized)
- Receive day-of-trip briefings via WhatsApp ("good morning, today's safari is at 5:30, gypsy 4")
- Reach the operator's lifeline (WhatsApp number, in-app SOS) for any issue
- Report issues mid-trip (room not ready, driver late, sighting disappointment) — captured into operator's issue queue
- Receive proactive updates from the operator (route change, weather, sighting alerts)

### Post-trip

- Receive review request 1 day after trip completion
- Submit ratings on operator, lodges, naturalists, drivers, vehicles, food
- Submit free-text feedback and tag specific incidents
- Download GST invoice (domestic), receipts, certificates, sighting log if applicable
- Rebook for a future trip; refer friends with operator-configurable benefits
- Access trip photos if shared by the operator

### Account

- Create an account with email + OTP or social login (operator can choose providers)
- Manage profile: contact info, preferences, dietary, accessibility
- Manage saved travelers (family members, frequent companions) so they don't need to be re-entered each booking
- View past trips, current bookings, draft itineraries
- View documents on file (passport, insurance) for reuse
- Receive multi-traveler magic links so companions can fill their own pre-trip forms without needing accounts

## Behavior categories

Eight categories of how travelers actually behave, with architectural implications.

### 1. Indecision and slow-burn planning

Adventure-tour travelers research for weeks, sometimes months. They open a chat, ask three questions, disappear for nine days, return with a friend and a different date.

**Implications:**
- Conversations must be **persistent and resumable** across sessions and devices, not "chat history" as a nice-to-have but as the primary state.
- The agent recalls preferences and context without re-asking ("Last time you mentioned April, photography focus, two adults — still on?").
- A single traveler can have multiple **draft itineraries** in parallel ("the cheaper one" vs "the dream one") and may want to compare them.
- The operator dashboard surfaces *active draft conversations*, not just confirmed bookings — these are the pipeline.

### 2. Group-think and the "let me check with..." pattern

Almost no one books a safari alone. A decision-maker is chatting with the agent, a spouse or friend has veto power, sometimes a parent is paying.

**Implications:**
- Mid-conversation freezes ("let me check with my husband and get back to you") happen often. The agent should gracefully hold the quote, set a soft expiry, send a follow-up reminder, and not pressure.
- **Shareable itinerary links** are common: "send me a link I can show my wife." The link displays the itinerary, day-by-day breakdown, and price, optionally letting the second person ask the agent their own questions in the same thread.
- Quote-then-pay split: the planner is often not the payer. The payment link must work for someone who never talked to the agent.

### 3. Expectation statements (formerly "will I see a tiger")

Travelers ask anxious questions about the things they care about most: tiger sightings, weather, child suitability, fitness requirements, accessibility. These questions have right and wrong answers, and getting them wrong is liability-adjacent.

**Three sub-behaviors:**
- **Genuine information need** — "what are the chances?" → honest expectation-setting using the operator's authored data
- **Emotional reassurance need** — "I'm spending a lot of money, please tell me it'll be worth it" → empathy and honesty without overpromising
- **Negotiation pretext** — "can you guarantee I'll see a tiger? Refund if I don't?" → firm, friendly, never agree to a guarantee

**Implications:**
- The agent has a **policy/disclaimer layer**: operator-authored expectation statements that the agent **quotes verbatim**, never paraphrases.
- This is a domain-agnostic primitive: the same mechanism powers tiger-sighting expectations for safari operators, photography-restriction notices for heritage operators, altitude-sickness warnings for trekking operators, etc.
- Operators author these statements during onboarding and update them seasonally. They cover sightings, weather, child policies, fitness, accessibility, refund-on-no-sighting language, and anything else where misstatement is risky.

### 4. Last-minute changes and force majeure

Trips get disrupted. Monsoon arrives early, a park closes a zone, a flight cancels, a traveler gets sick. Cancellations and reschedules are a meaningful percentage of bookings.

**Implications:**
- Refund/credit logic must be **explainable by the agent** in plain language: "you're 18 days out, your operator's policy gives you 50% credit toward a future trip, here's the breakdown." The agent quotes the operator's policy text verbatim alongside the calculation.
- Operators may need to **broadcast changes** to multiple bookings ("Tadoba's Moharli zone is closed for 3 days, here are the affected travelers, draft an apology and rebook offer"). This is operator-side tooling but the customer-facing impact is direct.
- Weather and closure-driven cancellations are **distinct from voluntary cancellation**: the system has a force-majeure flag on cancellations that bypasses normal penalty calculation per operator-configured policy.

### 5. Loss of patience → human handoff

Travelers reach a limit. Sometimes they want a person from the start. Sometimes the agent struggles and they get frustrated. Sometimes the situation is genuinely outside the agent's competence (complaint, complex group, special circumstance).

**Implications:**
- Handoff triggers: explicit ask, frustration signals (repeated rephrasing, negative sentiment, three failed clarifications), business-rule triggers (high-value bookings, complaints about past bookings, complex itineraries flagged by operator config).
- Handoff is **bidirectional**: a human can resolve and hand back to the agent ("customer is good with this, take over and finalize the booking"), or take the conversation human-permanent from that point.
- Handoff requires an operator inbox with assignment, SLA, presence/availability — covered in `operators.md`.
- Customer-facing presentation: handoff is framed as bringing in a colleague, not switching from AI to human. This is a deliberate product choice; see `agent.md` section on persona.

### 6. Anxiety and trust signals

Adventure trips are expensive trips to remote places with strangers. Travelers are anxious even when they don't say so.

**Common patterns:**
- Repeatedly asking about safety, especially solo female travelers and families with young children
- Asking for vendor names ("which lodge?") to Google them independently
- Wanting to see reviews, photos, naturalist credentials
- Asking what happens if something goes wrong on the ground

**Implications:**
- The agent should be **generous** with these answers. Withholding feels evasive. This implies operators populate rich vendor profiles (lodges, naturalists, drivers) with photos, credentials, sample reviews, and the agent has tools to surface them.
- Trust comes from specifics, not reassurance. "You'll be staying at Tigergarh Lodge — here are photos, here's the naturalist Vikram who'll be with you, he's worked here for 12 years and specializes in big-cat photography" is worth more than "we'll take great care of you."

### 7. Information overload and decision fatigue

The agent's biggest UX failure mode is wall-of-text answers. Travelers want progressive disclosure.

**Implications:**
- One digestible answer first, then "tell me more if you want."
- Comparison is a recurring need (Bandhavgarh vs Kanha, this lodge vs that lodge, gypsy vs canter). The agent produces **structured comparisons** — small tables, side-by-sides — not paragraphs.
- Visual is better than verbal where possible: maps, photos, day-by-day cards beat narrative.
- The chat is a **rich message stream**, not just text. It includes itinerary cards, quote tables, lodge cards, comparison views, payment buttons. The agent emits structured outputs the UI renders.
- The conversation store has to be richer than `{role, text}` — every message carries a content type and structured payload.

### 8. Domestic vs international travelers

Indian operators serve both. Behavior diverges sharply.

| Aspect | International | Domestic |
|---|---|---|
| Lead time | Months | Weeks (often last-minute) |
| Trip style | Premium packaged | Mix; often shorter, family-driven |
| Currency | USD/EUR/GBP/AUD | INR (with GST) |
| Language | English | Hindi/English mix (English-only in v1) |
| Channel | Email + web | WhatsApp-first |
| Anxiety level | Higher | Lower (familiarity with the country) |
| Booking pattern | Long single bookings, often with flights/visa adjacent | Often weekend/holiday trips, more rebooking |

**Implications for v1:**
- **Currency** is per-traveler at quote and payment time. Operator sets base currency (typically INR); display currencies are operator-supported.
- **Language** is English-only in v1 but the architecture supports adding Hindi later (per-message language detection, per-template translations).
- **GST** applies to domestic INR transactions. International transactions are zero-rated under export-of-services rules.
- **WhatsApp-first** posture for domestic travelers is real. Outbound notifications and inbound webhook ingestion are both v1.

## The three entry paths in detail

### Path 1: Browse fixed departures (no agent)

Pure e-commerce. The operator publishes fixed departures — packaged trips with set dates, prices, inclusions. Each appears on a listing page with filtering. Each has a detail page with day-by-day itinerary, lodge details, what's included, photos, expectation statements, cancellation policy.

The traveler:
1. Filters and selects a departure
2. Clicks "Book this departure"
3. Fills in lead traveler details and companions
4. Sees price breakdown, accepts terms and policies
5. Pays via Stripe
6. Receives confirmation immediately

If they have a question, the page shows static FAQ first; if the FAQ doesn't answer it, they reach the operator via WhatsApp. The agent is **not involved** in this path.

If a traveler wants to modify the fixed departure (different dates, swap a lodge, extend by 2 days), the flow routes them to either: a contact-the-operator path (human handles), or "Want something custom? Talk to our planner →" which routes them to the agent path.

### Path 2: Plan a custom trip (with the agent)

The traveler clicks "Plan a custom trip" on the operator's homepage or any other entry point. They land in a chat interface with the agent.

The agent:
1. Greets in the operator's persona, with the operator's name
2. Asks discovery questions (or lets the traveler describe what they want)
3. Builds a draft itinerary using the operator's components catalog
4. Iterates with the traveler ("what about Tadoba instead of Kanha?")
5. Produces a quote
6. Surfaces relevant expectation statements verbatim
7. Takes the traveler to payment when ready
8. Hands off to a human if needed at any point

Throughout, the agent persists conversation state, can be resumed from any device, and produces a shareable link so a companion can review.

Detailed agent design in `agent.md`.

### Path 3: FAQ + WhatsApp

Both the fixed-departure detail pages and the operator's general site include a static FAQ. Common questions: park information, lodge information, sample itineraries, what to bring, weather, payment terms, cancellation policy, child policies, accessibility.

For anything not in the FAQ, the customer reaches the operator via WhatsApp at the operator's published business number. Inbound messages are ingested into the operator dashboard via webhook so any team member can see and respond. The agent is **not in the loop** for this path in v1.

In v2 the agent may also answer questions on fixed-departure pages and via WhatsApp, but v1 keeps the agent's scope narrow to dynamic itinerary planning where it adds clear value.

## Post-booking unification

Regardless of entry path, once a booking is confirmed:

- A booking record is created with traveler details, itinerary, payment status
- The task system generates the operator's task list for this booking (see `task-system.md`)
- The traveler enters the pre-trip flow: forms to fill, documents to upload, briefing to receive
- T-2 days before pre-trip form due date, operator follow-up call is scheduled if incomplete
- During trip: WhatsApp updates, lifeline access, issue capture
- Post-trip: review request, GST invoice, photos, rebook flow

The customer experience is the same whether they booked a fixed departure or a custom trip. The internal record is differentiated (`booking_type: fixed | custom`) for reporting purposes.

## Account and identity

**Anonymous browsing and planning** — fixed-departure pages, FAQ, and the entire custom-trip agent conversation all work without an account. The traveler can describe what they want, iterate with the agent, and explore alternatives without ever signing up.

**Account at itinerary delivery** — the account-creation moment is when the customer commits to a specific trip they want to hold onto, not at payment.

- **Custom-trip path:** the agent prompts for account creation when it produces a priced itinerary the customer wants to keep. This lets slow-burn planners (the dominant pattern in this category — see behavior #1) leave and return to their saved itinerary across sessions and devices via login, instead of relying on browser cookies or magic links.
- **Fixed-departure path:** the account is created at the "Book this departure" form step (before payment), since there is no agent-produced itinerary moment to anchor it to.

In both paths, account creation happens *before* the Stripe payment step, not after it. Methods are email + OTP or social login; the operator chooses which are enabled.

When account creation fires, any in-flight `conversations` row with `customer_id = null` is re-keyed to the new (or matched-existing) `customer_id`. Returning customers (email match) merge into their existing account rather than creating a duplicate.

**Magic links for companions** — the lead traveler doesn't need to create accounts for their companions. Companions receive magic links by email or WhatsApp to fill their own pre-trip forms (passport, dietary, medical) without password setup.

**Returning customers** — accounts persist across trips. A traveler returning for a second trip with the same operator has their saved travelers, document library, and preferences pre-loaded. The agent is aware of past trips and can reference them ("welcome back — your last trip with us was Bandhavgarh in March 2025, how did it go?").

**Cross-operator privacy** — a traveler may book with multiple operators on TravelHub. Their account on operator A is separate from operator B; the canonical traveler identity is not currently shared across operators in v1. (This may change in v2 if travelers ask for it; for now, the privacy default is isolation.)

## Channels

**Web** — the white-labeled customer site at `{operator}.travelhub.hakido.co`. Primary channel for browsing, agent chat, account management.

**Mobile** — the customer mobile app (iOS and Android via Expo). Primary channel during trips. Can also browse and book pre-trip; account is shared with web.

**WhatsApp** — outbound notifications from the operator's verified business number; inbound replies are ingested into the operator dashboard. Primary channel for domestic travelers and during-trip communication.

**Email** — secondary channel for transactional messages (confirmations, receipts, invoices, magic links). International travelers use email more heavily than domestic.

**Phone** — operator-driven, used for pre-trip follow-up calls and high-touch customer service. Not a system primitive in v1; the system reminds the operator to make the call but does not place it.

## Constraints and out-of-scope

**Out of scope for v1:**
- The agent answering questions on fixed-departure pages or via WhatsApp (deferred to v2)
- Multi-language beyond English (deferred)
- Native mobile-first booking flow (the mobile app supports browsing and booking but the experience is optimized for during-trip use; pre-trip, web is primary)
- Cross-operator traveler identity (each operator's customer base is independent)
- Loyalty programs, points, gift cards (operator can configure rebook discounts but not full loyalty)
- Public traveler profiles, reviews visible to other travelers, social features
- Self-service refund processing (refunds initiated by customer require operator approval; agent quotes the math but doesn't authorize)

**Constraints that shape the design:**
- The agent quotes operator-authored content verbatim. It does not invent prices, dates, availability, sighting probabilities, or vendor commitments.
- The operator is the source of truth for everything customer-facing. The agent and the storefront are consumers of operator-configured content.
- Travelers never see vendor commercial details (rates, contracts) — only customer-facing information about lodges, naturalists, drivers.
- Payment card details are never collected, displayed, or stored by TravelHub. Stripe handles all of this.
