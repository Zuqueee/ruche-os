# 6 · Conversion tracking

**Server-side conversion tracking that optimises toward a booked appointment instead of a form fill
— one reusable workflow for every client, with onboarding reduced to a row and a webhook.**

[← back to index](../README.md)

---

## Context

Meta optimises campaign delivery toward whatever event you report as the conversion. Report form
submissions and it will find you people who submit forms — which, in home improvement, includes a
large population who fill in anything and never answer the phone.

The event the agency was actually paid for was a **booked in-home appointment**, which happens
minutes to days after the click, inside the CRM, on a human phone call. Getting that event back to
Meta, attributed to the right click, is the difference between buying leads and buying customers.

The constraint that rules out the obvious alternative: a *closed deal* would be the ideal
optimisation event, and the sales cycle in remodeling runs 120+ days. Meta's attribution window is
seven. A closed-deal event arrives long after the window has shut, so it can be measured but cannot
be optimised against. **The booked appointment is the latest-stage event that still lands inside the
attribution window** — which makes it the correct optimisation target, not a compromise.

## What was built

A three-event taxonomy applied identically across every client account:

| Event | Fires when | Used for |
|---|---|---|
| `Lead` | form submitted on the landing page | volume, top of funnel |
| `QualifiedLead` | CRM contact enters the "appointment booked" stage | **the optimisation event** |
| `Purchase` | deal closed, with value | measurement and ROAS — never optimisation |

```
Landing page
  │  captures Meta click identifiers and browser cookies alongside the form fields
  │  fires `Lead` browser-side
  ▼
CRM (one sub-account per client)
  │  contact enters "appointment booked" stage
  ▼  webhook
n8n — one reusable dispatcher workflow for all clients
  │  1  normalise the inbound payload
  │  2  look up the client by CRM location ID
  │  3  look up that client's dataset credentials
  │  4  hash the personal identifiers, build the payload, derive a stable event ID
  │  5  POST to the Conversions API
  │  6  log the attempt and the response
  ▼
Meta Events Manager → custom conversion → campaign optimisation
```

Supporting schema: a client registry keyed on CRM location ID, a separate credentials table, and an
event log. The landing-page snippet captures the Meta click ID and browser cookies so the
server-side event can be attributed to the click that caused it — without them, a server-side
conversion is an orphan.

## Design decision: one workflow, N clients

The tempting shape is one workflow per client, cloned and edited. With dozens of accounts that
becomes dozens of copies of the same logic, drifting apart, each needing the same fix applied by
hand.

Instead: a single dispatcher that resolves everything it needs — which client, which dataset, which
token — from the database at runtime, keyed on the CRM location ID that arrives in the webhook.

The operational payoff is the design goal stated as an acceptance test: **onboarding a new client is
one row in the database and one webhook configured in their CRM sub-account.** No deploy, no clone,
no code change. That constraint is what forced the credentials into a table instead of into the
workflow, and the client resolution into a lookup instead of a conditional.

## Design decision: idempotency shared across both delivery paths

The same conversion is reported twice by design — once from the browser, once from the server — for
coverage: the browser event catches users the server-side path misses, and the server-side event
catches everyone with an ad blocker or a privacy browser.

Both paths send the **same derived event ID**, so Meta deduplicates them into one conversion. Without
that, every conversion counts twice, every cost-per-conversion figure halves, and the optimiser
learns from inflated data.

This also makes the whole chain safe to retry, which is what let the webhook relay in
[case 8](08-resilience.md) work: a queued webhook replayed twenty minutes later re-sends the same
event ID and cannot double-count.

## Design decision: hash at the edge, log the attempt

Personal identifiers are hashed before leaving the automation layer, per Meta's requirements, and
never logged in the clear.

Every dispatch is logged with its outcome. This is the only part of the system with any observability
at all, because a conversion event is fire-and-forget: Meta's Events Manager shows aggregate
received events with a lag, and nothing tells you which specific lead failed to report. Without a
local log, "did this appointment reach Meta?" is unanswerable — and that question gets asked every
time a campaign's performance drops.

## Why this was the highest-leverage piece of media work

Everything in [case 4](04-paid-media-pipeline.md) measures what already happened. This changes what
happens next, because the optimiser acts on it.

Reporting form fills and reporting booked appointments produce materially different audiences from
the same budget. Appointments are also a far sparser signal than form fills, which is a real
trade-off: the optimiser has less data to learn from, and small accounts can struggle to exit the
learning phase. Keeping `Lead` as a reported event alongside `QualifiedLead` preserves the dense
signal for diagnostics while optimisation runs on the sparse, correct one.

## Outcome

A standard, reusable tracking stack across all client accounts instead of per-client bespoke setup.
Optimisation pointed at the event the business was paid for. Client onboarding reduced to a database
row and a webhook, which is what made it actually get rolled out rather than remaining a
best-practice document.

---

**Related:** [Paid-media pipeline](04-paid-media-pipeline.md) · [Campaign engine](05-campaign-engine.md) · [Infrastructure resilience](08-resilience.md)
