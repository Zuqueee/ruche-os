# 4 · Paid-media data pipeline

**Ad-level spend and results from Meta, Google Ads and Google Local Services, daily, for ~100 client
accounts, joined to the appointments that actually got booked.**

[← back to index](../README.md)

---

## Context

The agency was paid per booked appointment. That makes the only interesting question *cost per
booked appointment*, which requires spend from the ad platforms and bookings from the CRM and the
dialer to live in the same place, at the same grain, with the same notion of a day.

Before the pipeline, a media buyer managing dozens of accounts opened each platform's UI in turn.
Nobody could see which of a hundred accounts was drifting without checking a hundred accounts.

## What was built

`ad_insights`: one row per **client × date × platform × ad**. Deliberately at ad level rather than
campaign level, because the creative is the unit a media buyer actually changes.

```
Meta Marketing API ──┐
                     │  level=ad, time_increment=1, rolling window
Google Ads (GAQL) ───┼──► ad_insights  (client × date × platform × ad)
                     │    upsert on conflict
Google Local Services┘
                          └─► per-client tracker · wallet view · benchmarks
                              joined to events (dials, appointments)

campaign_status ◄── parallel branch, separate from the insights loop
```

Two daily workflows, one per platform, selecting only clients flagged live with an ad account
configured. Each writes through an upsert keyed on the natural composite, so a re-run or a backfill
cannot duplicate. Derived metrics — CVR, connect rate, cost per lead — are computed from the
components rather than stored from the platform, so they stay consistent with whatever definition the
rest of the system uses.

A separate branch in each workflow collects **campaign status** (active, paused, platform-flagged
issue) on its own key, without touching the insights loop. That separation matters: status is small,
fast, and useful even when an insights fetch for one account fails.

## Design decision: a rolling window instead of an incremental cursor

Both platforms are queried over a trailing window — seven days for Meta, forty-five for Google —
rather than from a stored high-water mark.

This is a deliberate trade of a little redundant work for a large amount of correctness. Ad platforms
restate recent data: attributed conversions arrive late, spend is adjusted, and a figure read
yesterday is not final. An incremental cursor captures the first, provisional version of every day
and never revisits it. A rolling window lets every day settle, and because the upsert is keyed on the
natural composite, re-reading is free of consequence.

It also makes backfill a non-event. Widening the window *is* the backfill.

## Design decision: a metric that is intentionally wrong

The Meta lead count sums four different conversion signals — the standard lead action, the pixel
lead, the on-site grouped lead, and the lead-gen form action. These overlap, so the total is
roughly **double** the unique lead count.

That looks like a bug and it is a documented choice. The metric's job in this system is *relative*:
comparing creatives inside one account, and tracking an account's direction week over week. For that,
a consistently inflated figure available for every account beats a correct figure that requires
per-account attribution configuration nobody maintains.

The absolute lead count comes from the CRM, where a lead is a contact record and the denominator is
defendable. **Two metrics with the same name and different jobs need different names and a written
reason** — which is why this one is written down here and in the system's own documentation, since
the next person to find it will otherwise "fix" it.

## Design decision: fail per client, not per run

A systemic error — expired credentials, an API version rejection — must stop the run loudly, because
every account is affected and silent partial data is worse than none.

A per-client error must skip that client and continue. The common cases are mundane and outside the
agency's control: a client revokes API access on their ad account, an account is suspended, a token
for one integration expires. One client's problem must not cost ninety-nine clients their data for
the day.

The corresponding diagnostic discipline: **a gap in this table is usually real.** The instinct is to
suspect the pipeline, and the common cause is an account that genuinely did not spend — paused
campaign, exhausted budget, a client who stopped. Checking delivery before debugging ingestion saved
considerable time once it became the first question instead of the last.

## Two problems with no clean solution

**An API deprecation with a deadline.** Google's Ads API deprecates versions on a schedule and
rejects the old one outright. Migrating the query layer forward was unavoidable maintenance with a
hard date attached — the kind of work that is invisible when done and an outage when not.

**Performance Max has no ad groups.** Google's automated campaign type does not expose the entity
this table is keyed on, so a campaign type that carried real spend is structurally invisible at ad
level. There is no fix at this grain; the honest response was to record the blind spot in the
documentation, next to the metric, so nobody reads a zero as a result. Google Local Services needed
its own detection and its own platform code for the same reason.

**UTM joins on name, not ID.** Campaign and ad attribution from the CRM's UTM parameters joins to
this table *by name*, because the ad platforms' IDs do not appear in the CRM's captured parameters.
That makes the agency's creative naming taxonomy load-bearing infrastructure: a rename breaks
attribution for the period after it. Fragile, known, and documented rather than quietly depended on.

## Outcome

A media buyer managing dozens of accounts could see all of them in one tracker, with per-client
benchmarks and week-over-week notes, instead of opening platform UIs one at a time. Spend joined to
dials and appointments in the same database made cost per booked appointment — the number the
business was actually paid on — a view rather than a monthly spreadsheet exercise.

---

**Related:** [Campaign creation engine](05-campaign-engine.md) · [Conversion tracking](06-conversion-tracking.md) · [The platform](01-platform.md)
