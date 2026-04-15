# Kathy — product briefing

## What Kathy is

Kathy is an AI agent that answers WhatsApp messages on behalf of small service businesses so they stop losing sales and bookings when a human can't reply fast enough.

Concrete failure modes Kathy prevents:
- A restaurant misses a delivery order because nobody saw the WhatsApp message.
- A doctor loses a booking because a patient didn't get a reply within the hour.
- A service provider (salon, clinic, mechanic) misses a repeat customer because the conversation went cold overnight.

The pitch to a business owner is: **"Kathy answers your WhatsApp 24/7, takes the booking, and gives you the customer — you just show up."**

## Who uses Kathy

Two audiences, two React panels:

- **Business owner** (the paying customer) — uses the **admin panel** to configure Kathy: connect WhatsApp, define products/services/menu, see bookings, check conversations, manage billing.
- **End consumer** (the business's customer) — chats on WhatsApp with Kathy. Does not touch the web.

There is also a **customer panel** (second React app) whose exact role inside Kathy should be documented here as it solidifies. For now, treat it as the self-service / billing / account surface for the paying business owner, separate from the operational admin panel.

## Architecture at a glance (for PM context only — do not plan at this level)

- **Rails API** (PostgreSQL, background jobs) — the brain: conversation state, business rules, WhatsApp integration, AI orchestration.
- **React admin panel** — daily operational UI for the business owner.
- **React customer panel** — account/billing/self-service surface.

These live in three separate repos. The PM plugin never reads these repos directly; it hands work off via `repo:api`, `repo:admin`, `repo:customer`, `repo:cross` tags so that `/api-dev:plan` and `/customer-dev:plan` can do the technical breakdown inside the right codebase.

## Current state (MVP in progress)

As of the last update:

- ✅ Core AI conversation flow for **bookings** (reservations) works end-to-end.
- ⏳ **WhatsApp Business Management permission** — pending approval from Meta. This is a hard blocker for onboarding real businesses at scale.
- ⏳ **Products, Services, Menu management** — admin UI + API to let a restaurant publish its menu, or a clinic publish its services. Not yet built.
- Most work in the backlog today is **new features** rather than bugs.

Keep this section honest: if the founder says something changed, update it during `weekly-review` or `roadmap`.

## Business goal (the number that matters)

**30 paying customers** across 3 tiers:

| Tier   | Monthly | Rough positioning (refine as you learn) |
|--------|--------:|-----------------------------------------|
| Basic  |    $15  | Solo professional / very small shop     |
| Pro    |    $30  | Small business with moderate volume     |
| Plus   |    $60  | Restaurant or clinic with full catalog  |

Every prioritization decision the PM makes should, where possible, trace back to one of:

1. **Unblocking onboarding at scale** (e.g., Meta WhatsApp permission, billing, multi-tenant signup flow).
2. **Raising conversion** (prompt quality, AC/delivery of the booking, menu publication so restaurants actually send customers).
3. **Retention signals** (business owner seeing value in the admin panel: bookings count, revenue attributed, conversation summaries).
4. **Cost control** (LLM spend per conversation, infra).

If a card doesn't fit any of those buckets, ask whether it should be deprioritized.

## Typical work types (what the backlog usually looks like)

- **Features** (most common right now): new admin screens, new AI capabilities (menu understanding, service catalogs), new WhatsApp flows.
- **Prompt / AI quality**: changes to how the agent replies, handles edge cases, asks for clarification.
- **Integrations**: WhatsApp API, payment providers, calendar, etc.
- **Analytics**: bookings funnel, conversion, cost per conversation.
- **Bugs**: usually conversation state issues, WhatsApp webhook reliability, admin UI.
- **Chores / infra**: deployments, background jobs, observability.

## Stakeholders

- **Founder / solo dev** (the user of this plugin). Writes code, makes product calls.
- **Sales contact** (external) — `erika.leon@kodim.tech` is the commercial contact for `core-web-customer` per that plugin; relevant if Kathy marketing/signup pages are touched.

## What the PM should assume when in doubt

- The founder has 10 hours a week for implementation. Not 40.
- Shipping something usable beats shipping something perfect.
- One well-scoped card closed is worth three half-done cards.
- If Meta permission is still blocked, anything requiring production WhatsApp at scale is not yet shippable — work that can be validated in sandbox is fine.
