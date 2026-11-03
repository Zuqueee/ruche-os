# 5 · Campaign creation engine

**A form that turns a folder of creative into a built, paused Meta campaign — idempotently, and
resumable halfway through.**

[← back to index](../README.md)

---

## Context

Launching a creative wave meant a media buyer clicking through Meta's Ads Manager: upload each video
and image, create a creative per asset, create the campaign, create the ad set, create one ad per
asset, name everything according to the agency's taxonomy. Twenty minutes to an hour per wave, across
dozens of accounts, and every step a chance to mistype a name that attribution later depends on
(see [case 4](04-paid-media-pipeline.md) — UTM joins on name).

The creative itself already existed, organised in a shared Drive folder per wave, with filenames
following the agency's own convention.

## What was built

One form: the account, the campaign and ad set settings, the ad copy, and a **link to the Drive
folder** for the wave. The engine does the rest.

```
CONSOLE (media-buying manager role)
  │ validate the order client-side  ── errors shown, nothing submitted
  │ fn_mb_campaign_begin(order)     ── same idempotency key? return what exists
  ▼
n8n  (Meta write token and Drive credential live here, never in the browser)
  │ 1  load: reject a stale token; an order already past `creating` returns its result
  │ 2  list the folder and its direct subfolders (VIDEO, IMAGES, …)
  │ 3  plan: one asset per file; re-validate server-side against live account context
  │        (pixels, pages, benchmarks, client record read fresh from Meta)
  │ 4  three locks, all required: explicit confirm · global creation flag · write token
  │        any missing → return the PREVIEW (assets + generated ad names), upload nothing
  │ 5  fn_mb_campaign_claim → exactly one execution per order
  │ 6  per asset: download from Drive → upload to Meta → RECORD THE ID IMMEDIATELY
  │        an asset that already has an ID is skipped
  │ 7  wait for video processing (poll, bounded), store the thumbnail
  │ 8  one creative per asset → campaign → ad set → one ad per asset
  │        everything born PAUSED
  │ 9  fn_mb_campaign_finish(ok | error) → created | partial | failed
  ▼
v_mb_campaign_pendencias ── partial, failed, or stuck in `creating` >15 min becomes a human task
```

## Design decision: three independent locks before anything is written

A bug in this engine does not produce a wrong number on a screen. It spends a client's money, or
publishes an ad with the wrong copy to the wrong audience. So nothing uploads unless all three of
these hold: the submitter explicitly confirmed, a global creation flag is enabled, and a Meta
**write** token is present.

If any is missing, the engine runs the entire planning phase anyway and returns the **preview** — the
list of assets it would upload and the exact ad names it would generate — without touching Meta. That
made the dry run the default development and review path, rather than a special mode somebody has to
remember to use.

**Creating and activating are separate flows.** Everything is created `PAUSED`. Turning spend on
requires its own confirmation of budget and start date, by a human, elsewhere. The engine cannot
start spending money, by construction.

## Design decision: idempotency at three levels

A remote write across ten steps will be interrupted. The network drops, the platform rate-limits, a
video is still processing, a user double-clicks. Each of those is handled by a different mechanism:

1. **At submission.** The order carries an idempotency key. Submitting the same order twice returns
   the existing one rather than creating a second.
2. **At claim.** `fn_mb_campaign_claim` guarantees exactly one executing run per order. Two
   simultaneous clicks cannot both proceed — the second gets the first one's handle.
3. **At each asset.** Every uploaded asset's platform ID is recorded *at the moment of upload*, not
   batched at the end. On resume, an asset that already has an ID is skipped. A run interrupted after
   seven of nine uploads resumes by doing two.

That third one is the load-bearing piece and the easiest to get wrong. Recording IDs in a batch at
the end is simpler, faster, and means an interruption wastes every upload that already succeeded —
and on Meta, re-uploading video is the slowest and most rate-limited part of the whole sequence.

## Design decision: retryable and terminal failure are different products

This was a product decision disguised as error handling.

**Retryable** — a platform usage limit, transient instability, a video still processing after the
polling budget expires. The engine releases its claim, the order *stays* in `creating` with
everything already uploaded intact, and the next invocation resumes. No human is involved, because
nothing is wrong.

**Terminal** — the platform rejected the request, or a response never came back. The order becomes
`partial` or `failed`, with the platform's error **translated into language the media buyer can act
on**. A raw Graph API error code in front of a media buyer is an escalation to an engineer; "this ad
account's pixel is not available on this page" is a task they can finish themselves.

Anything `partial`, `failed`, or stuck in `creating` for more than fifteen minutes surfaces in a
pendency view as human work. **A stuck state that nobody is told about is a lost campaign**, and the
only reliable detector of "stuck" is a timer.

## The unglamorous part: reading a human-organised folder

The creative folders were organised by people, for people. Videos follow a naming convention
encoding the creative number, format, variant and client. Images sit in themed subfolders. A stray
logo file lives alongside the real assets.

The rules the planner applies, each one with its own warning code so the preview explains itself:

- The order points at a folder: the whole wave, or just the video subfolder, or just the images.
- It reads that folder and its **direct** subfolders only. A theme folder nested inside IMAGES is not
  read when the order points at the wave — and the preview says so explicitly, with a code, rather
  than silently omitting files.
- **If any file matches the naming convention, only matching files become assets.** The stray logo
  is excluded, with a warning. If *no* file matches, every file is included and numbered by filename
  order, with a different warning.

That last rule is the one worth defending. The strict reading would reject unconventionally named
folders; the permissive reading would quietly include junk. Choosing behaviour based on whether the
convention is being used *at all*, and announcing which branch was taken in the preview, means the
engine works for the folders that exist rather than the folders the convention describes.

## Outcome

A creative wave went from a manual click-through to filling one form and reviewing a preview. Names
follow the taxonomy by construction, which keeps the attribution join in
[case 4](04-paid-media-pipeline.md) intact. Secrets never leave the automation layer. Nothing can
start spending without a separate human decision.

The engine reached pilot readiness as the agency wound down, so it has a documented contract, a
preview path exercised against real accounts, and less production mileage than the other systems
here. Recorded as built and validated, not as battle-tested.

---

**Related:** [Paid-media pipeline](04-paid-media-pipeline.md) · [Conversion tracking](06-conversion-tracking.md)
