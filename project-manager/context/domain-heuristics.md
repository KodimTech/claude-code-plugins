# Domain heuristics — Sales, Reservations, Customer Service for Kathy

Generic PM mechanics (priority, size, retros) live in `pm-playbook.md`. This file is the **domain lens**: things that are true about selling, taking bookings, and supporting customers on WhatsApp that should shape how the PM prioritizes and scopes Kathy's work.

Use this file during `weekly-plan`, `groom`, `create-card` (sizing & priority), and `roadmap` (theme selection).

---

## 1. Sales (pre-conversion conversations)

### The only sale metric that matters early

**Closed bookings / qualified conversations.** Not messages received, not replies sent. A conversation where Kathy understood the intent and got a booking is the numerator.

Track (or ask the founder to track) 3 numbers per week:
- Qualified conversations (intent = book / buy / ask catalog)
- Bookings closed by Kathy without human fallback
- Bookings lost because Kathy got confused / escalated wrongly

### Common Kathy-specific failure modes in sales

- **Out-of-menu questions** that look in-domain ("¿venden torta sin gluten?" when no gluten-free entries exist). Kathy should say what she knows and propose alternatives, not fabricate.
- **Price negotiation** — end consumers push on price; the agent must refuse gracefully without killing the conversation.
- **Off-hours uncertainty** — customer asks at 23:00, business opens at 09:00. Kathy should hold the intent and convert, not tell them to come back tomorrow.
- **Group orders / edits** — "cambiá el último pedido a mediano" → high risk of state confusion. Always a prioritization candidate.

### Prioritization lens (sales)

When ranking cards tied to sales:

- **Conversion-at-close > funnel top.** A change that improves the last 20% of the conversation (closing the booking / payment / confirmation) is worth more than a change that attracts more conversations but doesn't convert.
- **Menu-as-catalog beats menu-as-doc.** If the business can't expose inventory / services in a structured way, Kathy is guessing. `area:menu-catalog` unblocks `area:sales-flow`.
- **Kill silent failures first.** A Kathy that fails quietly (e.g., doesn't reply, replies with "no entiendo") is worse than one that escalates loudly. Instrument silence.
- **CAC ≠ priority.** It's tempting to prioritize marketing/onboarding UX, but Kathy's moat is the conversation. Invest there proportionally.

### Cards that smell wrong for sales

- "Mejorar onboarding" without a metric — probably low-value until you have 10+ paying customers to learn from.
- "Agregar integración con X" before the conversion at close is solid — integrations amplify whatever you have; if the core is leaky, you amplify the leak.
- "Nuevo tipo de promoción" before menu CRUD exists — you're building on sand.

---

## 2. Reservations (bookings / scheduling)

### The numbers that define a good booking product

- **Booking success rate** = bookings confirmed / bookings attempted. Target: >85% once menu/catalog is in.
- **No-show rate** = bookings not honored / bookings confirmed. Benchmark for restaurants/clinics: 15-30% industry baseline — Kathy should measure it even if she can't solve it yet.
- **Reschedule rate** = bookings changed after confirmation. Silent signal of friction or overbooking.
- **Time-to-confirm** = message in → booking confirmed. Anything >3 minutes loses customers to a human competitor.

### Common Kathy-specific failure modes in bookings

- **Timezone & calendar off-by-one.** The business owner in a different timezone, the customer typing "mañana a las 3" when it's already 23:00 — classic confusion. Always make the resolved date-time explicit in the confirmation.
- **Capacity blindness.** Kathy confirms a 4pm slot when the calendar already has 4pm booked. Needs real read of availability.
- **Modifiers & party size.** "Reserva para 6 con 2 niños" → different table logic. Underestimated scope.
- **Confirmation loop fragility.** WhatsApp delivery failures at the moment of confirmation are catastrophic — treat with retries and explicit logs.
- **Cancellations** — if Kathy can book but not cancel, she's a liability. Cancellation flow is not optional.

### Prioritization lens (reservations)

- **Real-time capacity read > pretty UI.** If Kathy can't see the current state of the calendar, nothing else matters.
- **Confirmation reliability > confirmation speed.** A 5-second confirm that always works beats a 1-second confirm that silently fails.
- **Explicit date-time echo in every confirmation.** Low-effort, high-value — "Te reservo el martes 16 de abril a las 15:00. ¿Confirmás?"
- **Cancellation / reschedule flow is sprint-1 worthy**, not a nice-to-have.
- **Reminders reduce no-shows by 20-40%** — historically one of the highest-ROI cards for booking products.

### Cards that smell wrong for reservations

- "UI de reservas super bonita" before the confirmation loop is rock-solid — optimizing the wrong layer.
- "Integración con N calendarios" before the read-availability primitive is clean — build the primitive first.
- "Smart suggestions" ("te sugiero este horario") before the basic "lo confirmo" flow is reliable.

---

## 3. Customer service (post-sale, support, retention)

### Who's the customer here?

Two customers, different SLAs:

| Customer                          | What they need                                         | SLA expectation                     |
|-----------------------------------|--------------------------------------------------------|-------------------------------------|
| **Business owner** (paying user)  | Admin works, their bookings flow, their bill is right  | <24h for issues, <1h for outages     |
| **End consumer** (not a Kathy customer, is the *business's* customer) | Getting an answer on WhatsApp | <5 min — usually handled by Kathy herself |

When a card mentions "support" or "cliente", always clarify which.

### Common failure modes in support

- **Silent degradation.** Business owner doesn't realize Kathy stopped replying on their WhatsApp for 4 hours because nothing alerts them. Health monitoring is support work.
- **Ambiguous billing.** "Me cobraron doble" conversations are expensive both in time and trust. Billing transparency is retention.
- **"Kathy contestó mal"** — no way to report it, no way to fix it fast. A feedback + correction loop is table stakes.
- **Escalation-to-human path unclear.** If the owner has no way to take over a conversation mid-flight, Kathy can trap customers.

### Prioritization lens (customer service)

- **Detection > resolution.** A dashboard that says "Kathy has not replied in 30 min" is worth more than a fancier prompt.
- **Owner self-service beats support tickets.** Every card that lets the owner fix something themselves removes a future support interaction.
- **Feedback ingestion is a product surface.** When a paying customer ($30 or $60 tier especially) sends feedback, it should land directly in the PM feedback inbox, not in a DM that gets forgotten. Use `/project-manager:log-feedback`.
- **Tier matters.** A $60-tier complaint signals willingness-to-pay-more; bias toward resolving those faster. A $15-tier feature request is lower priority unless it repeats across many customers.
- **Trust is asymmetric.** One bad bug in billing costs ten good features in reputation.

### Cards that smell wrong for support

- "Agregar chat de soporte en el admin" before there's a clear detection pipeline for issues.
- "Base de conocimiento" before you know what the top 5 issues actually are (use feedback inbox for 2-3 weeks first).
- "Respuesta automática a tickets" — don't automate the conversation with your paying customer; automate the one with theirs.

---

## 4. Cross-cutting heuristics

### Pricing tier as a prioritization lens

Kathy has 3 tiers ($15 / $30 / $60). For a feature or bug, ask: *which tier is this for?* and *does this card move someone up a tier?*.

Moves that tend to unlock a tier upgrade:
- Menu / catalog management (Basic → Pro)
- Multi-calendar or multi-location (Pro → Plus)
- Analytics / conversion dashboards (Pro → Plus)
- Brand / voice customization (Plus differentiator)

Avoid building a Plus-only capability when you still have 0 Plus customers — build what converts Basic into Pro first.

### The 30-customer heuristic

For every card over `size:S`, ask: *"If this ships, does a realistic prospect become a paying customer that wouldn't have become one otherwise?"* If no, the card is probably infrastructure or polish — still may be needed, but rank conservatively.

### Cost of conversation (LLM spend)

Every new prompt change or new AI capability has an economic floor:
- A booking conversation that costs $0.10 to confirm a $15/month bill pays back in ~1 confirmed booking.
- A conversation that costs $0.50 with no booking = negative unit economics at the Basic tier.
Prompt / AI cards should estimate rough cost-per-conversation impact, even informally.

### WhatsApp-specific gravity

- **Meta's approval cycle is external and slow.** Any card that needs new production WhatsApp capability is gated. Never plan a sprint where 2+ cards all depend on a still-pending Meta approval.
- **Template messages vs. session messages** have different constraints and costs. Outbound reminders use templates (slow to approve, paid per message). Session messages are within 24h of a user reply.
- **Media handling** (images, audio, location) is qualitatively harder than text. Size-M cards that "just add audio" are usually M-or-L.

---

## How the PM uses this file

At the start of any skill, after reading `pm-playbook.md`, the PM checks this file for the relevant domain section:

- Creating or grooming a sales card? → § 1
- Creating or grooming a booking/reservation card? → § 2
- Creating or grooming a support / billing / retention card? → § 3
- Always cross-check § 4 when considering trade-offs across the product.

If a card violates a heuristic here ("optimizing amplification before fixing the leak", "building Plus-tier features with zero Plus customers"), the PM flags it explicitly with a reference: `"Choca con domain-heuristics.md § 1 — conversion-at-close > funnel top. Recomiendo deprior hasta que tengamos datos de conversión base."`.
