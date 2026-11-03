# Engineering notes

Production lessons from running the Ruche OS, kept because each one was paid for once.

They share a theme. Almost nothing in this system failed loudly. The dangerous failures returned
HTTP 200, logged success, and produced a number that was plausible and wrong — and a plausible wrong
number survives review, gets presented to a client, and becomes the basis of a decision. Most of the
engineering effort below is not about making things work. It is about making failure visible.

---

## I. Silent truncation and absent data

### 1,000 rows, no error

PostgREST was configured with `max-rows=1000`. A query returning more than that is truncated with no
error, no header, no warning. Worse, without an explicit `order`, `limit` returns the *oldest* rows,
so an incremental job can read a thousand rows it has already processed, forever, and never see the
new ones.

This produced four separate incidents that looked unrelated: "leads stopped on July 30", a sum that
was quietly low, a dedup check that could not find a row that existed, and new bookings that were
never processed. The last one is the expensive version — see §III.

The rules that came out of it, now applied everywhere:

- Paginate every read that can exceed a thousand rows, and assert the count afterwards.
- Compute exact totals with `count(*)` / `sum()` in Postgres, never by summing a page in the client.
- Every `limit` carries an `order`. A `limit` without an `order` is a bug even when it works today.

### Absence of data becomes zero

The same class of bug, four times over, each time in a different team's report:

- An empty `tags_raw` column made a filter for internal test leads a no-op — the filter was correct
  and the data it filtered on was not populated yet.
- A missing call disposition made spam count as a real lead, because "real leads" was defined as
  *total minus spam minus bad*, and absent means not-spam.
- A client with no ad spend disappeared from a per-client table entirely rather than appearing with
  a zero.
- A dead `invoices` table made an executive overview display `$0` of revenue — not an error, a
  figure.

**Measure a column's coverage before you build a metric on it.** The question "what fraction of rows
actually have this field populated?" would have caught all four, and takes one query. Separate null
from zero in the UI: "no data" and "zero" look the same to a reader and mean opposite things to a
decision.

---

## II. Row-level security is subtle, and testing it wrong is easy

### Testing permissions with a service key tells you nothing

The service role bypasses RLS and grants by design. Every permission check that passed with it and
failed in the browser traces back to this. Test with the anon key, or by logging in as the real
user, and reproduce the *exact* query the front end sends — same columns, same `order` — not a
simplified version.

### A view without `security_invoker` ignores RLS entirely

A view owned by an admin role runs with the *owner's* permissions. Tightening the policy on the
underlying table changed nothing on screen, repeatedly, until someone checked `pg_class.reloptions`.

The nastier variant: a view that *does* have `security_invoker`, joined to a table that has no
policy at all. The base table returns zero rows for the logged-in user, the `LEFT JOIN` yields
nothing, and a `coalesce` turns the gap into a plausible number. In the commercial forecast this
produced an *optimistic* case that was lower than the *likely* case — an inversion nobody can read
as "permissions problem".

RLS on every new table, by default, including the ones that feel like reference data.

### Restrictive policies combine with `AND`

Two lockout policies — unapproved users see nothing, call-center setters are denied table access —
were written as `RESTRICTIVE`. Permissive policies combine with `OR`; restrictive ones with `AND`.
Any view joining `clients` therefore returned zero rows for a setter, silently.

The fix was a data-modelling one rather than a policy one: **denormalise `client_name` and
`client_number` into historical records** and drop `clients` out of the join. A log table that needs
a join to a permission-controlled table to be readable is a log table nobody can read.

### A policy function that reads the table it protects

`is_admin()` read `profiles`, whose policy called `is_admin()`. The symptom was beautifully
misleading: `select=*` returned HTTP 500 with SQLSTATE 54001 (stack depth exceeded) while
`select=id,status` worked fine. Any function called from a policy on a table must be
`SECURITY DEFINER`.

### Grants are part of the schema

A write RPC granted to `anon` is a write RPC granted to the internet, because the anon key ships in
the front-end bundle. A materialized view granted to `anon` exposed the name, phone and email of
every lead with no login at all. Both were one-line mistakes in migrations that otherwise did useful
work; both are the kind of thing that only a review checklist catches, so the checklist exists:
*every new table gets RLS, every new matview gets its grants reviewed, every new RPC is granted to
`authenticated` at most.*

---

## III. Cost is a failure mode

### The $6.77-a-day leak

A cron job pre-generated a short message for each newly booked appointment using an LLM. Its dedup
check — "which appointments already have a message?" — was a `select event_id` with no pagination,
so it stopped at 1,000 rows (§I). The 65 appointments past that ceiling were invisible to the dedup
check, went back to the model every five minutes, and were regenerated indefinitely. Working as
designed, logging success, costing $6.77 a day.

The detection heuristic that came out of it, and then found nothing else: **a credit leak looks like
constant non-zero volume per run, the same IDs in every run, and a paid call per item.** A legitimate
daily sync also reprocesses its whole list — but it does not call a paid API per item. A sweep of 26
scheduled workflows against that signature found eight that reprocess by design and none of them
touched an LLM. This was the only real leak.

### Pay for inference, then throw it away

A related bug in the same workflow: a node that could be reached from two branches only read items
from one of them. Twenty-five of 26 webhook-triggered runs failed *after* the model call had been
paid for. When an expensive step sits mid-pipeline, the steps after it deserve the same scrutiny as
the step itself.

### The index that was bigger than the problem

An upsert was slow. One index on the leads table was 75 MB; dropping it and reindexing brought the
set to 4.7 MB and unblocked the function. Indexes added speculatively for a report that was never
built are not free — they are a write-path tax collected forever.

---

## IV. Time zones move money between months

The agency operated from Brazil, served clients across every U.S. time zone, and billed monthly.
Every component of that sentence is a bug source.

- **Date-only imports wrote `T00:00Z`**, which in Eastern or Brazilian time is the *previous day*.
  June showed 219 appointments against a real 209 — ten appointments counted in May. The convention
  since: a date without a time is stored at **12:00 UTC**, which cannot cross a day boundary in any
  inhabited time zone. A live event keeps its real timestamp.
- **A wall-clock time labelled UTC.** The CRM reports an appointment's local time; the pipeline
  stored it with a `+00:00` suffix. Display was four hours off in either direction depending on the
  reader. Format in `UTC` and render the client's time-zone abbreviation next to it, so the number
  carries its own units.
- **A dunning ladder that ran at 12:00 UTC** — 08:00 Eastern — found every Eastern-time-zone client
  outside the permitted contact window, marked them deferred, and never retried. A whole segment of
  clients silently received no payment reminders.
- **A missing flag in a ClickUp date write** (`value_options: {time: true}`) shifted dates ~17 hours
  and landed them on the previous day.
- **A dialer that reports in Pacific time** regardless of the client's actual location.

The durable lesson is less about any one fix: **a timestamp without an explicit zone is not data, it
is a rumour.** Decide per metric which clock defines "a day" — and write that decision down, because
otherwise two screens will each pick a reasonable answer and disagree by one day forever.

---

## V. Upserts, triggers, and the ways a write disappears

### `COALESCE` cuts both ways

The client sync used `COALESCE(NULLIF(new, ''), old)` across 39 fields so that an empty value in
ClickUp would not erase good data. Correct, and it created three new symptoms: a field edited in the
OS reverted at the next sync, a field cleared in the database reappeared, and a field cleared only
in ClickUp would not clear. The resolution was `locked_fields` — a human edit can pin a field —
plus a documented order of operations for genuinely deleting a value.

### `COALESCE` does not skip the empty string

A webhook started sending `appointment_id: ""`. That became an empty `external_id`, and because
empty is not null, every upsert overwrote the same single row. The source system reported success;
about nineteen appointments were lost. **`NULLIF(x, '')` before `COALESCE` on anything that is part
of a key**, every time.

### A trigger that preserves the old value swallows your correction

The leads table had a trigger that kept the previous owning client when the new value arrived empty
— exactly the right behaviour for the sync, and it meant a deliberate `UPDATE` to fix a
misattributed lead ran, reported success, and changed nothing. An intentional remap now wins
explicitly. **Any trigger that protects data also blocks repair; the repair path needs to be designed
alongside it.**

### A `UNIQUE` constraint takes the whole row with it

Ad account IDs are unique per client. When an ad account was reassigned from a churned client to a
new one, the churned row still held the ID, and the entire upsert for the new client failed — not the
one field, the whole row. One client sat in a stale status for weeks before anyone connected the
two. A closing client now releases its external IDs, specifically to stop it contending for
constraints with its successor.

---

## VI. "Success" from a remote system means nothing

### HTTP 200 with `{success: false}`

The sync RPC could return a 200 whose body reported failure. Automation treats 200 as success, so
the run went green. Verification nodes now read the body and throw, which routes the execution to a
shared error workflow. **Check the body, not the status line.**

### Pagination that fails open

A misspelled option name in a paginated HTTP node — `maxRequestsHere` where `maxRequests` was
intended — meant the node silently stopped after one page. The sync processed 100 of roughly 319
clients for an unknown period, with no error anywhere. An unknown option should be an error; since
it was not, the post-condition has to be ours: **assert the count you expected to read.**

### A monolith that hides its own failures

The call-audit chain originally lived inside the dial-ingestion workflow. When transcription or the
model failed, the dial still landed and the run looked healthy; the audit simply did not exist.
Splitting it into a separately triggered workflow meant the audit could fail *visibly*.

Which is how a downstream dependency got discovered: booking messages are only generated for calls
that have a transcript. When an API credit balance ran out and the audit pipeline stalled, 28 of 29
calls got no audit, and therefore no booking message, and the reported symptom was "the booking
messages disappeared". **When a feature vanishes, look one step upstream from where the complaint
is.**

### Ghost records, and a guard rail that cannot self-heal

Leads deleted in the CRM but never marked as deleted locally inflated lead counts — one client's
dashboard showed 13 leads where the CRM had 2. The reconciliation function has a deliberate safety
valve: if more than 50% of a client's leads would be deleted in one pass, it aborts, because the CRM
occasionally returns an empty `200`.

That guard is correct and it has a consequence worth stating plainly: **a client whose data is badly
enough corrupted can never self-heal**, because every repair looks like the catastrophe the guard
exists to prevent. A one-time manual sweep cleared 967 ghost leads across 29 clients; after that,
ongoing deletions were small enough to pass the valve. A safety threshold needs a documented manual
override, or it becomes a permanent floor on data quality.

---

## VII. Working with LLMs in a pipeline

### Never trust the model's arithmetic

The call-quality rubric is a weighted multi-criteria score with criteria that may be
non-applicable. The model returns per-criterion scores and applicability through strict-schema
function calling; **the weighted percentage is then recomputed deterministically** over only the
applicable criteria. The model judges; code counts.

### A smaller model will answer a numeric question without calling the tool

The natural-language query agent runs on a cheap model, and the single most important line in its
system prompt is *no tool call, no number*. Without it, the model would confidently produce a
plausible figure from nothing. Model capability and prompt discipline trade off directly: the cheaper
the model, the more of the reasoning has to be structure rather than instruction.

### A filtered query that returns zero is not "there is none"

The agent would filter on a value that did not exist, get zero rows, and report "there are none" —
indistinguishable from a true zero and far more harmful. The RPC now returns, alongside an empty
result, **the set of values that do exist** for that column, so the agent can correct itself and
say "there are none *with that status*, and here are the statuses that occur".

Related: the dataset catalog is injected into the prompt rather than discovered through a tool call.
With a one-round-trip budget, spending the round trip discovering what exists leaves the actual
question unanswered. Valid values for enum-like columns are refreshed into the catalog by a
scheduled function, because a model without them invents plausible ones.

### Calibration is a product decision, not prompt tuning

The audit rubric's calibration rules were written from specific disagreements with the sales
manager, and each exists because the model was defensibly right and operationally wrong. Never
deduct for audio quality, background noise, accent, or difficult English — the setter cannot control
those and penalising them makes the score measure the recording instead of the work. When a visit is
already scheduled, criteria about offering time slots do not apply rather than scoring zero. A lead
who engaged on scope, price and timeline and then declined is a scored sales call, not an
unscoreable brush-off, because that call *is* the setter's performance.

**Scoring the wrong thing precisely is worse than scoring the right thing roughly**, because people
optimise against the score they are given.

### A transcript on disk is an asset

Because transcripts were stored in their own column rather than being discarded after scoring, the
entire history could be re-audited against a new rubric without re-transcribing anything or touching
the automation layer. Store the expensive intermediate artifact. One caveat learned the hard way:
re-audit by patching content columns directly, never through the normal upsert function, which
recomputes timestamps and re-resolves relationships and so destroys the very history you are
re-scoring.

---

## VIII. Infrastructure

### A stateful service that will not reconnect

The Docker Swarm network between the n8n host and its Postgres crossed the public internet and
flapped 5 to 60 times a day. Past roughly 20 seconds, n8n would lose its database and sit in
"waiting for database connection recovery" indefinitely. Postgres came back; n8n did not. Two
separate incidents cost about two hours each of no webhooks and no scheduled jobs.

Two layers fixed it without fixing the network, which was not ours to fix:

- **A watchdog** (systemd timer, 60-second interval) polls the readiness endpoint of each service
  and the worker's log. Three consecutive failures trigger a forced rolling restart in dependency
  order, with a ten-minute cooldown, and each restart is filed as an incident in the OS so the
  recovery is visible rather than invisible. Recovery time went from "until someone notices" to
  about three minutes.
- **A queuing relay** sits in front of the webhook endpoint. While n8n is healthy it is
  transparent — same response, same CORS headers, same latency. When n8n is unreachable it persists
  the request, answers `200 {"queued": true}`, and replays in order when n8n returns. Nothing
  inbound is lost. The landing pages also fire their browser-side conversion event with the *same*
  idempotency key as the server-side one, so the ad platform deduplicates and a replay cannot
  double-count.

### Capacity is a constraint, not a bug

Eleven services shared 3.7 GiB of RAM. Worker concurrency went 8 → 2 → 6 → 4 → 3 over four months,
every step forced by an out-of-memory kill. The honest diagnosis was written down rather than
re-litigated each time: *the host needs more RAM; tuning the heap just moves which service dies.*

The actual fix available at that budget was reducing payload size, not concurrency. Two ClickUp sync
workflows accounted for 76% of all stored execution data, and one workflow stored 13 MB per
execution because it passed entire task objects — 81 custom fields each — between nodes.
**In a queue-mode automation engine, every value passed between nodes is persisted. Select your
fields early.**

---

## IX. Migrations and the shape of regret

### Your repository is a log of intent, not a description of state

SQL was often run by hand in the database Studio and committed afterwards. File numbers were reused
by parallel workstreams — 62, 73, 79 and 80 each appear twice. Some fixes were applied surgically
to live views via `pg_get_viewdef` and exist in no file at all.

The bill arrived twice. Once when a migration recreated two views from an older file and silently
reverted two later surgical fixes: monthly leads went from 129 to 139, bookings from 70 to 62, one
person's attributed bookings went to zero and another received twelve that were not theirs. And
again at shutdown, when the honest answer to "can we rebuild the database from git?" was no.

A migration runner with a checksum table costs an afternoon. Not having one cost the ability to
reconstruct the system.

### Verify that a migration actually applied

The Studio editor could mangle a function body — a quote in a header comment confusing the statement
splitter, a `select ... into` parsed as `SELECT INTO`. Sometimes it failed loudly with a syntax
error pointing at the wrong line; sometimes it silently inserted an unrelated statement into the
middle of a function. Because a function body is not retrievable over the REST API, the check became
a query against the catalog: *does the new code fragment appear in `prosrc`?* "It ran before" is not
evidence that it ran this time.

---

## X. The non-technical lessons

**A shortcut taken under deadline does not announce itself later.** Secrets ended up in three
habitats: a proper credential store, hard-coded values inside workflow code nodes, and a public
config endpoint that returned live API tokens to any caller. Each arrived as a defensible
time-saving decision in a different week. Nobody decided to build that; it accreted. Security debt
compounds exactly like the financial kind, and the interest is paid at the worst moment — in this
case, during a shutdown, when every token was still valid and the question "what must be rotated
before these servers are visible to nobody?" had to be answered from memory.

**The documentation was the most valuable artifact.** A 2,000-line context document explaining *why*
each decision was made — including the ones that turned out wrong — was the only thing that let
someone new work on any of this. Code explains what the system does. Nothing but prose explains why
it does it that way instead of the obvious way, and the obvious way is what the next engineer will
otherwise try. When the system was finally appraised for resale, that document was judged the most
transferable asset in the stack.

**Build a product or build an internal tool, but know which one you are doing.** There is no
`tenant_id` anywhere in this schema. RLS partitions by role, not by organisation; the client roster
is coupled to specific ClickUp field IDs; lead-generation business rules live inside database
functions. All of that was correct for an internal system serving one company, and all of it is why
the honest answer to "can we sell this as a product?" was no. The decision was right. It was simply
never written down as a decision, so it had to be rediscovered under time pressure at the end.
